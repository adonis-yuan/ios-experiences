# On-device TTS for read-aloud: Kokoro, Matcha and the system voice, measured

The [read-aloud plan](read-aloud-player-plan.md) assumed a server pipeline
would narrate each article and the app would only play an m4a. That pipeline
was the critical path, and it had not started. So in parallel we tried the
other end: synthesize speech on the phone, from the article text the app
already has.

Three candidates:

- **The iOS system voice** (`AVSpeechSynthesizer`). Free, already on the
  device, no download.
- **Kokoro** (int8, English), a small open-weight TTS model, run through
  [sherpa-onnx](https://github.com/k2-fsa/sherpa-onnx) and ONNX Runtime.
- **Matcha-TTS with the Vocos vocoder**, through the same runtime.

Both open models went only into internal builds (debug, beta, daily). The App
Store build carried none of it. This article is the measurements, plus two
lessons from taking them.

All latency and speed numbers come from the **iOS simulator on an
Apple-silicon Mac** (4 threads). Nothing here was measured on a physical
device yet; expect a phone to be several times slower.

## Summary

- **Streaming is not optional for a neural model.** Synthesizing a whole
  article before playing took Kokoro 164 s. Playing the first sentence while
  the rest synthesizes brought tap-to-sound to 4.8 s. The system voice starts
  in 0.19 s and Matcha in 0.25 s.
- **Accuracy: Kokoro matches the baseline, Matcha is about a point worse.**
  Word error rate was 4.00% for Kokoro, 4.09% for the baseline voice and
  4.97% for Matcha. None of them skipped a sentence or made anything up.
- **Matcha is ~20× faster than Kokoro but speaks ~22% slower**, so the same
  article comes out as longer audio.
- **The cost moved into the build.** ~300 MB of models plus a 190 MB SDK. A CI
  checkout that wiped the model directory re-downloaded all of it on every PR
  run, until only the internal-build lanes fetched it.

![Two charts. Left: tap to first audio for one 987-word article; with streaming off, the system voice takes 3.55 s, Matcha 9.81 s, Kokoro 164 s; with streaming on, 0.19 s, 0.25 s and 4.82 s. Right: word error rate; macOS baseline 4.09%, Matcha 4.97%, Kokoro 4.00%](../../../../assets/feed/article/audio/on-device-tts-engines.png)

## 1. Tap to first audio

One 987-word article, the cache cleared before each run, timed from the tap
to the start of playback.

| Engine | Streaming off | Streaming on | First model load | Audio length |
| --- | --- | --- | --- | --- |
| iOS system voice | 3.55 s | 0.19 s | — | 328 s |
| Matcha + Vocos | 9.81 s | 0.25 s | 0.58 s | 398 s |
| Kokoro | 164 s | 4.82 s | 0.35 s | 328 s |

For the two open models, the "off" column includes loading the model for the
first time; the "on" column was measured with the model already loaded.

Two things in this table are easy to misread:

- **Matcha's "off" number is worse than its sentence benchmark predicts.**
  Matcha reads about 22% slower, so the same text becomes 398 s of audio
  instead of 328 s, and there is proportionally more to synthesize. Measure
  on a whole article, not by extrapolating from one sentence.
- **The simulator flatters every engine.** A real phone is expected to be
  several times slower, which makes streaming matter even more there.

## 2. Synthesis speed

Six news sentences, about 30 s of audio. RTF (real-time factor) is synthesis
time divided by audio length: below 1 is faster than real time.

| Engine | Audio | Synthesis | RTF | First sentence | Model load |
| --- | --- | --- | --- | --- | --- |
| Kokoro | 29.9 s | 14.6 s | 0.487 | 2.53 s | 0.36 s |
| Matcha + Vocos | 36.7 s | 0.86 s | 0.023 | 0.19 s | 0.56 s |

An RTF of 0.49 is comfortably fast enough to stream: synthesis stays ahead of
playback once the first sentence is out. It is not fast enough to synthesize
up front, which is the whole of §1.

## 3. Accuracy: word error rate

### Method

How accurately does each engine read a news article? Measure it by round
trip:

1. Take 50 real news articles, the first 6 paragraphs of each — 215
   paragraphs.
2. Synthesize every paragraph with each engine.
3. Transcribe the audio back to text with Whisper `small.en`.
4. Normalize both texts and compute word error rate against the original.

The baseline is the macOS system voice. Its error rate mostly measures the
transcriber and the normalization, not the voice, so what matters is how far
each engine sits **above** the baseline.

### Results

| Engine | WER | vs baseline | Substitutions | Deletions | Insertions | Paragraphs > 20% WER | Skips or hallucinations | Failed | Audio | Synthesis | First-pass WER |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| macOS voice (baseline) | 4.09% | — | 216 | 49 | 46 | 4 | 0 | 0 | 45.3 min | 3.3 min | 6.38% |
| Matcha + Vocos | 4.97% | +0.88 pp | 256 | 39 | 83 | 14 | 0 | 0 | 55.3 min | 0.65 min | 7.23% |
| Kokoro | 4.00% | −0.09 pp | 197 | 30 | 77 | 9 | 0 | 0 | 45.2 min | 21.8 min | 6.26% |

The audio column shows Matcha's slower speech again: 22% more audio for the
same text.

### The first pass was wrong by two points

The last column is the first run. Every engine looked about 2 points worse,
and none of it was the engines. The normalizer did not fold curly quotes into
straight ones, and it dropped text inside parentheses. Both inflate the error for every engine equally, which is why the ranking
looked plausible and nothing seemed off.

The lesson: before comparing engines, read the worst-scoring paragraphs of
the **baseline**. Any error the baseline shares with every engine is a
measurement error until shown otherwise.

### What the engines get wrong

Matcha and Kokoro both use espeak to turn text into phonemes, so they fail in
the same places, and the same fix applies to both: a custom pronunciation
lexicon in front of the phonemizer.

- **People's names.** Unusual surnames get an English-looking guess.
- **Brand names written as one word.** Names like ShopRite, VisionQuest or
  Butcherbird are read only in part, or split in the wrong place.
- **Numbers and symbols.** Prices like `$80`, abbreviations like `7s`; `©` is
  read as "copyright" and `/` as "slash".

### Limits of this measurement

- It measures **whether the words are right, not whether the voice sounds
  natural.** Those need a listening test.
- The macOS baseline voice is not identical to the high-quality voice the app
  would use on iOS.
- **Differences under one percentage point are within measurement noise.**
  "Kokoro equals the baseline" is the safe reading; "Kokoro beats it" is not.
- A hosted model (OpenAI's TTS) has not been run yet. The harness already
  supports it; it needs an API key and the same 50 articles.

## 4. What it costs in the build

### Size

| Item | Size |
| --- | --- |
| Kokoro model (int8, English) | 152 MB |
| Matcha model + Vocos vocoder | 151 MB |
| sherpa-onnx + ONNX Runtime SDK, uncompressed | 193.5 MB |
| The same SDK, compressed in git | 39.2 MB |
| Code linked into internal builds | 38 MB |

None of this is in the App Store build.

### CI: the 303 MB re-download

The models were fetched during the CocoaPods step. The CI checkout cleaned the
model directory, so **every PR run downloaded all 303 MB again**:

| Run | Prepare CocoaPods | Install CocoaPods |
| --- | --- | --- |
| This change, run 1 | 40 s | 15 s |
| This change, run 2 | 70 s | 37 s |
| This change, run 3 | 83 s | 42 s |
| Another PR, same period | 1 s | 10 s |
| Another PR, same period | 1 s | 14 s |

One to two minutes per run, on every PR, for files no PR build used. The fix
was to fetch the models only in the CI lanes that produce internal builds.
Ordinary PR builds no longer pay for them.

### Git LFS, if the SDK moved there

Moving the 0.19 GB SDK into Git LFS was considered. The organization's plan
includes 250 GB of LFS storage and 250 GB of bandwidth a month; it was using
under 40% of the storage and none of the bandwidth. Past the included amount,
GitHub charges $0.07/GiB for storage and $0.0875/GiB for bandwidth. With
normal CI caching, LFS would cost nothing extra. If CI re-downloaded on every
run, as above, the bandwidth could exceed the quota by roughly $75 a month.
**Fix the cache before deciding where the bytes live.**

## 5. For comparison: doing it on the server

A rough estimate, not a bill. It assumes 10,000 articles a day, about 6,000
characters and 5.5 minutes of audio each. Prices are public list prices.

| Option | Pricing | Per article | Per month |
| --- | --- | --- | --- |
| OpenAI `tts-1` | $15 per million characters | $0.09 | ~$27,000 |
| OpenAI `gpt-4o-mini-tts` | ~$0.015 per minute | $0.08 | ~$25,000 |
| Self-hosted Kokoro, one L4-class GPU | ~$0.8 per hour | — | ~$600 |

The self-hosted row uses Kokoro's own claim of 35–49× real time on a GPU, at
which one card handles about 10,000 articles a day. Real volume has to come
from the server side, and the self-hosted number excludes engineering and
operations time.

The comparison that matters: the open model that reads as accurately as the
system voice on a phone is the same model that makes server-side narration
about 40× cheaper than a hosted API.

## 6. Where this leaves the choice

What is measured:

- The **system voice** is instant, free and accurate. Its weakness is
  quality, which this study did not measure.
- **Kokoro** reads as accurately as the baseline and only works with
  streaming. On a real phone its RTF may approach or pass 1, at which point
  even streaming stalls.
- **Matcha** is fast enough to be indifferent to device speed, at the price
  of about a point of accuracy and noticeably slower speech.

What is not measured yet, and decides it:

1. **A physical device**, oldest supported model included. Kokoro's RTF there
   is the first number to get.
2. **A listening test** for naturalness, since WER says nothing about it.
3. **The hosted API** on the same 50 articles.
4. **The pronunciation lexicon** — names, brands, numbers — and its effect on
   the error rate.
