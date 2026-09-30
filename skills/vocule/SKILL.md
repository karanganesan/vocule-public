---
name: vocule
description: integrate `@karanganesan/vocule` into browser web apps for file transcription, dictation, recording or deployment. from 2.0.0, model is compulsory with a supported lowercase id; distinguish the published 1.x contract.
---

# vocule

vocule is private speech to text for web apps. a browser inference engine runs a deliberately selected speech to text model on webgpu and webassembly, in a worker off the ui thread. one instance handles complete audio, live speech and recordings, and reuses verified cached model bytes. audio processing stays on the device: there is no speech server, api key or cloud inference fallback. model files download from vocule’s cdn with an immutable hugging face fallback for the same checkpoint.

this review branch describes the **2.0.0 release candidate**. do not apply its new-major calls to a published 1.x install. **`model` is compulsory from 2.0.0**, entirely lowercase: `"fermionresearch/phonon-2"` or `"moondream/parakeet-redux"`, with no sdk default. published 1.x retains its implicit redux choice. missing, undefined, mixed-case or unsupported ids throw `SpeechError("CAPABILITY", ...)` synchronously before side effects. the installed package is the source of truth: if anything here disagrees with `node_modules/@karanganesan/vocule/dist/index.d.ts`, follow the types, which declare every public option, result and error with comments.

the selected checkpoint's original bytes and every reconstructed source weight value stay fixed. engine and cache layout changes preserve those values exactly; compact runtime packing is lossless storage. describe measured speed and quality separately, and keep provider/native reference settings labeled.

## check the fit first

vocule is for browser and web apps only. before writing code, tell the user plainly if the request needs any of these, and do not build workarounds:

- **a server, node, edge function, react native or native app.** every model call needs a browser page with a worker, webgpu and webassembly.
- **timestamps, subtitles or speaker labels.** transcripts are untimed text. `timestamps: "segment"` or `"word"` throws `CAPABILITY`, and `transcript.segments` is always empty.
- **browsers without webgpu.** there is no cpu or webgl fallback. vocule needs webgpu, webassembly and a secure context (https or localhost). 1.0.0 was tested in chrome 153 and safari 26.6 on apple silicon; final 2.0.0 chrome/safari qualification is pending. test the actual model, browser and device before claiming support.
- **choosing a language or translating.** there is no language option. phonon-2 is an english model. redux lists 25 languages upstream; english, spanish and french were tested in 1.0.0. no new language support is implied by the major.
- **a small first load.** the first preparation downloads the selected model’s runtime artifacts; use `getModelDescriptor(id).downloadBytes` from the audited installed release for a transfer hint. checkpoint/container bytes are different from transfer and resident memory. later visits reuse verified extracted or packed cache bytes.

## integrate

1. **check the installed major, then install** with the project's package manager: `npm install @karanganesan/vocule`, `pnpm add @karanganesan/vocule`, `yarn add @karanganesan/vocule` or `bun add @karanganesan/vocule`. this draft's new-major examples require the reviewed 2.0.0 candidate or the matching registry release after the user publishes it; an unversioned install still receives the current public 1.x while publication is held. there is no framework dependency, and the worker and audio worklet are built in, so there is nothing extra to host or configure.
2. **choose deliberately and share one instance** through the module below, and reuse it for sequential work. pass a required `model` before other common options. creating it makes no request and asks for no permission; each instance loads its own copy of the model.
3. **prepare at the right moment, with visible progress.** when voice is the page's main purpose, prepare as the page opens. when it is optional, prepare on the first voice action or when the voice panel opens, so visitors who never use it do not download weights. browsing or changing a model picker must not prepare either model. `transcribe()`, `listen()`, `realtime()` and `record()` also start preparation on demand; a caller can await `prepare()` first when the model must be ready before starting. show download progress while it runs.
4. **pick the call:**
   - `listen()` when text should appear while the person speaks: dictation into a field, captions, voice commands.
   - `record()`, then `clip.transcribe()`, when the text is needed once they finish: voice notes and messages. capture starts even while the model is still downloading, nothing can fall behind, and the audio stays available to play back or keep.
   - `transcribe()` for complete audio the app already has: uploads, urls, blobs, pcm.
   - `realtime()` for live pcm the app produces itself.
5. **run one inference call at a time per instance.** `transcribe()` and a live session exclude other inference calls; an overlapping call fails with `BUSY`. `prepare()` can overlap and callers share one model load. await the previous inference call. concurrent instances duplicate residency; create another only when the app truly requires overlapping inference, never to populate a model picker. when one page has several speech controls, give them one shared busy state (or put them in one component).
6. **handle errors by `code`**, and treat a denied microphone as an ordinary outcome.
7. **dispose** the instance when its owner goes away. a disposed instance is finished; create a new one.
8. **verify in a real browser**, as described at the end.

### the shared instance

