# Measure before you shrink: finding where an iOS app's storage actually goes

Users of a content app kept reporting that it "takes up too much storage". The
app's own number agreed: the average footprint was 900–1300MB. Nobody could say
which directory those bytes were in. Every proposed fix was a guess about a
breakdown nobody had measured: lower a cache cap, add expiry, clear more on
demand.

This article covers the metric built to answer that question: why the existing
number couldn't, the design choices that matter for this kind of metric, and how
it was validated before anyone trusted it. Two companion articles cover what it
found:

- [A one-line `receive(on:)` that silently disabled a video cache for nine months](combine-receive-on-dead-cache.md)
- ["Clear cache" didn't clear the biggest cache: WKWebView's disk cache](clearing-wkwebview-cache.md)

## 1. Why the existing number couldn't answer it

The only storage signal was an `app_size_mb` property on the analytics event for
tapping "Clear cache". It had two structural flaws, and rewriting the query
could not fix either:

- **One total.** It was the sum of bundle, Documents, Library and tmp. A
  subdirectory changing by tens of MB disappears inside a total of several hundred.
- **A tiny, self-selected sample.** The property was only attached when a user
  tapped Clear cache *and* a feature flag was on. The tap event fired 16–20k
  times a week; rows carrying the property numbered **6–18 a week**. Every one
  came from someone unhappy enough with storage to go looking for a cleanup
  button, and every one recorded the size *before* cleanup.

The dashboard didn't show the row count. It was recovered from the mean: the
weekly sums 7907, 13241 and 30377 only produce the observed repeating decimals
when divided by 6, 14 and 18. If a dashboard only shows you a mean, its decimal
tail can still tell you the n.

To check whether the sample could detect anything at all, the series was
compared across a release that raised an image cache's cap from 200KB to
200MB. That change should have produced a visible step. The weekly mean swung
between 928 and 1688MB with no step anywhere. The noise was far larger than the
signal, so the metric could not support any capacity decision.

## 2. The replacement: one breakdown per device per day

A new first-party event, sent once per device per calendar day, from every user:

| Field | Meaning |
| --- | --- |
| `app_size_mb` | bundle + Documents + Library + tmp: the same definition as the old number, so the two can be reconciled |
| `bundle_mb` | the installed app bundle |
| `library_mb` / `library_files` | `Library/`, bytes and file count |
| `documents_mb` / `documents_files` | `Documents/` |
| `tmp_mb` / `tmp_files` | `tmp/` |
| `caches_mb` / `caches_files` | `Library/Caches/` |
| `video_cache_mb` / `video_cache_files` | the video disk cache under `tmp/` |
| `top_cache_dir` / `top_cache_mb` | name and size of the largest child of `Library/Caches/`, found at runtime |

### Four design decisions

1. **A first-party event, not new properties on the existing analytics event.**
   The analytics event was generated from a tracking plan. Adding properties
   meant changing the plan, regenerating code, and waiting on another team's
   schedule. Attaching the breakdown to the tap event would also have kept the
   survivorship bias.
2. **Find the largest cache directory at runtime and report it by name.**
   Hard-coding a third-party path such as an image library's cache folder
   breaks when the SDK renames it. It also measures only the consumers you
   already suspect. Reporting the name lets the data surface the ones you
   didn't.
3. **Report file counts alongside bytes.** Some failures add entries without
   adding bytes. The video cache was a live example: 31 entries of exactly 1025
   bytes, 132KB in total, which rounds to 0 in MB. Bytes alone would have called
   the cache empty and healthy. The file count was the first sign it was
   neither.
4. **Throttle per calendar day, not per rolling 24 hours.** A rolling window
   systematically drops regular users. Someone who opens the app at 8:00 every
   morning is only 23h59m past their last report the next day and gets skipped,
   so they end up reporting every other day.

### Keeping the walk cheap

- Walk the directories on a dedicated queue, not the serial queue the settings
  screen uses to read sizes.
- Walk `Library/Caches/` once, computing the total and the largest child in the
  same pass. The first version took three passes.
- Cache the bundle size keyed by build version, since it only changes on update.
- Call `fileExists` before walking the video cache directory. That directory can
  legitimately be missing, and enumerating a missing directory logged at
  `.error`, the one level the logger did not rate-limit.
- Enumerate lazily, with an `errorHandler` and an `autoreleasepool` per entry.
  Otherwise peak memory during the walk grows with the total file count.

### Why not sample

Volume is capped by daily active devices, one row each. That came to under 0.1%
of the warehouse's daily event volume, and an order of magnitude less than a
single existing video-playback event. Sampling would have made the data worse:

- It breaks the strongest validation check, "exactly one row per device per day".
- The long tail of `top_cache_dir`, the consumers nobody predicted, is exactly
  what sampling drops first.

If you do need to cut volume, sample by a stable hash of the device id, never by
`random()` per launch. Otherwise every device's time series fills with gaps.

The event shipped behind a remote kill switch, defaulting to on.

## 3. Validating it before launch

