---
name: vocule
description: Add private, on-device speech to text to a web app with vocule (npm package `vocule`), which runs the parakeet redux model in the browser on webgpu and webassembly, with no server. Covers transcribing audio and video files, live microphone transcription for dictation, captions and voice input, recording clips and transcribing them, model download progress, errors, content security policy and self-hosting, in react, next.js, vue, nuxt, svelte, sveltekit, angular, astro or plain typescript. Use when the user mentions vocule, createSpeech or speech.transcribe, listen, record or realtime, or wants speech recognition, transcription or voice notes that run in the browser without uploading audio.
---

# vocule

vocule is private speech to text for web apps. one object from `createSpeech()` downloads a pinned 178 mb model once, keeps it in the browser's cache, and transcribes complete audio, live speech and recordings on the device, in a web worker, on webgpu and webassembly. audio never leaves the page: there is no server, api key or cloud fallback.

this skill describes vocule 0.0.5. the installed package is the source of truth: if anything here disagrees with `node_modules/vocule/dist/index.d.ts`, follow the types, which declare every public option, result and error with comments.

## check the fit first

vocule is for browser and web apps only. before writing code, tell the user plainly if the request needs any of these, and do not build workarounds:

- **a server, node, edge function, react native or native app.** every model call needs a browser page with a worker, webgpu and webassembly.
- **timestamps, subtitles or speaker labels.** transcripts are untimed text. `timestamps: "segment"` or `"word"` throws `CAPABILITY`, and `transcript.segments` is always empty.
- **browsers without webgpu.** there is no cpu or webgl fallback. vocule needs webgpu, webassembly and a secure context (https or localhost). it was tested in chrome 153 and safari 26.6 on apple silicon; test any other browser or device before calling it supported.
- **choosing a language or translating.** there is no language option. the model lists 25 languages upstream; english, spanish and french were tested.
- **a small first load.** the first `prepare()` downloads about 178 mb; later visits load it from the browser's cache.

## integrate

1. **install** with the project's package manager: `npm install vocule`, `pnpm add vocule`, `yarn add vocule` or `bun add vocule`. there is no framework dependency, and the worker and audio worklet are built in, so there is nothing extra to host or configure.
2. **share one instance** through the module below, and reuse it for sequential work. creating it makes no request and asks for no permission; each instance loads its own copy of the model.
3. **prepare at the right moment, with visible progress.** when voice is the page's main purpose, prepare as the page opens. when it is optional, prepare on the first click of a voice control or when the voice panel opens, so visitors who never use it don't download 178 mb. keep the controls that need the model disabled, or replaced by the progress, until it is ready.
4. **pick the call:**
   - `listen()` when text should appear while the person speaks: dictation into a field, captions, voice commands.
   - `record()`, then `clip.transcribe()`, when the text is needed once they finish: voice notes and messages. capture starts even while the model is still downloading, nothing can fall behind, and the audio stays available to play back or keep.
   - `transcribe()` for complete audio the app already has: uploads, urls, blobs, pcm.
   - `realtime()` for live pcm the app produces itself.
5. **run one model call at a time per instance.** `prepare()`, `transcribe()` and a live session each hold the instance until they finish, and an overlapping call throws `BUSY` instead of waiting. await the shared preparation and the previous call first, or create a second instance for work that must run at the same time. when one page has several speech controls, give them one shared busy state (or put them in one component), so the others stay disabled while one runs.
6. **handle errors by `code`**, and treat a denied microphone as an ordinary outcome.
7. **dispose** the instance when its owner goes away. a disposed instance is finished; create a new one.
8. **verify in a real browser**, as described at the end.

### the shared instance

```ts
// src/lib/speech.ts
import { createSpeech, type Progress, type Speech } from "vocule";

let speech: Speech | undefined;
let preparing: Promise<void> | undefined;
const listeners = new Set<(event: Progress) => void>();

/** the app's one instance, created on first use. */
export function getSpeech(): Speech {
  speech ??= createSpeech({
    onProgress: (event) => {
      for (const listener of listeners) listener(event);
    },
  });
  return speech;
}

/** download, verify and warm the model once; every caller shares one attempt. */
export function prepareSpeech(): Promise<void> {
  preparing ??= getSpeech()
    .prepare()
    .catch((error: unknown) => {
      preparing = undefined; // a failed attempt can be retried
      throw error;
    });
  return preparing;
}

export function onSpeechProgress(
  listener: (event: Progress) => void,
): () => void {
  listeners.add(listener);
  return () => {
    listeners.delete(listener);
  };
}

/** release the worker, gpu memory and capture; the next getSpeech() starts over. */
export function disposeSpeech(): void {
  speech?.dispose();
  speech = undefined;
  preparing = undefined;
}
```

always `await prepareSpeech()` before `transcribe()`, `listen()`, `realtime()` or `clip.transcribe()`: a model call made while `prepare()` is still running throws `BUSY`, while awaiting the shared promise costs nothing once the model is ready. `speech.ready` reports whether the model is currently loaded.

