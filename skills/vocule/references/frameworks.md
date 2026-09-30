# framework examples

draft examples for the **2.0.0 release candidate**. each uses the shared `src/lib/speech.ts` and `src/lib/speech-errors.ts` in [SKILL.md](../SKILL.md). that helper intentionally selects `"fermionresearch/phonon-2"`; to keep redux, initialize its `selectedModel` to `"moondream/parakeet-redux"`. **`model` is compulsory from 2.0.0**, with no sdk default. published 1.x calls keep their old contract.

all nine framework families share these rules:

- call the engine in browser events, effects or mount hooks. imports and the controller factory below do not prepare a model or capture audio.
- use one chosen shared instance. other options and methods are common. a fixed-model app may use the matching lean factory import while retaining its required model id.
- preparation is a product decision: prewarm the selected model when speech is the page's purpose, or start loading on an explicit optional speech action. never prepare both choices or download while browsing a picker.
- run one inference call at a time. several controls must share a busy state; recording capture may overlap, but `clip.transcribe()` is inference.
- cancel capture and pending microphone requests when the owning view goes away. dispose the shared instance only when the entire feature goes away; react strict-mode view cleanup should cancel its session, not permanently dispose the singleton.
- keep progress, denial and backpressure visible. `listen()` can queue audio while preparation runs, bounded by `maxQueuedSeconds`; a first large download can exceed that queue. prewarm before listening when that suits the product, or explain the waiting boundary.

## contents

