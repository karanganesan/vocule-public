# vocule api reference

draft public api for the **2.0.0 release candidate**. published 1.x keeps its implicit redux factory; new-major examples below require 2.0.0. the declarations in `node_modules/@karanganesan/vocule/dist/*.d.ts` are authoritative when this file disagrees with them.

## contents

- [createSpeech](#createspeechoptions)
- [the speech instance](#the-speech-instance)
- [one call at a time](#one-call-at-a-time)
- [transcribe](#transcribeaudio-options)
- [the transcript](#the-transcript)
- [realtime and listen](#realtimeoptions-and-listenoptions)
- [record](#recordoptions)
- [errors](#errors)
- [other entry points](#other-entry-points)

## `createSpeech(options)`

```ts
import { createSpeech } from "@karanganesan/vocule";

const speech = createSpeech({
  model: "fermionresearch/phonon-2",
  onProgress: (event) => console.log(event),
});
```

the factory makes no network request and asks for no permission. **`model` is compulsory from 2.0.0**, entirely lowercase, and immutable for the instance. missing options/model, `undefined`, an empty string, a mixed-case or unknown id throws `SpeechError("CAPABILITY", ...)` synchronously before workers, downloads or capture. there is no sdk default and no automatic switch to the other model.

```ts
// published 1.x only: createSpeech() or createSpeech({ onProgress })
// 2.0.0: retain the old checkpoint deliberately
const redux = createSpeech({ model: "moondream/parakeet-redux" });
```

all other options, methods, events and ownership rules are common. select again by stopping/cancelling active work, disposing, and creating a new instance when the user explicitly starts it. mutating the old options object cannot retarget an instance.

| option | type | default | effect |
| --- | --- | --- | --- |
| `model` | `ModelId`: `"fermionresearch/phonon-2"` or `"moondream/parakeet-redux"` | required; none | selects the pinned checkpoint for this instance |
| `onProgress` | `(event: Progress) => void` | none | receives preparation and inference progress |
| `cache` | `boolean` | `true` | keeps the verified model in the browser's cache storage for later visits |
| `modelBaseURL` | `string` | vocule's cdn, then hugging face | a directory serving the selected descriptor’s exact pinned delivery representation, used instead of default sources with no fallback; checkpoint and metadata hashes still apply |
| `artifactHashes` | `ReduxHashes` or `PhononHashes`, discriminated by `model` | pinned hashes | redux: `config.json`, `ternary.json`, `tokenizer.json`; phonon: `config.json`, `packed_manifest.json`. metadata overrides do not change checkpoint, architecture or weight pins |
| `assetBaseURL` | `string \| URL` | built-in worker and worklet | a same-origin directory holding the root `dist/` or the matching lean model directory, for strict content security policies |
| `workerFactory` | `() => Worker` | none | zero-argument factory; takes priority over `assetBaseURL`. the supplied worker must handshake with this version and supported model |

`Progress` is `{ phase, completed?, total?, detail? }`:

| phase | when | `completed` / `total` |
| --- | --- | --- |
| `download` | preparing, when the model is not cached | bytes of the model file |
| `verify`, `load` | preparing | absent; `detail` may describe the step |
| `features`, `encode`, `decode` | during every transcription, including live drafts | not a user-facing percentage |

## the speech instance

| member | type | notes |
| --- | --- | --- |
| `capabilities` | `Capabilities` | today `{ runtime: "webgpu-wasm", timestamps: ["none"], maxAudioSeconds: null, liveAudio: true, cloudFallback: false }` |
| `ready` | `boolean` | whether this instance has the model prepared |
| `prepare({ signal })` | `Promise<void>` | downloads, verifies, caches and warms the model |
| `transcribe(audio, options)` | `Promise<Transcript>` | prepares on demand, then transcribes complete audio |
| `realtime(options)` | `RealtimeSession` | synchronous; a live session fed with pcm through `push()` |
| `listen(options)` | `Promise<RealtimeSession>` | a live session fed by the microphone or a supplied `MediaStream` |
| `record(options)` | `Promise<RecordingSession>` | captures a clip and prepares the model in the background |
| `dispose()` | `void` | cancels active work and releases the instance |

the stable runtime family is `"webgpu-wasm"`, not a report of individual stage placement. no full-model wasm-only/webgl or cloud inference fallback is supplied.

after a cancelled call or a gpu failure, the next call prepares the model again; with the model cached, that needs no download.

## one call at a time

an instance runs one inference call at a time. `transcribe()` and live sessions from `realtime()` or `listen()` exclude one another; an overlapping inference call fails with `BUSY` (`realtime()` throws synchronously; the others reject). `prepare()` is not exclusive: callers join the same in-progress preparation, including when an inference call or recording started it. `transcribe()`, `listen()` and `realtime()` prepare on demand.

| already running | `prepare()` | `transcribe()` | `realtime()` / `listen()` | `record()` |
| --- | --- | --- | --- | --- |
| `prepare()` | joins | starts and joins preparation | starts and joins preparation | starts |
| `transcribe()` | joins or returns ready | `BUSY` | `BUSY` | starts |
| live session | joins or returns ready | `BUSY` | `BUSY` | starts |
| recording only | joins or returns ready | starts | starts | starts |

capturing a recording does not use the model, so `record()` can start alongside other work and begins preparation in the background. `clip.transcribe()` is a `transcribe()` call and follows the table. to run inference at the same time, create another instance; each one loads its own copy of the model.

## `transcribe(audio, options)`

| option | type | default | effect |
| --- | --- | --- | --- |
| `signal` | `AbortSignal` | none | cancels the call with `ABORTED` |
| `sampleRate` | `number` | none | the source rate of a bare `Float32Array`; use it only for that input |
| `timestamps` | `"none" \| "segment" \| "word"` | `"none"` | only `"none"` is supported; the others throw `CAPABILITY` |
| `profileGpu` | `boolean` | `false` | adds a gpu timing trace to `metrics.gpuProfile` where the browser allows it |

| input | example |
| --- | --- |
| `File`, `Blob` | `transcribe(file)` |
| url string, `URL` | `transcribe("/clips/intro.mp3")`; cors applies across origins |
| `Request` | `transcribe(new Request(url, { headers }))` |
| `Response` | `transcribe(await fetch(url))` |
| encoded bytes: `ArrayBuffer`, `Uint8Array`, `DataView` | `transcribe(bytes)` |
| `AudioBuffer` | `transcribe(buffer)` |
| webcodecs `AudioData` | `transcribe(frame)`; the caller still closes the frame |
| a file handle | `transcribe(handle)`, for a `showOpenFilePicker()` result |
| `{ samples: Float32Array, sampleRate }` | `transcribe({ samples, sampleRate: 48_000 })` |
| `{ samples: Int16Array, sampleRate }` | `transcribe({ samples: int16, sampleRate: 16_000 })` |
| `{ channels: Float32Array[], sampleRate }` | `transcribe({ channels, sampleRate })` |
| `Float32Array` | `transcribe(samples, { sampleRate: 16_000 })` |
| a finite `AsyncIterable` or `ReadableStream` of pcm, `AudioData` or `Uint8Array` chunks | `transcribe(chunks)`; one sample rate per stream, and one text at the end; use `realtime()` for incremental text |

- uncompressed wav is read directly; other formats use the browser's decoder, so codec and container support depends on the browser. multichannel audio is mixed to mono and other sample rates are converted to 16 khz.
- long recordings are split at natural pauses automatically, with no duration cap, but the whole input is held in memory.
- a `MediaStream` has no natural end; it belongs in `listen()` or `record()`.

## the transcript

| field | type | notes |
| --- | --- | --- |
| `text` | `string` | the transcript |
| `segments` | `Segment[]` | empty, because timestamps are not produced |
| `model` | `{ id, revision, sha256 }` | source checkpoint identity, independent of cdn/archive/pack transport; lowercase public id |
| `runtime` | `"webgpu-wasm"` | |
| `verified` | `boolean` | currently always `false`; not a success flag. aggregate benchmark evidence does not certify an individual transcript |
| `metrics.audioSeconds` | `number` | input duration |
| `metrics.wallMs` | `number` | time for the whole call |
| `metrics.inferenceMs` | `number` | time spent transcribing |
| `metrics.realtimeFactor` | `number` | processing time divided by audio duration, so lower is faster: distinct from complete-call throughput `audioSeconds / (wallMs / 1000)` |
| `metrics.windows` | `{ startSeconds, endSeconds }[]` | the windows used for a long recording |
| `warnings` | `string[]` | notes about the result; today every transcript carries the same standing notes, so log them instead of showing them as errors |

### selected checkpoint identity

| public id | source revision | source checkpoint sha-256 |
| --- | --- | --- |
| `fermionresearch/phonon-2` | `e357655f6325aa70d4800d125273a5ed3a703b9c` | `4b6bfa3a12cc3c4e0a54f2ab3ec4ca7a842b09e5c7ecfc8e7ca0ac6cc8c11468` (`model.fermion`) |
| `moondream/parakeet-redux` | `a049528989ebbc6097c6478551266f1e3d26d1b3` | `78ec25733ee0d0c1586d1346fc86db9d0c2e436e3a8ab1d32a82d1bb8f848d21` (`model.safetensors`) |

phonon’s canonical source id is `FermionResearch/Phonon-2`; preserve that case only in source urls and provenance. archive and deterministic repack hashes live in descriptors/provenance, not in `Transcript.model`.

the source checkpoint keeps its original bytes and every reconstructed weight value. exact engine/cache layouts may have their own format version and storage hash while preserving the source identity and values. benchmark speed and quality as separate measurements.

## `realtime(options)` and `listen(options)`

both return a `RealtimeSession` and hold the instance until it ends. a session starts preparing the model as soon as it opens, and audio that arrives first waits for it; that waiting audio counts toward `maxQueuedSeconds`. to start with a ready model, await the preparation first.

| option | type | default | effect |
| --- | --- | --- | --- |
| `onUpdate` | `(update: RealtimeUpdate) => void` | none | receives every draft, settled and final view |
| `onError` | `(error: unknown) => void` | none | reports a background failure; `stop()` also rejects |
| `signal` | `AbortSignal` | none | cancels the session |
| `draftIntervalMs` | `number` | `100` | time between draft attempts while speech is active; `0` turns drafts off |
| `vad` | `boolean` | `true` | `false` settles fixed-length segments without detecting speech |
| `speechThreshold` | `number` | `0.006` | rms level that opens an utterance; the shared live energy gate is separate from a model-specific long-file planner |
| `silenceMs` | `number` | `700` | trailing silence that settles an utterance |
| `minSpeechMs` | `number` | `120` | continuous speech needed to open an utterance |
| `preRollMs` | `number` | `240` | audio kept from before speech started |
| `maxUtteranceMs` | `number` | `18000` | hard segment boundary; no audio is dropped |
| `maxQueuedSeconds` | `number` | `45` | buffered audio allowed before `BACKPRESSURE` |

`listen()` also takes:

| option | type | default | effect |
| --- | --- | --- | --- |
| `stream` | `MediaStream` | none | capture this stream instead of requesting the microphone; a supplied stream stays the caller's |
| `audio` | `MediaTrackConstraints` | constraints that favor unprocessed audio | used only without `stream` |
| `workletURL` | `string \| URL` | built-in worklet | the capture worklet module, for strict policies |
| `captureSampleRate` | `number` | `16000` | requested capture rate |
| `onLevel` | `(rms: number) => void` | none | input level, for a meter |

`RealtimeSession`:

| member | notes |
| --- | --- |
| `text`, `settledText`, `draftText`, `segments` | the current view |
| `push({ samples, sampleRate })` | `realtime()` input: consecutive mono pcm chunks that share one sample rate |
| `stop()` | finishes pending work and resolves to the settled text |
| `cancel()` | discards pending text and releases capture |

`RealtimeUpdate` is `{ kind, text, settledText, draftText, segments }`, where `kind` is `"draft"`, `"settled"` or `"final"`, and `text` is the settled text followed by the draft. a `RealtimeSegment` is `{ text, startSeconds, endSeconds, reason }`, with `reason` one of `"silence"`, `"limit"` or `"stop"`; its times are capture times, not word timestamps. live drafts are re-transcribed windows of recent audio, not streaming tokens, and the draft interval is a schedule, not a latency promise.

`listen()` asks for the microphone only when called and needs https or localhost. when the microphone is refused or missing, it rejects with the browser's `DOMException`, such as `NotAllowedError` or `NotFoundError`.

## `record(options)`

`record()` takes `signal`, `stream`, `audio`, `workletURL`, `captureSampleRate`, `onLevel` and `onError`, with the same meaning as for `listen()`. capture never waits for the model, and if the background preparation fails, `clip.transcribe()` prepares again and reports any error.

`RecordingSession`:

| member | notes |
| --- | --- |
| `state` | `"recording"`, `"paused"`, `"stopping"`, `"stopped"` or `"cancelled"` |
| `durationSeconds` | captured audio so far |
| `pause()`, `resume()` | leave the paused audio out of the clip |
| `stop()` | resolves to a `Recording` |
| `cancel()` | releases capture and discards the clip |

`Recording` is also a valid `transcribe()` input:

| member | notes |
| --- | --- |
| `samples`, `sampleRate` | the uncompressed clip |
| `durationSeconds` | its length |
| `wav` | a lazily encoded wav `Blob` |
| `play()` | starts playback and resolves to the `HTMLAudioElement` for pausing and seeking |
| `transcribe(options)` | transcribes with the instance that recorded it |

## errors

engine failures use `SpeechError` (`import { SpeechError } from "@karanganesan/vocule"`) with a stable `code` of type `ErrorCode`; the message is for people. microphone failures can also arrive as the browser's own `DOMException`, as described under `listen()`.

| code | meaning |
| --- | --- |
| `ABORTED` | the call or session was cancelled, including by `dispose()` |
| `AUDIO_INVALID` | the input could not be read or decoded |
| `AUDIO_TOO_LONG` | reserved; complete files have no duration cap, so no current call raises it |
| `BACKEND_UNAVAILABLE` | no usable webgpu adapter, or the worker could not start |
| `CAPABILITY` | missing/invalid model at the factory; wrong/stale or unsupported-model worker; unsupported options or missing browser features |
| `NETWORK` | the model could not be downloaded |
| `INTEGRITY` | downloaded bytes did not match the pinned hashes |
| `MODEL_FORMAT` | the model files were not what the engine expects |
| `BUSY` | another inference call is running on this instance, or a session is not in the right state for the call |
| `BACKPRESSURE` | live audio arrived faster than it could be transcribed |
| `DISPOSED` | the instance was disposed |
| `GPU_LOST`, `GPU_ERROR` | the gpu device was lost or failed; the next call prepares again |
| `DECODE_LIMIT` | a safety limit stopped decoding; no truncated transcript is returned |

## other entry points

the root entry exports the public types, including `ModelId`, `SpeechOptions`, `BrowserOptions`, `ReduxHashes`, `PhononHashes` and existing audio/session/result types. metadata helpers are also available without an engine import:

```ts
import { getModelDescriptor, getModelCacheStatus } from "@karanganesan/vocule/model";

const descriptor = getModelDescriptor("fermionresearch/phonon-2");
const cache = await getModelCacheStatus(descriptor.id); // presence only, no fetch/hash
console.log(descriptor.downloadBytes, cache.complete);
```

`ModelDescriptor` is immutable. it includes lowercase `id`, canonical `sourceId`, label, revision, source checkpoint `sha256`/`bytes`, `downloadBytes`, format/pack version, sample rate, license/attribution/source link, delivery sources, runtime artifacts with names/bytes/hashes/mime, cache identity and capabilities. `getModelCacheStatus(id)` reports `{ weights, metadata, complete }` as a best-effort presence hint. preparation still verifies cached bytes; a complete hint is not an integrity promise. readiness in memory does not prove persistent cache writes succeeded. neither helper creates a worker, downloads weights or asks for a microphone.

`MODEL` is deprecated fixed-redux metadata. legacy `MODEL_BASE_URL`, `MODEL_CDN_URL` and `MODEL_SOURCES` remain fixed redux exports; they never fill a missing factory choice.

| import | use |
| --- | --- |
| `@karanganesan/vocule/model` | explicit metadata/cache lookup plus fixed-redux legacy constants; no engine graph |
| `@karanganesan/vocule/phonon-2` | lean factory, with required matching `model: "fermionresearch/phonon-2"`; excludes the redux adapter/worker assets |
| `@karanganesan/vocule/parakeet-redux` | lean factory, with required matching `model: "moondream/parakeet-redux"`; excludes the phonon adapter/reader/worker assets |
| `@karanganesan/vocule/microphone` | `captureMicrophone()` for a short clip (up to 18 seconds) and `streamMicrophoneAudio()` for raw pcm blocks |
| `@karanganesan/vocule/video` | `captureVideo({ source: "camera" \| "screen" })` returns video and raw audio; a screen share must include audio |
| `@karanganesan/vocule/diagnostics` | `runDiagnostics()`, a browser webgpu/webassembly self-test |
| `@karanganesan/vocule/worker` | root hosted/custom worker dispatcher |
| `@karanganesan/vocule/phonon-2/worker` | matching phonon hosted/custom worker |
| `@karanganesan/vocule/parakeet-redux/worker` | matching redux hosted/custom worker |

```ts
import { createSpeech } from "@karanganesan/vocule/phonon-2";
const speech = createSpeech({ model: "fermionresearch/phonon-2" });
```

root lazy dispatch and lean exclusion are separate properties. a runtime model string at the root does not promise that the other engine disappears from the emitted package. metadata-only 1.0.0 was measured at **959 bytes** in the historical consumer build; the final packed 2.0.0 metadata/root/lean emitted-byte measurements are pending and must be rerun before publishing new size claims. prefer `speech.record()` for recordings longer than the short capture helper limit.