```ts
// src/lib/speech.ts
import { createSpeech, type ModelId, type Progress, type Speech } from "@karanganesan/vocule";

let selectedModel: ModelId = "fermionresearch/phonon-2"; // intentional app choice
let speech: Speech | undefined;
let preparing: Promise<void> | undefined;
const listeners = new Set<(event: Progress) => void>();

/** the app's one instance, created on first use. */
export function getSpeech(): Speech {
  if (speech) return speech;
  const current = createSpeech({
    model: selectedModel,
    onProgress: (event) => {
      if (speech !== current) return; // discard late events from a replaced instance
      for (const listener of listeners) listener(event);
    },
  });
  speech = current;
  return current;
}

/** download, verify and warm the model once; every caller shares one attempt. */
export function prepareSpeech(): Promise<void> {
  const current = getSpeech();
  if (current.ready) return Promise.resolve();
  preparing ??= current.prepare().finally(() => {
    if (speech === current) preparing = undefined; // retry or reload after a reset
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

/** call only after the owner has stopped/cancelled active work and preserved wanted text. */
export function selectSpeechModel(model: ModelId): void {
  if (model === selectedModel) return;
  disposeSpeech();
  selectedModel = model; // no creation, download or microphone request
}
```

the helper’s phonon choice is app state, not an sdk default. to retain redux, initialize `selectedModel` to `"moondream/parakeet-redux"`. a fixed-model app can import its factory from `@karanganesan/vocule/phonon-2` or `@karanganesan/vocule/parakeet-redux`; keep the matching required id. the root factory lazily dispatches, while a lean entry excludes the other engine.

for a selector, finish or cancel capture first, preserve only an intentionally awaited final text, release app-owned streams, dispose the old instance, then call `selectSpeechModel()`. fence late updates, errors, progress and pending microphone resolutions with an owner/session generation. leave the next preparation and capture behind an explicit start/resume action. cancel a late returned session; stop late app-owned tracks. never prepare two choices just to browse them. recordings transcribe through their original instance, so finish them before replacing it.

call `prepareSpeech()` when the app wants to warm the model before an action. inference calls also prepare on demand and join an in-progress preparation; they do not fail with `BUSY` because the model is loading. `speech.ready` reports whether the model is currently loaded. keep overlapping inference calls separate.

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
import { getSpeech } from "./speech";

