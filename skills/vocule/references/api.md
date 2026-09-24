# vocule api reference

the public api of vocule 0.0.5, as published on npm. the declarations in `node_modules/vocule/dist/*.d.ts` are authoritative when this file disagrees with them.

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
import { createSpeech } from "vocule";

const speech = createSpeech({ onProgress: (event) => console.log(event) });
```

the factory makes no network request and asks for no permission.

| option | type | default | effect |
| --- | --- | --- | --- |
| `onProgress` | `(event: Progress) => void` | none | receives preparation and inference progress |
| `cache` | `boolean` | `true` | keeps the verified model in the browser's cache storage for later visits |
| `modelBaseURL` | `string` | vocule's cdn, then hugging face | a directory serving the pinned revision's `config.json`, `ternary.json`, `tokenizer.json` and `model.safetensors`, used instead of the defaults with no fallback; the pinned hashes still apply |
| `artifactHashes` | `Record<"config.json" \| "ternary.json" \| "tokenizer.json", string>` | pinned hashes | only for a mirror that intentionally serves different metadata bytes |
| `assetBaseURL` | `string \| URL` | built-in worker and worklet | a same-origin directory holding a copy of the package's `dist/`, for strict content security policies |
| `workerFactory` | `() => Worker` | none | builds the worker yourself; takes priority over `assetBaseURL` |

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

after a cancelled call or a gpu failure, the next call prepares the model again; with the model cached, that needs no download.

## one call at a time

an instance runs one model operation at a time. `prepare()`, `transcribe()` and a live session from `realtime()` or `listen()` each hold it until they settle, and a call that overlaps them throws `BUSY` instead of waiting (`realtime()` throws synchronously; the others reject).

| already running | `prepare()` | `transcribe()` | `realtime()` / `listen()` | `record()` |
| --- | --- | --- | --- | --- |
| `prepare()` | `BUSY` | `BUSY` | `BUSY` | starts |
| `transcribe()` | `BUSY` | `BUSY` | `BUSY` | starts |
| live session | `BUSY` | `BUSY` | `BUSY` | starts |
| recording only | starts | starts | starts | starts |

capturing a recording does not use the model, so `record()` can start alongside other work. `clip.transcribe()` is a `transcribe()` call and follows the table. to run model work at the same time, create another instance; each one loads its own copy of the model.

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
| `model` | `{ id, revision, sha256 }` | the checkpoint that produced it |
| `runtime` | `"webgpu-wasm"` | |
| `verified` | `boolean` | stays `false` until a broad accuracy evaluation exists; not a success flag |
| `metrics.audioSeconds` | `number` | input duration |
| `metrics.wallMs` | `number` | time for the whole call |
| `metrics.inferenceMs` | `number` | time spent transcribing |
| `metrics.realtimeFactor` | `number` | processing time divided by audio duration, so lower is faster: `0.0175` is about 57× real time (`1 / realtimeFactor`) |
| `metrics.windows` | `{ startSeconds, endSeconds }[]` | the windows used for a long recording |
| `warnings` | `string[]` | notes about the result; today every transcript carries the same standing notes, so log them instead of showing them as errors |

## `realtime(options)` and `listen(options)`

both return a `RealtimeSession` and hold the instance until it ends. a session starts preparing the model as soon as it opens, and audio that arrives first waits for it; that waiting audio counts toward `maxQueuedSeconds`. to start with a ready model, await the preparation first.

| option | type | default | effect |
| --- | --- | --- | --- |
| `onUpdate` | `(update: RealtimeUpdate) => void` | none | receives every draft, settled and final view |
| `onError` | `(error: unknown) => void` | none | reports a background failure; `stop()` also rejects |
| `signal` | `AbortSignal` | none | cancels the session |
| `draftIntervalMs` | `number` | `100` | time between draft attempts while speech is active; `0` turns drafts off |
| `vad` | `boolean` | `true` | `false` settles fixed-length segments without detecting speech |
| `speechThreshold` | `number` | `0.006` | rms level that opens an utterance |
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

every vocule failure is a `SpeechError` (`import { SpeechError } from "vocule"`) with a stable `code` of type `ErrorCode`; the message is for people.

| code | meaning |
| --- | --- |
| `ABORTED` | the call or session was cancelled, including by `dispose()` |
| `AUDIO_INVALID` | the input could not be read or decoded |
| `AUDIO_TOO_LONG` | reserved; complete files have no duration cap, so no current call raises it |
| `BACKEND_UNAVAILABLE` | no usable webgpu adapter, or the worker could not start |
| `CAPABILITY` | an unsupported option, such as timestamps, or a missing browser capability, such as an audio track or a capture api |
| `NETWORK` | the model could not be downloaded |
| `INTEGRITY` | downloaded bytes did not match the pinned hashes |
| `MODEL_FORMAT` | the model files were not what the engine expects |
| `BUSY` | another operation is running on this instance, or a session is not in the right state for the call |
| `BACKPRESSURE` | live audio arrived faster than it could be transcribed |
| `DISPOSED` | the instance was disposed |
| `GPU_LOST`, `GPU_ERROR` | the gpu device was lost or failed; the next call prepares again |
| `DECODE_LIMIT` | a safety limit stopped decoding; no truncated transcript is returned |

## other entry points

the root `vocule` entry also exports `MODEL` (the pinned model's id, revision, hash, size and license) and every public type, such as `AudioInput`, `ListenOptions`, `Progress`, `RealtimeUpdate`, `Recording`, `RecordingSession`, `Speech`, `SpeechOptions` and `Transcript`.

| import | use |
| --- | --- |
| `vocule/model` | `MODEL` metadata and the model's source urls (`MODEL_SOURCES`), without the engine; about 1 kb |
| `vocule/microphone` | `captureMicrophone()` for a short clip (up to 18 seconds) and `streamMicrophoneAudio()` for raw pcm blocks |
| `vocule/video` | `captureVideo({ source: "camera" \| "screen" })` returns the video and its raw audio, which goes straight to `transcribe()`; a screen share must include audio |
| `vocule/diagnostics` | `runDiagnostics()`, a quick self-test of this browser's webgpu and webassembly support |
| `vocule/worker` | the worker module, for `workerFactory` or your own hosting |

prefer `speech.record()` over the capture helpers for anything longer than a short clip. an import of `MODEL` alone stays about 1 kb; the engine loads when inference starts.
