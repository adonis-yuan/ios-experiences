# One writer for the audio session: replacing "will play" notifications with a call that answers

The bug was easy to state. Start a spoken read-aloud of an article, tap the
Video tab, and the immersive video page starts playing with sound — while the
read-aloud keeps talking over it. If read-aloud was merely *paused*, it stayed
alive in the background instead of being dismissed, though the product rule
says the user's next deliberate playback ends it.

The cause was not a missing call. Coordination between players was done with
two `NotificationCenter` notifications, and the video page simply did not take
part: it wrote `AVAudioSession` directly, as it always had. This change made
the session's owner a thing you *call*, gave the call a return value, and put a
lint rule in front of every remaining direct write.

It is the first of a series of migrations; the background — why read-aloud
outranks everything automatic and what "deliberate" means — is in
[the read-aloud tech design](read-aloud-player-plan.md).

## Why the notifications were the wrong shape

The old protocol had two broadcasts:

- *spoken audio will play* — other players were expected to pause themselves;
- *user playback will start* — posted by a player about to make sound, so that
  read-aloud would close.

Both are fire-and-forget, and that is the defect:

1. **Participation is opt-in and invisible.** A player that never posts and never
   observes is indistinguishable, in review, from one that does. Every new
   video surface is a fresh chance to forget.
2. **A broadcast cannot answer.** A muted autoplay video needs to learn
   *"read-aloud holds the session; stay muted and do not touch it"*. A
   notification can tell everyone something happened; it cannot tell the
   sender what to do next.
3. **Ordering is implicit.** "Pause first, then claim the session" depends on
   observers running synchronously before the poster continues — true for
   `post` on the same thread, but nothing in the type system says so.

## The new contract

