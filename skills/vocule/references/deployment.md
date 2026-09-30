# deployment

draft guidance for the **2.0.0 release candidate**. published 1.x selects redux implicitly; every new-major factory call below requires a selected model. validate against the packed declarations and matching worker assets.

## contents

- [content security policy](#content-security-policy)
- [hosting vocule's files on your origin](#hosting-vocules-files-on-your-origin)
- [serving the model yourself](#serving-the-model-yourself)
- [caching](#caching)
- [bundlers, server rendering and tests](#bundlers-server-rendering-and-tests)
- [browser support](#browser-support)
- [iframes and browser extensions](#iframes-and-browser-extensions)

## content security policy

skip this section when the site sends no policy. the 1.0.0 policies below were checked in chromium under a real header with preparation, transcription, recording and playback. repeat those checks for each 2.0.0 model and root/lean delivery; historical tests do not qualify a new worker.

**built-in worker.** nothing extra to host:

```text
Content-Security-Policy: default-src 'self'; script-src 'self' 'wasm-unsafe-eval' data:; worker-src 'self' blob:; connect-src 'self' data: https://cdn.karanganesan.com https://huggingface.co https://*.hf.co; media-src 'self' blob:
```

**hosted files (`assetBaseURL`).** no worker `blob:` or wasm/worklet `data:` sources:

```text
Content-Security-Policy: default-src 'self'; script-src 'self' 'wasm-unsafe-eval'; worker-src 'self'; connect-src 'self' https://cdn.karanganesan.com https://huggingface.co https://*.hf.co; media-src 'self' blob:
```

| directive | why |
| --- | --- |
| `script-src 'wasm-unsafe-eval'` | vocule runs webassembly |
| `script-src data:` | built-in capture worklet for `listen()` and `record()` |
| `worker-src blob:` or `'self'` | built-in worker uses a `blob:` url; hosted worker stays on your origin |
| `connect-src` model hosts | verified cdn first, pinned hugging face and `*.hf.co` download hosts as fallback; with `modelBaseURL`, list that origin instead |
| `connect-src data:` | built-in worker loads its webassembly from a `data:` url |
| `media-src blob:` | `clip.play()` |

also allow any origin passed to `transcribe()` as an audio url. the browser console names blocked directives. a blocked worker or wasm generally fails with `BACKEND_UNAVAILABLE`; unreachable model hosts fail with `NETWORK`. a blocked capture worklet can reject with the browser's `AbortError` and “Unable to load a worklet's module”.

## hosting vocule's files on your origin

copy the root `dist` tree for the root factory, retaining relative imports:

```sh
mkdir -p public/vocule
cp -R node_modules/@karanganesan/vocule/dist/. public/vocule/
```

```ts
import { createSpeech } from "@karanganesan/vocule";
const speech = createSpeech({
  model: "fermionresearch/phonon-2",
  assetBaseURL: "/vocule/",
});
```

- the static folder is `public/` in vite, next.js, nuxt, astro and recent angular projects (`src/assets/` in older angular ones), and `static/` in sveltekit.
- keep workers on the page's own origin; a different worker origin fails with `CAPABILITY`. serve wasm as `application/wasm` and modules as javascript.
- copy again after every upgrade. a prebuild copy script can keep generated hosting files in step; ignore that copied directory in version control.
- `assetBaseURL` covers the worker and capture worklet. `workerFactory: () => Worker` keeps its zero-argument signature and takes priority. `workletURL` on `listen()` or `record()` overrides capture's worklet when needed.

### lean hosted entries

copy only the matching model directory for a lean factory:

```sh
mkdir -p public/vocule/phonon-2
cp -R node_modules/@karanganesan/vocule/dist/phonon-2/. public/vocule/phonon-2/
```

```ts
import { createSpeech } from "@karanganesan/vocule/phonon-2";
const speech = createSpeech({
  model: "fermionresearch/phonon-2",
  assetBaseURL: "/vocule/phonon-2/",
});
```

for redux use `@karanganesan/vocule/parakeet-redux`, the matching required `model: "moondream/parakeet-redux"`, and `dist/parakeet-redux/`. the applicable directory contains its own `worker.js` and minimal asset tree. root `dist/worker.js` dispatches lazily to both choices. matching worker exports are `@karanganesan/vocule/worker`, `@karanganesan/vocule/phonon-2/worker` and `@karanganesan/vocule/parakeet-redux/worker`.

a supplied stale or wrong-model worker fails its version/supported-id handshake with `CAPABILITY` before weights load. copy the entire applicable tree from the same package version, including required notices, rather than mixing old workers with new factories.

## serving the model yourself

each selected model uses a verified cdn mirror first and an immutable hugging face fallback for **the same artifact bytes and checkpoint**. this is transport recovery: it never uploads audio or substitutes another model. phonon's canonical source repository is `FermionResearch/Phonon-2`; the public option remains lowercase.

use the installed selected descriptor instead of a fixed four-file loop. this metadata-only staging script does not run an inference engine:

```ts
// scripts/stage-speech-model.ts (run with bun)
import { mkdir } from "node:fs/promises";
import { getModelDescriptor } from "@karanganesan/vocule/model";

const descriptor = getModelDescriptor("fermionresearch/phonon-2");
const directory = "public/models/phonon-2";
await mkdir(directory, { recursive: true });
for (const artifact of descriptor.artifacts) {
  const response = await fetch(new URL(artifact.name, descriptor.sources[0]));
  if (!response.ok) throw new Error(`model download failed: ${response.status}`);
  const bytes = await response.arrayBuffer();
  const hash = [...new Uint8Array(await crypto.subtle.digest("SHA-256", bytes))]
    .map((byte) => byte.toString(16).padStart(2, "0")).join("");
  if (bytes.byteLength !== artifact.bytes || hash !== artifact.sha256)
    throw new Error(`model identity mismatch: ${artifact.name}`);
  await Bun.write(`${directory}/${artifact.name}`, bytes);
}
```

```ts
const speech = createSpeech({
  model: "fermionresearch/phonon-2",
  modelBaseURL: "/models/phonon-2/",
});
```

- `modelBaseURL` mirrors the **selected pinned representation**, with no default-source fallback. it is not an arbitrary-model loader. redux uses safetensors plus its three metadata files. phonon's cold representation is the byte-identical `phonon-2.bps.tar.zst`, containing `model.fermion`, authoritative `config.json`/`packed_manifest.json` and its attestation. do not substitute the different root metadata or a derived pack.
- keep exact filenames, bytes, hashes and lengths from the installed descriptor/release provenance. serve `.zst` as `application/zstd`, safetensors as `application/octet-stream`, json as `application/json`, and unchanged notices as text. model files do not use a texture's four-byte alignment rule.
- `artifactHashes` is discriminated by `model`: redux uses `config.json`, `ternary.json`, `tokenizer.json`; phonon uses `config.json`, `packed_manifest.json`. metadata overrides cannot change weight pins, checkpoint or architecture. ordinary mirrors retain the pinned hashes.
- use https (localhost http is allowed). a different model origin needs cors and a `connect-src` allowance. immutable storage/cdn hosts suit files larger than a static host's cap; never overwrite a pinned url.
- **before serving a mirror**, retain the applicable model card, source/revision/change attribution, cc by 4.0 weight license and supplied notices from the verified redistribution set. phonon supplies `LICENSE-WEIGHTS-CC-BY-4.0.txt`, `NOTICE` and an apache-2.0 code license for reference code; redux retains moondream/nvidia credit. the sdk's code license does not relicense weights. a notice/card may use a text filename when its bytes and mapping are preserved. copying only runtime artifacts does not complete redistribution attribution.
- repeat staging after an upgrade. source checkpoint bytes, compressed transfer, extracted/repacked cache bytes and gpu/wasm/process residency are distinct quantities.
- preserve the original checkpoint bytes and all reconstructed source weight values. compact cache layouts are lossless storage; their version and access layout can change while checkpoint identity and values stay fixed.

## caching

- `cache: true` reuses verified bytes after reload. redux preserves the exact 1.x cache name and keys, so 2.0.0 can reuse a populated 1.0.0 redux cache. phonon uses separate checkpoint/member-hash/pack-version keys independent of serving host; warm preparation reuses verified extracted/repacked bytes without decompressing the archive again.
- `getModelCacheStatus(id)` reads key presence only. a selector may display that hint; preparation still verifies integrity. do not download or hash full weights just to draw a picker.
- corrupt entries are replaced individually. quota failure is tolerated without clearing the other model's cache. successful preparation proves readiness in memory, not that persistent cache writes succeeded; recheck the lightweight presence hint before labeling a model saved on this device.
- browsers may evict site storage, and private windows may not persist it. `navigator.storage.persist()` can request retention after successful preparation. `cache: false` stores nothing; clearing site data makes the next preparation download again.

## bundlers, server rendering and tests

- vocule is es modules only; use `import`. `require("@karanganesan/vocule")` fails with `ERR_PACKAGE_PATH_NOT_EXPORTED`.
- browser bundlers select the browser export condition. root factories dispatch lazily; a lean entry excludes the other adapter/reader/worker assets and still requires its matching id. verify actual production output, including inline-worker contents, rather than assuming a root runtime string guarantees elimination.
- metadata imports no worker, wasm or engine. 1.0.0's metadata-only consumer measured 959 bytes; final packed 2.0.0 emitted-byte measurements are pending.
- vite and bun need no special configuration. test production preparation/transcription with the target bundler; hosted files are available when its worker handling requires them.
- imports during server rendering have no model side effects, but model calls belong in browser events/effects/mount hooks.
- jsdom unit tests lack webgpu/workers. mock the app's shared `src/lib/speech.ts` and use real browser tests for inference, inline/hosted/csp paths, capture, cancellation and both choices.

## browser support

vocule needs webgpu, webassembly and https or localhost. published 1.0.0 was tested in chrome 153 and safari 26.6 on apple silicon; final 2.0.0 per-model chrome/safari qualification is pending. other devices, phones, low-memory adapters and browsers need their own evidence. phone layout testing does not prove phone inference support. no full-model wasm-only/webgl or cloud inference fallback is shipped.

```ts
export function mightSupportSpeech(): boolean {
  return globalThis.isSecureContext === true && "gpu" in navigator;
}
```

this does not prove a usable gpu adapter exists. `prepare()` may fail with `BACKEND_UNAVAILABLE`; handle it. `runDiagnostics()` from `@karanganesan/vocule/diagnostics` is a browser webgpu/wasm self-test, not a model quality test. audio/video formats depend on the browser decoder; wav works wherever vocule runs.

## iframes and browser extensions

- cross-origin microphone use needs `allow="microphone"` on the iframe and an unblocked parent `permissions-policy`.
- manifest v3 extension pages should use hosted assets, the selected model and `'wasm-unsafe-eval'` in the `extension_pages` policy. example: `createSpeech({ model: "fermionresearch/phonon-2", assetBaseURL: chrome.runtime.getURL("vocule/") })`. preserve matching asset paths and model-host policy. this extension path remains untested; validate in the target browser.