- [shared dictation owner](#shared-dictation-owner)
- [plain typescript](#plain-typescript)
- [react](#react)
- [next.js](#nextjs)
- [vue](#vue)
- [nuxt](#nuxt)
- [svelte](#svelte)
- [sveltekit](#sveltekit)
- [angular](#angular)
- [astro](#astro)
- [file transcription and recording](#file-transcription-and-recording)
- [adding a model selector](#adding-a-model-selector)

## shared dictation owner

the examples reuse this small browser-action controller so a late permission/session resolution cannot revive an unmounted view. it accepts only its current generation's updates and treats a natural `"final"` update as the end of capture.

```ts
// src/lib/dictation-controller.ts
import type { RealtimeSession } from "@karanganesan/vocule";
import { getSpeech } from "./speech";

export type DictationState = "idle" | "starting" | "listening" | "stopping";
export function createDictation(callbacks: {
  render(text: string): void;
  state(value: DictationState): void;
  fail(error: unknown): void;
}) {
  let generation = 0;
  let session: RealtimeSession | undefined;
  let controller: AbortController | undefined;
  let starting = false;
  let finishing: Promise<string> | undefined;
  let settled = "";

  function cancel(): void {
    ++generation;
    controller?.abort();
    controller = undefined;
    const old = session;
    session = undefined;
    starting = false;
    finishing = undefined;
    void old?.cancel().catch(() => {});
  }

  async function start(): Promise<void> {
    if (session || starting || finishing) return;
    const token = ++generation;
    controller = new AbortController();
    const signal = controller.signal;
    starting = true;
    settled = "";
    callbacks.state("starting");
    let ended = false;
    try {
      const next = await getSpeech().listen({
        signal,
        onUpdate: (update) => {
          if (token !== generation) return;
          settled = update.settledText;
          callbacks.render(update.text);
          if (update.kind === "final") {
            ended = true;
            session = undefined;
            starting = false;
            callbacks.state("idle");
          }
        },
        onError: (error) => {
          if (token !== generation) return;
          ended = true;
          session = undefined;
          starting = false;
          callbacks.state("idle");
          callbacks.fail(error);
        },
      });
      if (token !== generation || ended) {
        await next.cancel().catch(() => {});
        return;
      }
      session = next;
      starting = false;
      callbacks.state("listening");
    } catch (error) {
      if (token !== generation) return;
      starting = false;
      callbacks.state("idle");
      callbacks.fail(error);
    }
  }

  function stop(): Promise<string> {
    if (finishing) return finishing;
    const old = session;
    if (!old) {
      cancel(); // includes a pending microphone request
      callbacks.state("idle");
      return Promise.resolve(settled);
    }
    const token = ++generation;
    const previous = settled;
    // Retain old so cancel() can release it even while its final is pending.
    finishing = Promise.resolve().then(() => old.stop()).then((text) => {
      if (token === generation) {
        settled = text;
        callbacks.render(text);
        callbacks.state("idle");
      }
      return text;
    }, (error: unknown) => {
      if (token === generation) {
        callbacks.state("idle");
        callbacks.fail(error);
      }
      return previous;
    }).finally(() => {
      if (token !== generation) return;
      session = undefined;
      finishing = undefined;
    });
    callbacks.state("stopping");
    return finishing;
  }

  return { start, stop, cancel, get text() { return settled; } };
}
```

subscribe to `onSpeechProgress()` from the shared helper to show preparation phases/byte progress; `preparationLabel()` formats them. unsubscribe with the view's cleanup. keep listeners guarded after unmount, just like the callbacks below.

## plain typescript

works with vite, bun or another npm-aware browser bundler. selecting a file or pressing the microphone button is the loading boundary.

```html
<input id="file" type="file" accept="audio/*,video/*" />
<button id="mic" type="button">start dictation</button>
<p id="status" role="status"></p>
<p id="text"></p>
<script type="module" src="/src/main.ts"></script>
```

```ts
// src/main.ts
import { createDictation, type DictationState } from "./lib/dictation-controller";
import { getSpeech, onSpeechProgress, preparationLabel } from "./lib/speech";
import { speechErrorMessage } from "./lib/speech-errors";

const mic = document.querySelector<HTMLButtonElement>("#mic")!;
const file = document.querySelector<HTMLInputElement>("#file")!;
const status = document.querySelector<HTMLElement>("#status")!;
const output = document.querySelector<HTMLElement>("#text")!;
let state: DictationState = "idle";
let fileAbort: AbortController | undefined;
const owner = createDictation({
  render: (text) => { output.textContent = text; },
  state: (next) => {
    state = next;
    mic.disabled = next === "starting" || next === "stopping";
    mic.textContent = next === "listening" ? "stop" : "start dictation";
    file.disabled = next !== "idle";
    status.textContent = next;
  },
  fail: (error) => { status.textContent = speechErrorMessage(error) ?? ""; },
});
const unsubscribe = onSpeechProgress((event) => {
  const label = preparationLabel(event);
  if (label) status.textContent = label;
});
mic.addEventListener("click", () => {
  void (state === "listening" ? owner.stop() : owner.start());
});
file.addEventListener("change", async () => {
  const selected = file.files?.[0];
  file.value = "";
  if (!selected) return;
  fileAbort = new AbortController();
  const signal = fileAbort.signal;
  mic.disabled = file.disabled = true;
  try { output.textContent = (await getSpeech().transcribe(selected, { signal })).text; }
  catch (error) { if (!signal.aborted) status.textContent = speechErrorMessage(error) ?? ""; }
  finally { if (!signal.aborted) mic.disabled = file.disabled = false; }
});
window.addEventListener("pagehide", () => { unsubscribe(); fileAbort?.abort(); owner.cancel(); });
```

## react

the controller is local to the view; the selected speech instance stays in the shared module. strict-mode cleanup cancels the controller without permanently disposing that instance.

```tsx
// src/components/dictation.tsx
import { useEffect, useRef, useState } from "react";
import { createDictation, type DictationState } from "../lib/dictation-controller";
import { onSpeechProgress, preparationLabel } from "../lib/speech";
import { speechErrorMessage } from "../lib/speech-errors";

export function Dictation() {
  const [state, setState] = useState<DictationState>("idle");
  const [text, setText] = useState("");
  const [status, setStatus] = useState("");
  const [error, setError] = useState<string>();
  const mounted = useRef(false);
  const [owner] = useState(() => createDictation({
    render: (value) => { if (mounted.current) setText(value); },
    state: (value) => {
      if (!mounted.current) return;
      setState(value);
      if (value !== "starting") setStatus("");
    },
    fail: (reason) => { if (mounted.current) setError(speechErrorMessage(reason)); },
  }));
  useEffect(() => {
    mounted.current = true;
    const unsubscribe = onSpeechProgress((event) => {
      const label = preparationLabel(event);
      if (mounted.current && label) setStatus(label);
    });
    return () => { mounted.current = false; unsubscribe(); owner.cancel(); };
  }, [owner]);
  return <section>
    <button type="button" disabled={state === "starting" || state === "stopping"}
      onClick={() => { setError(undefined); void (state === "listening" ? owner.stop() : owner.start()); }}>
      {state === "listening" ? "stop" : "start dictation"}
    </button>
    <p role="status">{status || state}</p>
    {error && <p role="alert">{error}</p>}
    <p>{text}</p>
  </section>;
}
```

## next.js

use the react component above with `"use client"` as its first line. its hook/helper imports join the client bundle. render it from a server page without calling the speech helpers there:

```tsx
// src/app/dictate/page.tsx
import { Dictation } from "../../components/dictation";
export default function Page() {
  return <main><h1>dictate</h1><Dictation /></main>;
}
```

keep engine calls out of server components, route handlers, server actions and middleware. the pages router uses the same component without the directive. check the production build in a browser.

## vue

```vue
<script setup lang="ts">
import { onBeforeUnmount, ref } from "vue";
import { createDictation, type DictationState } from "../lib/dictation-controller";
import { speechErrorMessage } from "../lib/speech-errors";
const state = ref<DictationState>("idle");
const text = ref("");
const error = ref<string>();
let active = true;
const owner = createDictation({
  render: (value) => { if (active) text.value = value; },
  state: (value) => { if (active) state.value = value; },
  fail: (reason) => { if (active) error.value = speechErrorMessage(reason); },
});
onBeforeUnmount(() => { active = false; owner.cancel(); });
function toggle() {
  error.value = undefined;
  void (state.value === "listening" ? owner.stop() : owner.start());
}
</script>
<template>
  <button type="button" :disabled="state === 'starting' || state === 'stopping'" @click="toggle">
    {{ state === "listening" ? "stop" : "start dictation" }}
  </button>
  <p role="status">{{ state }}</p>
  <p v-if="error" role="alert">{{ error }}</p>
  <p>{{ text }}</p>
</template>
```

the setup creates only a controller; engine calls happen in the click handler. add a mounted progress subscription and remove it in `onBeforeUnmount` when preparation status is needed.

## nuxt

use the vue component above and adjust helper imports to `~/lib/`. no engine call occurs during setup or server rendering. use `Dictation.client.vue` or `<ClientOnly>` when skipping the server's idle markup is useful. do not call the helpers in `server/` routes or `useAsyncData`.

## svelte

svelte 5 with runes:

```svelte
<script lang="ts">
  import { onMount } from "svelte";
  import { createDictation, type DictationState } from "./dictation-controller";
  import { speechErrorMessage } from "./speech-errors";
  let status = $state<DictationState>("idle");
  let text = $state("");
  let error = $state<string>();
  let active = false;
  const owner = createDictation({
    render: (value) => { if (active) text = value; },
    state: (value) => { if (active) status = value; },
    fail: (reason) => { if (active) error = speechErrorMessage(reason); },
  });
  onMount(() => { active = true; return () => { active = false; owner.cancel(); }; });
  function toggle() {
    error = undefined;
    void (status === "listening" ? owner.stop() : owner.start());
  }
</script>
<button type="button" disabled={status === "starting" || status === "stopping"} onclick={toggle}>
  {status === "listening" ? "stop" : "start dictation"}
</button>
<p role="status">{status}</p>
{#if error}<p role="alert">{error}</p>{/if}
<p>{text}</p>
```

## sveltekit

put the svelte component in `src/lib/` and import it with `$lib/Dictation.svelte`. calls stay in events and `onMount`; server rendering can remain enabled. for other browser-only code check `browser` from `$app/environment`. keep engine calls out of server `load`, `+page.server.ts` and `+server.ts`.

## angular

a standalone component with signals. the controller makes no model call until the browser click:

```ts
import { Component, type OnDestroy, signal } from "@angular/core";
import { createDictation, type DictationState } from "../lib/dictation-controller";
import { speechErrorMessage } from "../lib/speech-errors";

@Component({
  selector: "app-dictation",
  standalone: true,
  template: `
    <button type="button" [disabled]="state() === 'starting' || state() === 'stopping'" (click)="toggle()">
      {{ state() === 'listening' ? 'stop' : 'start dictation' }}
    </button>
    <p role="status">{{ state() }}</p>
    @if (error()) { <p role="alert">{{ error() }}</p> }
    <p>{{ text() }}</p>
  `,
})
export class DictationComponent implements OnDestroy {
  protected readonly state = signal<DictationState>("idle");
  protected readonly text = signal("");
  protected readonly error = signal<string | undefined>(undefined);
  private active = true;
  private readonly owner = createDictation({
    render: (value) => { if (this.active) this.text.set(value); },
    state: (value) => { if (this.active) this.state.set(value); },
    fail: (reason) => { if (this.active) this.error.set(speechErrorMessage(reason)); },
  });
  protected toggle(): void {
    this.error.set(undefined);
    void (this.state() === "listening" ? this.owner.stop() : this.owner.start());
  }
  ngOnDestroy(): void { this.active = false; this.owner.cancel(); }
}
```

signals update from the callbacks with or without zone.js. use a browser-only render/mount hook for optional prewarming or progress subscriptions; keep them out of server initialization.

## astro

use a client-only island for the framework component:

```astro
---
import { Dictation } from "../components/dictation";
---
<main><h1>dictate</h1><Dictation client:only="react" /></main>
```

this needs `@astrojs/react`; use `client:only="vue"` or `client:only="svelte"` for those frameworks. without a framework, put the plain typescript integration in an astro `<script>`; astro bundles it for the browser.

## file transcription and recording

the shared helper already supplies the required model. use these functions behind events in any framework and cancel with the owning view:

```ts
import { getSpeech } from "./speech";
import type { RecordingSession } from "@karanganesan/vocule";

export async function transcribeFile(file: File, signal: AbortSignal) {
  return (await getSpeech().transcribe(file, { signal })).text;
}
export async function startRecording(signal: AbortSignal): Promise<RecordingSession> {
  return getSpeech().record({ signal });
}
export async function finishRecording(session: RecordingSession, signal: AbortSignal) {
  const clip = await session.stop();
  const transcript = await clip.transcribe({ signal });
  return { clip, text: transcript.text }; // clip.play(), clip.wav remain available
}
```

recording starts capture without waiting for the model and begins preparation in the background. keep that instance alive for `clip.transcribe()`. a cancelled or superseded `record()` resolution must be cancelled immediately; pass an owner `AbortSignal` and guard its callbacks/results. let `clip.play()` run only from the user's playback action.

## adding a model selector

all framework examples inherit one intentional app choice from the shared helper; none prepares two models. metadata-only `getModelDescriptor(id)` and `getModelCacheStatus(id)` can populate a picker without downloads or microphone requests.

on a new choice, stop/finalize or cancel the current controller, preserve settled text and only the intentionally awaited final, abort pending file/record work, stop any app-owned stream, and call `selectSpeechModel(id)` from the helper. fence session/model generations, stale progress and late microphone results. serialize finalization/disposal before a new preparation so rapid choices keep the latest id without two full engines. leave the next `start()` behind an explicit click or tap; switching alone must not prepare or listen. an old recording's transcript action still belongs to its old instance, so finish it before disposal.
