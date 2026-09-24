# deployment

## contents

- [content security policy](#content-security-policy)
- [hosting vocule's files on your origin](#hosting-vocules-files-on-your-origin)
- [serving the model yourself](#serving-the-model-yourself)
- [caching](#caching)
- [bundlers, server rendering and tests](#bundlers-server-rendering-and-tests)
- [browser support](#browser-support)
- [iframes and browser extensions](#iframes-and-browser-extensions)

## content security policy

skip this section when the site sends no `content-security-policy` header or meta tag. otherwise choose one of two setups. both were checked in chromium under a real policy header, with preparation, transcription, recording and playback.

**built-in worker (the default).** nothing extra to host:

```text
Content-Security-Policy: default-src 'self'; script-src 'self' 'wasm-unsafe-eval' data:; worker-src 'self' blob:; connect-src 'self' data: https://cdn.karanganesan.com https://huggingface.co https://*.hf.co; media-src 'self' blob:
```

**hosted files (`assetBaseURL`).** no `blob:` or `data:` sources:

```text
Content-Security-Policy: default-src 'self'; script-src 'self' 'wasm-unsafe-eval'; worker-src 'self'; connect-src 'self' https://cdn.karanganesan.com https://huggingface.co https://*.hf.co; media-src 'self' blob:
```

| directive | why |
| --- | --- |
| `script-src 'wasm-unsafe-eval'` | vocule runs webassembly |
| `script-src data:` | built-in setup only: the capture worklet for `listen()` and `record()` is a `data:` url |
| `worker-src blob:` or `'self'` | the built-in worker is a `blob:` url; the hosted worker is on your origin |
| `connect-src` model hosts | the model comes from `https://cdn.karanganesan.com`, falling back to `https://huggingface.co` and its download hosts `https://*.hf.co`; with `modelBaseURL`, list that origin instead |
| `connect-src data:` | built-in setup only: the built-in worker loads its webassembly from a `data:` url |
| `media-src blob:` | `clip.play()` plays a `blob:` url |

also allow any origin the app passes to `transcribe()` as a url. when a directive is missing, the browser console names it, and vocule fails like this:

| blocked | result |
| --- | --- |
| the worker | `BACKEND_UNAVAILABLE`: "the speech worker failed. check worker csp and urls." |
| `data:` in `connect-src` (built-in worker) | `MODEL_FORMAT`: "Failed to fetch" |
| `'wasm-unsafe-eval'` | `MODEL_FORMAT`, with a message about compiling webassembly |
| the capture worklet | `listen()` and `record()` reject with a `DOMException` named `AbortError`: "Unable to load a worklet's module" |
| the model hosts | `NETWORK` |

## hosting vocule's files on your origin

copy the package's `dist` folder into the folder the app serves as static files, and point vocule at it:

```sh
mkdir -p public/vocule
cp -R node_modules/vocule/dist/. public/vocule/
```

```ts
const speech = createSpeech({ assetBaseURL: "/vocule/" });
```

- the static folder is `public/` in vite, next.js, nuxt, astro and recent angular projects (`src/assets/` in older angular ones), and `static/` in sveltekit.
- the folder must be on the page's own origin, with its files at their relative paths; a worker on another origin fails with `CAPABILITY`.
- serve `.wasm` files as `application/wasm`.
- copy again after every vocule upgrade. a script keeps it in step, for example `"prebuild": "rm -rf public/vocule && cp -R node_modules/vocule/dist public/vocule"` in `package.json`, with `public/vocule/` in `.gitignore`.
- `assetBaseURL` covers the worker and the capture worklet. `workerFactory` builds the worker yourself (`new Worker(url, { type: "module" })` for `vocule/worker`) and `workletURL` on `listen()` or `record()` overrides the worklet; both are only needed when one of them lives somewhere else.

## serving the model yourself

by default the model comes from vocule's cdn, falling back to hugging face. to serve it from your own origin or storage, download the pinned revision's files once, using the urls in the installed package:

```sh
BASE=$(node --input-type=module -e 'import { MODEL_CDN_URL } from "vocule/model"; console.log(MODEL_CDN_URL)')
mkdir -p public/models/parakeet-redux
for file in config.json ternary.json tokenizer.json model.safetensors README.md; do
  curl -fL "$BASE$file" -o "public/models/parakeet-redux/$file"
done
```

```ts
const speech = createSpeech({ modelBaseURL: "/models/parakeet-redux/" });
```

- the files must stay byte for byte identical: vocule checks the pinned hashes on every load, so a mirror can move the model but never change it. a changed file fails with `INTEGRITY`.
- with `modelBaseURL` there is no fallback source.
- serve the files over https (plain http works on localhost). another origin needs cors, and a content security policy must list it in `connect-src`.
- `model.safetensors` is about 178 mb; some static hosts cap file sizes, so object storage or a cdn may suit it better. cache it for a long time: the revision never changes.
- `README.md` is the model card. the weights are licensed cc by 4.0: keep the card and the attribution with any copy you serve.
- repeat the download after upgrading vocule, since a new version can pin a new revision (`MODEL.revision` in `vocule/model`).

## caching

- with the default `cache: true`, the verified model is kept in the browser's cache storage, and later visits prepare without downloading it.
- browsers can evict cache storage under storage pressure, and private windows do not keep it. `navigator.storage.persist()` asks the browser to keep the site's storage; call it after a successful preparation if the app depends on the model being available.
- `cache: false` stores nothing, for shared or kiosk machines.
- clearing the site's data removes the model; the next preparation downloads it again.

## bundlers, server rendering and tests

- vocule is published as es modules only. `import` works everywhere; `require("vocule")` fails with `ERR_PACKAGE_PATH_NOT_EXPORTED`.
- bundlers that build for the browser pick vocule's browser build through its `browser` export condition. in a vite production build, the engine and its worker were a separate chunk of about 130 kb (about 41 kb gzipped) that loads on the first model call.
- vite and bun apps need no configuration. with any other bundler, run the production build and prepare once in a browser before shipping; if the worker fails to start, use the hosted files above.
- importing vocule during server rendering has no side effects, but every model call needs the browser. keep calls in effects and event handlers.
- in unit tests (vitest, jest with jsdom), there is no webgpu or worker: mock the shared `src/lib/speech.ts` module instead of vocule itself, and test real transcription in a browser, for example with playwright against chromium.

## browser support

vocule needs webgpu, webassembly and a secure context (https or localhost). it was tested in chrome 153 and safari 26.6 on apple silicon; other browsers, gpus, phones and low-memory devices are not measured, so test the ones you plan to support.

a quick check decides whether to show the feature at all:

```ts
// src/lib/speech-support.ts
export function mightSupportSpeech(): boolean {
  return globalThis.isSecureContext === true && "gpu" in navigator;
}
```

it cannot prove that a working gpu adapter exists: on a device that passes it but cannot run the model, `prepare()` fails with `BACKEND_UNAVAILABLE`, so handle that code too. `runDiagnostics()` from `vocule/diagnostics` runs a quick self-test of the browser's webgpu and webassembly support, useful on a support or debug page. which audio and video formats `transcribe()` accepts depends on the browser's decoder; wav works everywhere vocule runs.

## iframes and browser extensions

- a cross-origin iframe needs `allow="microphone"` for `listen()` and `record()`, and the parent's `permissions-policy` must not block the microphone.
- extension pages (manifest v3) have a strict default policy, so use the hosted-files setup: package `node_modules/vocule/dist` in the extension, pass `createSpeech({ assetBaseURL: chrome.runtime.getURL("vocule/") })`, add `'wasm-unsafe-eval'` to the `extension_pages` policy, and make sure the model hosts are reachable. this path has not been tested; verify it in the target browser.