### Tests, and a mutation that changed nothing

158 unit tests passed, followed by three rounds of mutation testing. In one
round, removing an `isDirectory` guard didn't turn a single test red. Instead of
adding a test just to make the mutant fail, the line was investigated. At that
point it really was redundant, because walking a non-directory path already
returned 0. A later refactor made it load-bearing. A surviving mutant is a
question, not automatically a coverage gap.

### On a real device

1. **Uninstall first.** The throttle stores the last reported day in
   `UserDefaults`. A leftover value makes the first launch skip the event, which
   looks exactly like the event being broken.
2. Cold launch, stay for more than 10 seconds, kill the app. Repeat once.
3. Pull the debug log out of the app container with `devicectl`. The debug build
   already wrote every analytics event with all its fields to a log file, so no
   extra logging was needed.

There was **one** event across two cold launches, so the throttle worked.

| Field | Value | Check |
| --- | --- | --- |
| `app_size_mb` | 531 | = 502 + 26 + 2 + 1 |
| `bundle_mb` | 502 | 94.5% of the total, but this is a debug build |
| `library_mb` / `library_files` | 26 / 142 | files > 0 |
| `documents_mb` / `documents_files` | 2 / 32 | files > 0 |
| `tmp_mb` / `tmp_files` | 1 / 4 | |
| `caches_mb` / `caches_files` | 13 / 81 | 13 ≤ `library_mb` |
| `video_cache_mb` / `video_cache_files` | 0 / 2 | invisible in MB, visible as a count |
| `top_cache_dir` / `top_cache_mb` | `WebKit` / 5 | 5 ≤ `caches_mb` |

Every structural constraint held: the four parts sum to the total, both file
counts are positive, Caches fits inside Library, the video cache fits inside tmp,
and the largest cache fits inside Caches.

That proves the **mechanism**, not the distribution. A debug bundle carries
symbols and unoptimized code, so 502MB says nothing about release builds. The
device had just been reinstalled, so its caches were empty, and `WebKit` at 5MB
reflects nothing about the fleet.

It did raise a question the old single total could never have asked. If the
*release* bundle also turned out to be a large share of the footprint, that share
is a floor no cleanup can touch. The work would then be "shrink the binary", a
different project entirely.

## 4. Post-launch validation plan

**Before ramping, register the event name** in the data platform. Unregistered
events were dropped silently, without any error.

| Step | Check | A failure means | When |
| --- | --- | --- | --- |
| 1 | `library_files > 0` and `documents_files > 0` | a directory walk is failing | first 24h |
| 2 | rows per device per day = 1 | > 1: throttle broken; always 0: suspect the clock | first 24h |
| 3 | rows ÷ daily actives, as a trend | a sudden drop is a regression; the absolute ratio is below 1 by design | first week |
| 4 | compare with the old metric's mean, magnitude only | if the new mean isn't clearly lower, ~1GB is the norm across all users, not a problem of a few | once |
| A | share of devices by `top_cache_dir` | which cache the next ticket should target | after a week |
| B | `bundle_mb` as a share of `app_size_mb` | if the bundle dominates, shrink the binary instead of clearing caches | after a week |

Step 1 deliberately does **not** check that the parts sum to the total. That is
a tautology: if one directory walk fails and returns 0, the total drops by the
same amount and the equation still holds.

Step 4 compares across two systems. The old event turned out to go only to the
product analytics tool and had zero rows in the warehouse. The query itself was
fine: the same partition held millions of rows of another event. The two
systems differ in deduplication, sampling and session rules, so the comparison
can only judge orders of magnitude.

Steps 1–3 are one-time acceptance checks, not dashboards. Step 4 is a one-time,
falsifiable judgement. Only A and B are worth watching over time.

## 5. A fix that made the numbers honest but freed nothing

Before the metric existed, a cleanup change had already shipped:

- Clear cache also wiped the video cache and counted the freed bytes.
- The app size started to include `tmp/`.
- Freed bytes were measured by deleting files one by one, instead of trusting the
  cache's own running total, which drifts.

These changes made the number truthful and gave users a cleanup button that
worked. They did not reduce the steady-state footprint by a single byte. Once
the video cache turned out to hold 132KB, the real gain was far smaller than
expected. The user report that had been closed on the strength of the change was
reopened.

Two lessons:

- Adding `tmp/` changed the metric's definition, so values before and after the
  change can't be compared. Split the data by the first release build that
  contains the change, not by merge date.
- "Users can now clear it" and "the app is smaller" are different claims. Close
  a ticket only on the claim you actually measured.

## Takeaways

- A metric collected only when users complain measures complainers. Before
  building on it, check its n. A natural experiment, such as a release that
  should have moved the number, tells you whether it can detect anything.
- For storage, report directories, file counts, and the name of whatever is
  largest. Totals and hard-coded paths hide the surprises.
- Throttle daily metrics by calendar day.
- Write acceptance checks that can fail. "Parts sum to the total" can't.
- Prove the event arrives (registered, one row per device per day) before you
  read anything into its values.