### progress

`onProgress` reports `download`, `verify` and `load` while the model prepares, and `features`, `encode` and `decode` during every later transcription. only `download` has a meaningful percentage: `completed` and `total` are bytes. show the other phases as a label.

```ts
// src/lib/speech.ts (continued)

/** a status line for model preparation; undefined for inference phases. */
export function preparationLabel(event: Progress): string | undefined {
  if (event.phase === "download") {
    const percent = event.total
      ? Math.floor((100 * (event.completed ?? 0)) / event.total)
      : 0;
    return `downloading the speech model: ${percent}%`;
  }
  if (event.phase === "verify") return "checking the speech model";
  if (event.phase === "load") return "starting the speech engine";
  return undefined;
}
```

### transcribe complete audio

```ts
// src/lib/transcribe-file.ts
import { getSpeech, prepareSpeech } from "./speech";

export async function transcribeFile(
  file: File,
  signal?: AbortSignal,
): Promise<string> {
  await prepareSpeech();
  const transcript = await getSpeech().transcribe(file, { signal });
  return transcript.text;
}
```

- accepted inputs: `File`, `Blob`, a url string, `URL`, `Request`, `Response`, encoded bytes (`ArrayBuffer`, `Uint8Array`, `DataView`), `AudioBuffer`, webcodecs `AudioData`, a file picker handle, `{ samples, sampleRate }` with float or 16-bit samples, `{ channels, sampleRate }`, a `Float32Array` with `{ sampleRate }`, or a finite stream of pcm, `AudioData` or byte chunks. a `MediaStream` belongs in `listen()` or `record()`.
- video files work when the browser can decode their audio track; chromium decoded mp4, mov and webm video, and mp3, m4a, ogg, webm and flac audio. wav is read sample for sample; other formats go through the browser's decoder and resampling, so their text can differ slightly from the same audio as wav.
- long recordings are split at natural pauses automatically, with no duration cap, but the whole file is held in memory.
- the result is a `Transcript`: `text`, `metrics` (`audioSeconds`, `wallMs`, `inferenceMs`, `realtimeFactor`), `warnings` and `model`. `realtimeFactor` is processing time over audio time, so lower is faster. `warnings` holds standing notes today; log them rather than showing them as errors. `verified` is always `false` today; it is not a success flag.
- silence gives an empty `text`; treat it as "nothing was heard".
- for several files, await them one after another. an `AbortSignal` cancels a call with `ABORTED`; the next call prepares the model again, which is quick once it is cached.

### live microphone text

call `listen()` from a click or tap, once the model is ready: it may show the browser's permission prompt, and browsers may refuse to start audio capture long after the click, for example after a first download.

```ts
// src/lib/dictation.ts
import type { RealtimeSession, RealtimeUpdate } from "vocule";
import { getSpeech, prepareSpeech } from "./speech";

let live: RealtimeSession | undefined;

export async function startDictation(
  render: (update: RealtimeUpdate) => void,
  fail: (error: unknown) => void,
): Promise<void> {
  await prepareSpeech();
  live = await getSpeech().listen({
    onUpdate: (update) => {
      if (update.kind === "final") live = undefined; // the session is over
      render(update);
    },
    onError: fail,
  });
}

export async function stopDictation(): Promise<string> {
  const session = live;
  live = undefined;
  return session ? session.stop() : "";
}
```

- every update carries the whole view. render `update.text`, which is `settledText` followed by `draftText`. the draft is revised as speech continues; settled text and `segments` only grow. `kind` is `"draft"`, `"settled"` or `"final"`.
- `stop()` finishes pending work and resolves to the settled text. `cancel()` discards it.
- while a live session runs, `transcribe()` on the same instance throws `BUSY`.
- pass `{ stream }` to reuse a `MediaStream` the app already has; vocule leaves its tracks running, so the app stops them. `realtime()` takes pcm chunks through `push({ samples, sampleRate })` and asks for no permission.
- drafts are repeated transcriptions of recent audio, not streamed tokens. segment times are capture times from an energy gate, not word timings.
- a `"final"` update means the session is over, whether after `stop()` or because capture ended on its own, for example when the microphone is unplugged. reset the ui there.
- a background failure ends the session through `onError`, and `stop()` rejects with the same error.

### record, play back, transcribe

```ts
// src/lib/recording.ts
import type { RecordingSession } from "vocule";
import { getSpeech, prepareSpeech } from "./speech";

let session: RecordingSession | undefined;

export async function startRecording(): Promise<void> {
  session = await getSpeech().record(); // from a click or tap
}

export async function finishRecording() {
  if (!session) throw new Error("no recording in progress");
  const clip = await session.stop();
  session = undefined;
  await prepareSpeech();
  const transcript = await clip.transcribe();
  return { clip, text: transcript.text };
}
```