export async function transcribeFile(
  file: File,
  signal?: AbortSignal,
): Promise<string> {
  const current = getSpeech();
  const transcript = await current.transcribe(file, { signal });
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

call `listen()` from a click or tap: it may show the browser's permission prompt. a ready model makes live text prompt. if it is still loading, `listen()` can capture and queue audio during preparation, subject to `maxQueuedSeconds`; avoid delaying the permission prompt until after a first download. if the product prepares before inference, visibly say to wait until listening before speaking. the controller in [references/frameworks.md](references/frameworks.md#shared-dictation-owner) fences late microphone resolutions, errors and text; copy it alongside the shared helper before using this integration:

```ts
// src/dictation.ts
import { createDictation } from "./lib/dictation-controller";
import { speechErrorMessage } from "./lib/speech-errors";

const output = document.querySelector<HTMLElement>("#transcript")!;
const status = document.querySelector<HTMLElement>("#status")!;
const dictation = createDictation({
  render: (text) => { output.textContent = text; },
  state: (value) => { status.textContent = value; },
  fail: (error) => { status.textContent = speechErrorMessage(error) ?? ""; },
});
document.querySelector("#start")!.addEventListener("click", () => { void dictation.start(); });
document.querySelector("#stop")!.addEventListener("click", () => { void dictation.stop(); });
window.addEventListener("pagehide", () => { dictation.cancel(); });
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
import type { RecordingSession } from "@karanganesan/vocule";
import { getSpeech } from "./speech";

export async function startRecording(signal: AbortSignal) {
  const session = await getSpeech().record({ signal }); // from a click or tap
  if (signal.aborted) { await session.cancel(); return undefined; }
  return session;
}

export async function finishRecording(session: RecordingSession, signal: AbortSignal) {
  const clip = await session.stop();
  const transcript = await clip.transcribe({ signal });
  return { clip, text: transcript.text };
}
```

`record()` never waits for the model, and it starts preparing it in the background. the owning view creates an `AbortController`, aborts it and cancels its session at cleanup, and accepts only its current generation's results. the session has `pause()`, `resume()`, `cancel()`, `state` and `durationSeconds`. the clip is uncompressed pcm (`samples`, `sampleRate`, `durationSeconds`), with `clip.wav` as a lazily encoded wav `Blob` and `clip.play()`, which resolves to an `HTMLAudioElement`. `clip.transcribe()` uses the instance that recorded it, so finish before disposing or selecting another model.

### errors

engine failures use `SpeechError` with a stable `code`. microphone problems can also arrive as the browser's own `DOMException`.

```ts
// src/lib/speech-errors.ts
import { SpeechError } from "@karanganesan/vocule";

/** a message for the user, or undefined when the app cancelled on purpose. */
export function speechErrorMessage(error: unknown): string | undefined {
  if (error instanceof DOMException && error.name === "NotAllowedError")
    return "microphone access is blocked. allow it in the site settings and try again.";
  if (error instanceof DOMException && error.name === "NotFoundError")
    return "no microphone was found.";
  if (error instanceof DOMException && (error.name === "NotReadableError" || error.name === "OverconstrainedError"))
    return "the microphone could not be opened. check that it is connected and available, then try again.";
  if (!(error instanceof SpeechError))
    return "something went wrong. please try again.";
  switch (error.code) {
    case "ABORTED":
    case "DISPOSED":
      return undefined;
    case "BACKEND_UNAVAILABLE":
      return "this browser could not start the speech engine. try a supported browser and device.";
    case "NETWORK":
      return "the speech model could not be downloaded. check the connection and try again.";
    case "AUDIO_INVALID":
      return "this file could not be read as audio.";
    case "BACKPRESSURE":
      return "this device could not keep up with live speech. try again.";
    case "BUSY":
      return "speech recognition is still busy. wait for it to finish.";
    default:
      return error.message;
  }
}
```

`GPU_LOST` and `GPU_ERROR` reset the model, so the next call prepares it again. [references/troubleshooting.md](references/troubleshooting.md) lists every code with its cause and fix.

## framework rules

- create the instance and call it only in browser code: event handlers, effects, `onMount` or `onMounted`. importing `@karanganesan/vocule` during server rendering has no side effects, but model calls fail there. in the next.js app router, components that use it need `"use client"`.
- keep the instance in the shared module instead of component state. react strict mode mounts twice in development, and a `dispose()` in that cleanup rejects the pending `prepare()` with `ABORTED`.
- cancel live sessions and recordings in the component's cleanup; dispose the shared instance only when the whole speech feature goes away.
- in a browser extension or behind a strict content security policy, read [references/deployment.md](references/deployment.md) first.

[references/frameworks.md](references/frameworks.md) has complete react, next.js, vue, nuxt, svelte, sveltekit, angular, astro and plain typescript examples built on these files.

## content security policy and hosting

skip this section when the site sends no content security policy. otherwise choose one of two setups. these policies were checked in chromium for 1.0.0 under a real policy header; repeat the checks with each actual 2.0.0 worker before claiming qualification:

| directive | built-in worker (default) | hosted files (`assetBaseURL`) |
| --- | --- | --- |
| `script-src` | `'wasm-unsafe-eval' data:` | `'self' 'wasm-unsafe-eval'` |
| `worker-src` | `blob:` | `'self'` |
| `connect-src` | the model hosts and `data:` | the model hosts and `'self'` |
| `media-src` | `blob:`, for `clip.play()` | `blob:`, for `clip.play()` |

the model hosts are `https://cdn.karanganesan.com`, plus `https://huggingface.co` and `https://*.hf.co` for the fallback, or the origin given as `modelBaseURL`. the built-in worker loads its webassembly from a `data:` url, so without `data:` in `connect-src` preparation fails with `BACKEND_UNAVAILABLE`. for hosted files, copy `node_modules/@karanganesan/vocule/dist/` to a same-origin folder and pass `createSpeech({ model: "fermionresearch/phonon-2", assetBaseURL: "/vocule/" })`. for a lean hosted entry, copy its model directory and point `assetBaseURL` at that directory, whose `worker.js` must match the selected id and package version.

[references/deployment.md](references/deployment.md) has complete policies, the copy commands, the model mirror, caching and bundler notes.

## verify

1. run the app on localhost or https and open it in a browser with webgpu, such as a recent chrome or safari on a supported device. `"gpu" in navigator` should be true; actual preparation still checks the adapter's limits.
2. prepare once, watch the download reach 100%, then transcribe a short clip and check the text. reload the page: the network panel should show no second model download, and preparation should finish much sooner.
3. for live speech, allow the microphone, speak, stop, and check the final text. deny the permission once and check that the app says so.
4. check the browser console for errors, then repeat in the production build (`vite build` and `vite preview`, `next build` and `next start`, and so on), since bundlers differ.
5. test both selected ids where the app offers them, including switch while preparing/listening, rapid selection, cancellation, reload/cache and wrong/stale hosted workers. selecting alone must produce no weights or microphone request.
6. do not quote historical 1.x results as 2.0.0 or phonon results. release figures require final package/worker/model/input hashes, a shared device/browser session, timing boundaries and raw samples; phone layout is not proof of phone inference support.

## reference files

- [references/api.md](references/api.md): every option, input, result field, error code and entry point.
- [references/frameworks.md](references/frameworks.md): per-framework examples.
- [references/deployment.md](references/deployment.md): content security policy, self-hosted assets, model mirror, caching, browser support.
- [references/troubleshooting.md](references/troubleshooting.md): symptoms and error codes, with causes and fixes.