![Sequence: the user taps the video tab; the page asks the arbiter for a user-initiated playback; the arbiter releases the spoken session and synchronously closes read-aloud, then the page's category write is allowed. A passive autoplay request gets proceed-muted and its category write is refused](../../../../assets/feed/article/audio/audio-session-single-writer.png)

```swift
public enum AudioPlaybackKind { case userInitiated, passive }

public enum AudioPlaybackDecision: Equatable {
    case proceed
    /// Spoken audio holds the session: play muted, leave the session alone.
    case proceedMuted
}

public protocol SpokenAudioHolder: AnyObject {
    func closeForUserPlayback(source: PlaybackSource)
}

public protocol SpokenAudioParticipant: AnyObject {
    func spokenAudioWillBegin()
}
```

```swift
@discardableResult
public func requestPlayback(source: PlaybackSource,
                            kind: AudioPlaybackKind) -> AudioPlaybackDecision {
    guard isSpokenSessionActive else { return .proceed }
    guard kind == .userInitiated else { return .proceedMuted }
    let holder = spokenHolder
    releaseSpokenSession()
    holder?.closeForUserPlayback(source: source)
    return .proceed
}

@discardableResult
public func setCategoryIfAllowed(_ category: AVAudioSession.Category) -> Bool {
    guard !isSpokenSessionActive else { return false }
    try? session.setCategory(category)
    return true
}
```

Things worth noticing:

- **The caller classifies itself.** *User-initiated* means the user just asked
  for sound; *passive* is autoplay and ads. The arbiter does not guess.
- **A user request closes read-aloud even when it is paused.** The rule is
  "the next deliberate playback ends it", not "the next one that collides with
  sound".
- **The arbiter mutes nothing itself.** `proceedMuted` is an instruction to
  the caller. Reaching into other players' volume from the arbiter would make
  it depend on all of them.
- **Release before callback.** The session is released before the holder is
  told to close, so when `requestPlayback` returns the caller owns the session
  — and the holder's own teardown cannot be blocked by the guard it is
  tearing down.
- **When there is no spoken session, nothing changes.** `requestPlayback`
  returns `.proceed`, `setCategoryIfAllowed` performs exactly the single-argument
  `setCategory` the call site used to make. That equivalence is what lets this
  ship without a new flag (see Tests).
- **Participants and the holder are held weakly**, in an
  `NSHashTable.weakObjects()`. `beginSpokenSession(holder:)` pauses every
  participant synchronously, *then* claims the session — the ordering is now
  in one function body instead of in the semantics of `post`.

## Pages never see the arbiter

Full-screen video pages talk to a three-method facade:

```swift
public struct VideoPageAudio {
    public func pageDidAppear() {
        arbiter.requestPlayback(source: source, kind: .userInitiated)
        arbiter.setCategoryIfAllowed(.playback)
    }
    public func pageWillDisappear()        { arbiter.setCategoryIfAllowed(.ambient) }
    public func playbackMovedToMiniPlayer() { arbiter.setCategoryIfAllowed(.playback) }
}
```

Each page swaps one `try? AVAudioSession.sharedInstance().setCategory(…)` per
lifecycle hook for one facade call. Entering the page closes read-aloud
synchronously; leaving or moving to the mini player can no longer demote a
session read-aloud holds. On the normal path read-aloud is already closed by
then — those guards are the backstop.

## What changed, by scenario

| Scenario | Before | After | Reach |
| --- | --- | --- | --- |
| Read-aloud playing or paused; user enters the immersive, full-screen or short-drama video page | Playing: two voices. Paused: read-aloud survives | Page appear closes read-aloud synchronously, paused or not | 3 `viewDidAppear`s |
| Category writes while read-aloud holds the session (page leave, hand-off to mini player, app launch) | Direct `.ambient` / `.playback` writes could demote it | Skipped | 5 call sites |
| Same 8 call sites with no read-aloud session | Direct write | Byte-for-byte the same write | 8 call sites |
| Audio tab vs read-aloud | Two notifications | Direct calls: read-aloud start pauses the registered audio-tab player; a user play there closes read-aloud; first-story autoplay and auto-next are *passive* — they load but do not play while read-aloud plays | 3 sites + 1 registration |
| Audio tab's dependency | Read the arbiter singleton directly | A delegate protocol, implemented by the arbiter at the app layer; `nil` means "behave as before read-aloud existed" | 1 injection chain |
| Read-aloud controller lifetime | Static `shared` and a static `stopIfActive` | A provider owned by the app delegate, injected into routing; `UIApplication`, `MPRemoteCommandCenter`, `MPNowPlayingInfoCenter` injected through `init` | 3 sites |

### Passive also has an analytics edge

The audio tab labels each play with how it started — autoplay, switch, auto-next.
The label is computed before the start and consumed by the next real start.
When read-aloud blocks the first-story autoplay, the pending "autoplay" label
must be dropped: otherwise the user's next swipe, which really started
playback, gets logged as autoplay. A one-line fix, but exactly the kind a
"passive request may not play" change introduces silently. The denominator of
that metric does not change; one action value now has a rule in a new state.

### Lock screen ownership through the same delegate

The audio tab's Now Playing writes were read-modify-write on the global
`MPNowPlayingInfoCenter`, with artwork arriving asynchronously. The delegate
exposes "do you own Now Playing", and the artwork callback now checks both
that its story is still current *and* that it still owns the lock screen —
on the main queue, at write time, not when the download started.

## Locking the door: a lint rule with an exit list

```yaml
audio_session_direct_write:
  # The owner plus writers not migrated yet; each later PR deletes its own name.
  excluded: '/(SessionArbiter|<six not-yet-migrated files>)\.swift$'
  regex: '(?:\b\w*[Ss]ession[?!]?|sharedInstance\(\))\.(?:setCategory|setActive)\('
  excluded_match_kinds: [comment, doccomment]
  severity: error
```

- **Tree-wide, severity error, from day one.** The exclusion list is the
  migration backlog written down: six files remain, and each follow-up PR's
  definition of done includes deleting its own name. The list can only shrink;
  a new file that writes the session fails the build.
- **The message says what to call instead and why**, citing the user-visible
  bug — a lint message is read at the moment someone is about to reintroduce it.
- **The rule has its own fixture**, kept as `.swift.txt` so neither the
  tree-wide nor the PR-scoped lint sees it. Positive lines carry an
  expected-violation tag; a self-test script asserts every tagged line reports
  and nothing else does. The negatives are the interesting half: a
  `setCategory(.news)` on a UI cell, `setActive(true)` on a chip or a view model,
  a read of `.category`, and a session write mentioned in a comment. A regex
  rule without negatives is a rule nobody can tighten safely.

## Tests

- **Pinning tests for "nothing changes without read-aloud":** one asserts that
  every caller's write, with no spoken session, is identical to its original
  write; another asserts the same for the video page facade. These are what
  let the change ship without a feature flag — the new logic is reachable only
  after read-aloud starts, and read-aloud is itself behind a default-off flag.
- **Behaviour tests:** a user request closes spoken audio and leaves the session
  to the caller; a passive request stays muted and cannot touch the session;
  entering a page closes read-aloud then takes the session; leaving cannot
  demote it; autoplay waits while a user start closes read-aloud; participants
  are held weakly; the provider builds the controller once.
- **Mutation checks** on the four load-bearing lines in the arbiter — the
  passive guard, the release-before-callback, the allowed-write guard, the
  participant pause — each mutation was caught by the matching assertion.
  260 tests green across the affected suites.

### On a device

All passed on a recent iPhone with a debug build and a local mock of the
read-aloud field:

- start, leave the article → bubble; background and foreground → still playing;
- expand the bubble, tap the avatar → back to the source article;
- read-aloud playing **or paused**, tap the Video tab or open a video from the
  feed → read-aloud closes, only the video is heard;
- read-aloud playing, scroll past muted autoplay video → unaffected;
- enter and leave a video page first, then start read-aloud, flip the mute
  switch → still audible (the page's `.ambient` write no longer leaks forward);
- log out or switch language during playback → stops;
- lock-screen controls work; another app playing, start read-aloud → the other
  app stops; start another app during read-aloud → read-aloud pauses and can be
  resumed.

## What this did not fix

- **A 0.35 s overlap on push.** During the push transition into the immersive
  page, the video is already audible; read-aloud closes only at `viewDidAppear`.
  The right fix is to make "this player is about to be audible" a single choke
  point in the video controller, and to ask the arbiter there. That is a
  follow-up, not a tweak to page lifecycle timing.
- **Full-screen and short-drama pages** were not separately verified on device;
  they make the identical facade call as the immersive page.
- **The audio tab** was kept regression-free but not device-tested; it is
  scheduled for removal.
- **Six files, seventeen lines** still write the session directly: the video
  controller and player, radio, community video, the map's audio player, and the
  audio tab's own activation. Ads, related videos, mini player, in-article
  HTML5 media and recording follow in their own PRs.

## What happened next

The follow-up migrations have since landed, and they changed the shape a little:

- **The exclusion list emptied.** Only the arbiter and the audio tab, which is
  being removed rather than migrated, still write the session.
- **Video start got its single choke point.** "About to be audible" is now
  decided in one place in the video controller instead of in page lifecycle
  hooks — the fix the 0.35 s overlap above was waiting for.
- **The facade pattern generalized into injection.** Each feature module
  declares only a narrow protocol of its own (video, radio, interstitial,
  recording, upload overlay); the app delegate injects the implementations at
  launch. No module references the arbiter, and a second lint rule forbids it.
  Ad speaker toggles got the same treatment: one function, one lint rule.
- **The classification is the easy thing to get wrong.** After the migration,
  the immersive and short-drama pages *took over* an already-playing video
  without marking the request user-initiated — so read-aloud survived. Review
  caught it. Every new call site of `requestPlayback` deserves the question
  "did the user just ask for sound?"

## Takeaways

- If a coordination protocol needs an answer, it must be a call, not a
  broadcast. "Should I be muted?" is a question.
- Make callers declare *why* they are playing. The arbiter's whole policy fits
  in two `guard`s once intent is part of the request.
- Put the backlog in the lint config. An exclusion list that each migration PR
  shortens turns "we will get to it" into a build failure for anyone adding to
  the problem.
- Ship a refactor without a flag only when you can prove — with a pinning test
  and a mutation check, not an argument — that the old path is byte-for-byte
  unchanged.