`record()` never waits for the model, and it starts preparing it in the background. the session has `pause()`, `resume()`, `cancel()`, `state` and `durationSeconds`. the clip is uncompressed pcm (`samples`, `sampleRate`, `durationSeconds`), with `clip.wav` as a lazily encoded wav `Blob` and `clip.play()`, which resolves to an `HTMLAudioElement`. `clip.transcribe()` uses the instance that recorded it, so do not dispose that instance first.

### errors

every vocule failure is a `SpeechError` with a stable `code`. microphone problems arrive as the browser's own `DOMException`.

```ts
// src/lib/speech-errors.ts
import { SpeechError } from "vocule";

/** a message for the user, or undefined when the app cancelled on purpose. */
export function speechErrorMessage(error: unknown): string | undefined {
  if (error instanceof DOMException && error.name === "NotAllowedError")
    return "microphone access is blocked. allow it in the site settings and try again.";
  if (error instanceof DOMException && error.name === "NotFoundError")
    return "no microphone was found.";
  if (error instanceof DOMException && error.name === "NotReadableError")
    return "the microphone is in use by another app. close it and try again.";
  if (!(error instanceof SpeechError))
    return "something went wrong. please try again.";
  switch (error.code) {
    case "ABORTED":
    case "DISPOSED":
      return undefined;
    case "BACKEND_UNAVAILABLE":
      return "this browser can't run speech recognition on this device. try a recent chrome or safari.";
    case "NETWORK":
      return "the speech model could not be downloaded. check the connection and try again.";
    case "AUDIO_INVALID":
      return "this file could not be read as audio.";
    case "BACKPRESSURE":
      return "speech recognition fell behind. stop and start again.";
    case "BUSY":
      return "speech recognition is still busy. wait for it to finish.";
    default:
      return error.message;
  }
}
```

`GPU_LOST` and `GPU_ERROR` reset the model, so the next call prepares it again. [references/troubleshooting.md](references/troubleshooting.md) lists every code with its cause and fix.

## framework rules

- create the instance and call it only in browser code: event handlers, effects, `onMount` or `onMounted`. importing `vocule` during server rendering has no side effects, but model calls fail there. in the next.js app router, components that use it need `"use client"`.
- keep the instance in the shared module instead of component state. react strict mode mounts twice in development, and a `dispose()` in that cleanup rejects the pending `prepare()` with `ABORTED`.
- cancel live sessions and recordings in the component's cleanup; dispose the shared instance only when the whole speech feature goes away.
- in a browser extension or behind a strict content security policy, read [references/deployment.md](references/deployment.md) first.

[references/frameworks.md](references/frameworks.md) has complete react, next.js, vue, nuxt, svelte, sveltekit, angular, astro and plain typescript examples built on these files.

## content security policy and hosting

skip this section when the site sends no content security policy. otherwise choose one of two setups, both checked in chromium under a real policy header:

| directive | built-in worker (default) | hosted files (`assetBaseURL`) |
| --- | --- | --- |
| `script-src` | `'wasm-unsafe-eval' data:` | `'self' 'wasm-unsafe-eval'` |
| `worker-src` | `blob:` | `'self'` |
| `connect-src` | the model hosts and `data:` | the model hosts and `'self'` |
| `media-src` | `blob:`, for `clip.play()` | `blob:`, for `clip.play()` |

the model hosts are `https://cdn.karanganesan.com`, plus `https://huggingface.co` and `https://*.hf.co` for the fallback, or the origin given as `modelBaseURL`. the built-in worker loads its webassembly from a `data:` url, so without `data:` in `connect-src` preparation fails with `MODEL_FORMAT: Failed to fetch`. for hosted files, copy `node_modules/vocule/dist/` to a same-origin folder and pass `createSpeech({ assetBaseURL: "/vocule/" })`.

[references/deployment.md](references/deployment.md) has complete policies, the copy commands, the model mirror, caching and bundler notes.

## verify

1. run the app on localhost or https and open it in a browser with webgpu, such as chrome or safari 26. `"gpu" in navigator` should be true.
2. prepare once, watch the download reach 100%, then transcribe a short clip and check the text. reload the page: the network panel should show no second model download, and preparation should finish much sooner.
3. for live speech, allow the microphone, speak, stop, and check the final text. deny the permission once and check that the app says so.
4. check the browser console for errors, then repeat in the production build (`vite build` and `vite preview`, `next build` and `next start`, and so on), since bundlers differ.
5. do not quote vocule's published speeds as the app's own; they were measured on a macbook air m4.

## reference files

- [references/api.md](references/api.md): every option, input, result field, error code and entry point.
- [references/frameworks.md](references/frameworks.md): per-framework examples.
- [references/deployment.md](references/deployment.md): content security policy, self-hosted assets, model mirror, caching, browser support.
- [references/troubleshooting.md](references/troubleshooting.md): symptoms and error codes, with causes and fixes.
