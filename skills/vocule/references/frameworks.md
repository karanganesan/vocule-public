# framework examples

every example builds on two files from SKILL.md: `src/lib/speech.ts` (the shared instance, `prepareSpeech()`, `onSpeechProgress()` and `preparationLabel()`) and `src/lib/speech-errors.ts` (`speechErrorMessage()`). adjust the import paths to where the project keeps them. the rules are the same in every framework:

- create and call vocule only in the browser: event handlers, effects and mount hooks.
- prepare before an action when the model should already be ready; inference calls also prepare on demand. start one inference call at a time per instance.
- start `listen()` and `record()` from a click or tap. these examples prewarm the model before enabling dictation so live text can appear promptly.
- treat an update with `kind: "final"` as the end of a live session: it also arrives when capture ends on its own.
- cancel live sessions and recordings when their component unmounts.
- when one page shows several speech controls, share one busy state between them (or combine them into one component), so the others stay disabled while one runs; otherwise the second call throws `BUSY`.

## contents

- [plain typescript](#plain-typescript)
- [react](#react)
- [next.js](#nextjs)
- [vue](#vue)
- [nuxt](#nuxt)
- [svelte](#svelte)
- [sveltekit](#sveltekit)
- [angular](#angular)
- [astro](#astro)

## plain typescript

works with vite, bun or any bundler that understands npm packages.

```html
<!-- index.html -->
<input id="file" type="file" accept="audio/*,video/*" disabled />
<button id="mic" type="button" disabled>start dictation</button>
<p id="status">preparing speech recognition</p>
<p id="text"></p>
<script type="module" src="/src/main.ts"></script>
```

```ts
// src/main.ts
import type { RealtimeSession } from "@karanganesan/vocule";
import {
  getSpeech,
  onSpeechProgress,
  preparationLabel,
  prepareSpeech,
} from "./lib/speech";
import { speechErrorMessage } from "./lib/speech-errors";

const fileInput = document.querySelector<HTMLInputElement>("#file")!;
const micButton = document.querySelector<HTMLButtonElement>("#mic")!;
const status = document.querySelector<HTMLElement>("#status")!;
const output = document.querySelector<HTMLElement>("#text")!;
let live: RealtimeSession | undefined;

function setBusy(busy: boolean) {
  fileInput.disabled = busy;
  micButton.disabled = busy;
}

function report(error: unknown) {
  status.textContent = speechErrorMessage(error) ?? "";
}

onSpeechProgress((event) => {
  const label = preparationLabel(event);
  if (label) status.textContent = label;
});

// enable the controls either way: a failed preparation is retried by the next action
prepareSpeech()
  .then(() => {
    status.textContent = "ready";
  }, report)
  .finally(() => setBusy(false));

fileInput.addEventListener("change", async () => {
  const file = fileInput.files?.[0];
  if (!file) return;
  setBusy(true);
  status.textContent = "transcribing";
  try {
    await prepareSpeech();
    output.textContent = (await getSpeech().transcribe(file)).text;
    status.textContent = "done";
  } catch (error) {
    report(error);
  } finally {
    setBusy(false);
  }
});

micButton.addEventListener("click", async () => {
  setBusy(true);
  try {
    if (live) {
      const session = live;
      live = undefined;
      output.textContent = await session.stop();
      micButton.textContent = "start dictation";
      status.textContent = "done";
    } else {
      await prepareSpeech();
      live = await getSpeech().listen({
        onUpdate: (update) => {
          output.textContent = update.text;
          if (update.kind === "final") {
            live = undefined; // the session is over
            micButton.textContent = "start dictation";
            fileInput.disabled = false;
          }
        },
        onError: (error) => {
          live = undefined;
          micButton.textContent = "start dictation";
          fileInput.disabled = false;
          report(error);
        },
      });
      micButton.textContent = "stop";
      status.textContent = "listening";
    }
  } catch (error) {
    report(error);
  } finally {
    setBusy(false);
    fileInput.disabled = live !== undefined; // no file transcription while listening
  }
});
```

## react

one hook prepares the shared model and reports progress; components use it and keep their own sessions. none of this needs a provider or context. react strict mode is safe because the instance lives in the module, not in component state.

the hook starts preparing when its component mounts. for an optional voice feature, mount that component when the user opens it, for example behind a "voice" button, so other visitors don't download the model. each component below tracks only its own work: to show two of them on one page, lift their busy state into a shared place or combine them, as described above.

```ts
// src/hooks/use-speech-model.ts
import { useEffect, useState } from "react";
import {
  onSpeechProgress,
  preparationLabel,
  prepareSpeech,
} from "../lib/speech";
import { speechErrorMessage } from "../lib/speech-errors";

/** prepares the shared model while a component that needs it is mounted. */
export function useSpeechModel() {
  const [ready, setReady] = useState(false);
  const [status, setStatus] = useState("preparing speech recognition");
  const [error, setError] = useState<string>();
  const [attempt, setAttempt] = useState(0);

  useEffect(() => {
    let active = true;
    const unsubscribe = onSpeechProgress((event) => {
      const label = preparationLabel(event);
      if (active && label) setStatus(label);
    });
    prepareSpeech().then(
      () => {
        if (active) setReady(true);
      },
      (reason: unknown) => {
        if (active)
          setError(speechErrorMessage(reason) ?? "speech recognition stopped.");
      },
    );
    return () => {
      active = false;
      unsubscribe();
    };
  }, [attempt]);

  function retry() {
    setError(undefined);
    setAttempt((count) => count + 1);
  }

  return { ready, status, error, retry };
}
```

### transcribe a file

```tsx
// src/components/file-transcriber.tsx
import { type ChangeEvent, useEffect, useRef, useState } from "react";
import { useSpeechModel } from "../hooks/use-speech-model";
import { getSpeech, prepareSpeech } from "../lib/speech";
import { speechErrorMessage } from "../lib/speech-errors";

export function FileTranscriber() {
  const model = useSpeechModel();
  const controller = useRef<AbortController | null>(null);
  const [busy, setBusy] = useState(false);
  const [text, setText] = useState("");
  const [error, setError] = useState<string>();

  // an unfinished transcription would keep the shared instance busy
  useEffect(() => () => controller.current?.abort(), []);

  async function transcribe(event: ChangeEvent<HTMLInputElement>) {
    const file = event.target.files?.[0];
    event.target.value = ""; // lets the same file be chosen again
    if (!file) return;
    const abort = new AbortController();
    controller.current = abort;
    setBusy(true);
    setError(undefined);
    try {
      await prepareSpeech();
      const transcript = await getSpeech().transcribe(file, {
        signal: abort.signal,
      });
      setText(transcript.text);
    } catch (reason) {
      setError(speechErrorMessage(reason));
    } finally {
      setBusy(false);
    }
  }

  if (model.error)
    return (
      <p role="alert">
        {model.error}{" "}
        <button type="button" onClick={model.retry}>
          try again
        </button>
      </p>
    );
  if (!model.ready) return <p>{model.status}</p>;
  return (
    <section>
      <input
        type="file"
        accept="audio/*,video/*"
        disabled={busy}
        onChange={transcribe}
      />
      {busy && <p>transcribing</p>}
      {error && <p role="alert">{error}</p>}
      <p>{text}</p>
    </section>
  );
}
```

### live dictation

```tsx
// src/components/dictation.tsx
import { useEffect, useRef, useState } from "react";
import type { RealtimeSession } from "@karanganesan/vocule";
import { useSpeechModel } from "../hooks/use-speech-model";
import { getSpeech, prepareSpeech } from "../lib/speech";
import { speechErrorMessage } from "../lib/speech-errors";

type State = "idle" | "starting" | "listening" | "stopping";

export function Dictation() {
  const model = useSpeechModel();
  const session = useRef<RealtimeSession | null>(null);
  const [state, setState] = useState<State>("idle");
  const [text, setText] = useState("");
  const [error, setError] = useState<string>();

  // releases the microphone and the shared instance on unmount
  useEffect(() => () => void session.current?.cancel(), []);

  async function start() {
    setState("starting");
    setError(undefined);
    setText("");
    try {
      await prepareSpeech();
      session.current = await getSpeech().listen({
        onUpdate: (update) => {
          setText(update.text);
          if (update.kind === "final") {
            session.current = null; // the session is over
            setState("idle");
          }
        },
        onError: (reason) => {
          session.current = null;
          setState("idle");
          setError(speechErrorMessage(reason));
        },
      });
      setState("listening");
    } catch (reason) {
      setState("idle");
      setError(speechErrorMessage(reason));
    }
  }

  async function stop() {
    const current = session.current;
    session.current = null;
    if (!current) return;
    setState("stopping");
    try {
      setText(await current.stop());
    } catch (reason) {
      setError(speechErrorMessage(reason));
    } finally {
      setState("idle");
    }
  }

  if (model.error)
    return (
      <p role="alert">
        {model.error}{" "}
        <button type="button" onClick={model.retry}>
          try again
        </button>
      </p>
    );
  if (!model.ready) return <p>{model.status}</p>;
  const listening = state === "listening";
  return (
    <section>
      <button
        type="button"
        disabled={state === "starting" || state === "stopping"}
        onClick={listening ? stop : start}
      >
        {listening ? "stop" : "start dictation"}
      </button>
      {error && <p role="alert">{error}</p>}
      <p>{text}</p>
    </section>
  );
}
```

### record, play back and transcribe

recording does not need the model, so the record button works while the model is still downloading; transcription waits for it.

```tsx
// src/components/recorder.tsx
import { useEffect, useRef, useState } from "react";
import type { Recording, RecordingSession } from "@karanganesan/vocule";
import { getSpeech, prepareSpeech } from "../lib/speech";
import { speechErrorMessage } from "../lib/speech-errors";

export function Recorder() {
  const session = useRef<RecordingSession | null>(null);
  const [recording, setRecording] = useState(false);
  const [busy, setBusy] = useState(false);
  const [clip, setClip] = useState<Recording>();
  const [text, setText] = useState("");
  const [error, setError] = useState<string>();

  useEffect(() => () => void session.current?.cancel(), []);

  async function start() {
    setError(undefined);
    setText("");
    try {
      session.current = await getSpeech().record();
      setRecording(true);
    } catch (reason) {
      setError(speechErrorMessage(reason));
    }
  }

  async function stop() {
    const current = session.current;
    session.current = null;
    setRecording(false);
    if (!current) return;
    setBusy(true);
    try {
      const finished = await current.stop();
      setClip(finished);
      await prepareSpeech();
      setText((await finished.transcribe()).text);
    } catch (reason) {
      setError(speechErrorMessage(reason));
    } finally {
      setBusy(false);
    }
  }

  return (
    <section>
      <button
        type="button"
        disabled={busy}
        onClick={recording ? stop : start}
      >
        {recording ? "stop" : "record"}
      </button>
      {clip && (
        <button type="button" onClick={() => void clip.play()}>
          play {clip.durationSeconds.toFixed(1)} s
        </button>
      )}
      {busy && <p>transcribing</p>}
      {error && <p role="alert">{error}</p>}
      <p>{text}</p>
    </section>
  );
}
```

`clip.wav` is a wav `Blob` for downloads or uploads the user asks for.

## next.js

use the react components above as client components. add `"use client"` as the first line of each component file (`file-transcriber.tsx`, `dictation.tsx`, `recorder.tsx`); the hook and the `lib` files are pulled into the client bundle through them and need no directive. render them from any page:

```tsx
// src/app/dictate/page.tsx
import { Dictation } from "../../components/dictation";

export default function Page() {
  return (
    <main>
      <h1>dictate</h1>
      <Dictation />
    </main>
  );
}
```

- during server rendering the components output their initial "preparing" state; vocule itself runs only after hydration, in effects and handlers.
- never call `getSpeech()` or `prepareSpeech()` in a server component, route handler, server action or middleware.
- in the pages router the same components work without the directive.
- run `next build` and `next start` and test in the browser before shipping.

## vue

```vue
<!-- src/components/Dictation.vue -->
<script setup lang="ts">
import { onBeforeUnmount, onMounted, ref } from "vue";
import type { RealtimeSession } from "@karanganesan/vocule";
import {
  getSpeech,
  onSpeechProgress,
  preparationLabel,
  prepareSpeech,
} from "../lib/speech";
import { speechErrorMessage } from "../lib/speech-errors";

const ready = ref(false);
const status = ref("preparing speech recognition");
const failure = ref<string>();
const state = ref<"idle" | "starting" | "listening" | "stopping">("idle");
const text = ref("");
const error = ref<string>();
let session: RealtimeSession | undefined;
let unsubscribe: (() => void) | undefined;

function prepare() {
  failure.value = undefined;
  prepareSpeech().then(
    () => {
      ready.value = true;
    },
    (reason: unknown) => {
      failure.value = speechErrorMessage(reason) ?? "speech recognition stopped.";
    },
  );
}

onMounted(() => {
  unsubscribe = onSpeechProgress((event) => {
    status.value = preparationLabel(event) ?? status.value;
  });
  prepare();
});

onBeforeUnmount(() => {
  unsubscribe?.();
  void session?.cancel();
});

async function start() {
  state.value = "starting";
  error.value = undefined;
  text.value = "";
  try {
    await prepareSpeech();
    session = await getSpeech().listen({
      onUpdate: (update) => {
        text.value = update.text;
        if (update.kind === "final") {
          session = undefined; // the session is over
          state.value = "idle";
        }
      },
      onError: (reason) => {
        session = undefined;
        state.value = "idle";
        error.value = speechErrorMessage(reason);
      },
    });
    state.value = "listening";
  } catch (reason) {
    state.value = "idle";
    error.value = speechErrorMessage(reason);
  }
}

async function stop() {
  const current = session;
  session = undefined;
  if (!current) return;
  state.value = "stopping";
  try {
    text.value = await current.stop();
  } catch (reason) {
    error.value = speechErrorMessage(reason);
  } finally {
    state.value = "idle";
  }
}
</script>

<template>
  <p v-if="failure" role="alert">
    {{ failure }} <button type="button" @click="prepare">try again</button>
  </p>
  <p v-else-if="!ready">{{ status }}</p>
  <section v-else>
    <button
      type="button"
      :disabled="state === 'starting' || state === 'stopping'"
      @click="state === 'listening' ? stop() : start()"
    >
      {{ state === "listening" ? "stop" : "start dictation" }}
    </button>
    <p v-if="error" role="alert">{{ error }}</p>
    <p>{{ text }}</p>
  </section>
</template>
```

file transcription follows the same shape: `await prepareSpeech()`, then `getSpeech().transcribe(file)` in the input's change handler.

## nuxt

use the vue component above. `onMounted` never runs on the server, so it is already safe; to skip server rendering of the "preparing" state as well, name the file `Dictation.client.vue` or wrap it in `<ClientOnly>`. keep the two `lib` files anywhere under the source directory and import them with `~/`, for example `~/lib/speech`. do not call them from `server/` routes or `useAsyncData`.

## svelte

svelte 5, with runes:

```svelte
<!-- src/lib/Dictation.svelte -->
<script lang="ts">
  import { onMount } from "svelte";
  import type { RealtimeSession } from "@karanganesan/vocule";
  import {
    getSpeech,
    onSpeechProgress,
    preparationLabel,
    prepareSpeech,
  } from "./speech";
  import { speechErrorMessage } from "./speech-errors";

  let ready = $state(false);
  let status = $state("preparing speech recognition");
  let failure = $state<string>();
  let listening = $state(false);
  let busy = $state(false);
  let text = $state("");
  let error = $state<string>();
  let session: RealtimeSession | undefined;

  function prepare() {
    failure = undefined;
    prepareSpeech().then(
      () => {
        ready = true;
      },
      (reason: unknown) => {
        failure = speechErrorMessage(reason) ?? "speech recognition stopped.";
      },
    );
  }

  onMount(() => {
    const unsubscribe = onSpeechProgress((event) => {
      status = preparationLabel(event) ?? status;
    });
    prepare();
    return () => {
      unsubscribe();
      void session?.cancel();
    };
  });

  async function toggle() {
    busy = true;
    error = undefined;
    try {
      if (session) {
        const current = session;
        session = undefined;
        listening = false;
        text = await current.stop();
      } else {
        text = "";
        await prepareSpeech();
        session = await getSpeech().listen({
          onUpdate: (update) => {
            text = update.text;
            if (update.kind === "final") {
              session = undefined; // the session is over
              listening = false;
            }
          },
          onError: (reason) => {
            session = undefined;
            listening = false;
            error = speechErrorMessage(reason);
          },
        });
        listening = true;
      }
    } catch (reason) {
      error = speechErrorMessage(reason);
    } finally {
      busy = false;
    }
  }
</script>

{#if failure}
  <p role="alert">{failure} <button type="button" onclick={prepare}>try again</button></p>
{:else if !ready}
  <p>{status}</p>
{:else}
  <button type="button" disabled={busy} onclick={toggle}>
    {listening ? "stop" : "start dictation"}
  </button>
  {#if error}<p role="alert">{error}</p>{/if}
  <p>{text}</p>
{/if}
```

## sveltekit

use the svelte component above from `src/lib/`, and import it in a page as `$lib/Dictation.svelte`. `onMount` runs only in the browser, so server rendering is safe without `export const ssr = false`. for other browser-only code, check `browser` from `$app/environment`. never call vocule from `+page.server.ts`, `+server.ts` or a `load` function that runs on the server.

## angular

a standalone component with signals, for angular 17 or later. `afterNextRender` runs only in the browser, so it is safe with server-side rendering.

```ts
// src/app/dictation.component.ts
import {
  Component,
  type OnDestroy,
  afterNextRender,
  signal,
} from "@angular/core";
import type { RealtimeSession } from "@karanganesan/vocule";
import {
  getSpeech,
  onSpeechProgress,
  preparationLabel,
  prepareSpeech,
} from "../lib/speech";
import { speechErrorMessage } from "../lib/speech-errors";

@Component({
  selector: "app-dictation",
  standalone: true,
  template: `
    @if (!ready()) {
      <p>{{ failure() ?? status() }}</p>
    } @else {
      <button type="button" [disabled]="busy()" (click)="toggle()">
        {{ listening() ? "stop" : "start dictation" }}
      </button>
      @if (error()) {
        <p role="alert">{{ error() }}</p>
      }
      <p>{{ text() }}</p>
    }
  `,
})
export class DictationComponent implements OnDestroy {
  protected readonly ready = signal(false);
  protected readonly status = signal("preparing speech recognition");
  protected readonly failure = signal<string | undefined>(undefined);
  protected readonly listening = signal(false);
  protected readonly busy = signal(false);
  protected readonly text = signal("");
  protected readonly error = signal<string | undefined>(undefined);
  private session?: RealtimeSession;
  private unsubscribe?: () => void;

  constructor() {
    afterNextRender(() => {
      this.unsubscribe = onSpeechProgress((event) => {
        const label = preparationLabel(event);
        if (label) this.status.set(label);
      });
      prepareSpeech().then(
        () => this.ready.set(true),
        (reason: unknown) =>
          this.failure.set(
            speechErrorMessage(reason) ?? "speech recognition stopped.",
          ),
      );
    });
  }

  protected async toggle(): Promise<void> {
    this.busy.set(true);
    this.error.set(undefined);
    try {
      if (this.session) {
        const current = this.session;
        this.session = undefined;
        this.listening.set(false);
        this.text.set(await current.stop());
      } else {
        this.text.set("");
        await prepareSpeech();
        this.session = await getSpeech().listen({
          onUpdate: (update) => {
            this.text.set(update.text);
            if (update.kind === "final") {
              this.session = undefined; // the session is over
              this.listening.set(false);
            }
          },
          onError: (reason) => {
            this.session = undefined;
            this.listening.set(false);
            this.error.set(speechErrorMessage(reason));
          },
        });
        this.listening.set(true);
      }
    } catch (reason) {
      this.error.set(speechErrorMessage(reason));
    } finally {
      this.busy.set(false);
    }
  }

  ngOnDestroy(): void {
    this.unsubscribe?.();
    void this.session?.cancel();
  }
}
```

signals update the view from vocule's callbacks with or without zone.js.

## astro

render a framework component as a client-only island, so it never runs during the build or on the server:

```astro
---
// src/pages/dictate.astro
import { Dictation } from "../components/dictation";
---

<main>
  <h1>dictate</h1>
  <Dictation client:only="react" />
</main>
```

the react component above needs the `@astrojs/react` integration; use `client:only="vue"` or `client:only="svelte"` for the others. without a framework, put the plain typescript example in a `<script>` tag of an `.astro` page: astro bundles it and runs it in the browser.
