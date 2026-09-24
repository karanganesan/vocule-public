# troubleshooting

start with the browser console: vocule's errors carry a stable `code`, and content security policy problems name the blocked directive there.

## symptoms

| symptom | likely cause | fix |
| --- | --- | --- |
| `BUSY` | `transcribe()` or a live session overlaps another inference call on the same instance | await the previous call; disable the controls while one runs; create another instance only for work that truly overlaps |
| `BUSY` from `pause()` or `resume()` | the recording is not in the state the call needs | check `session.state` first |
| `BACKEND_UNAVAILABLE` | no usable webgpu (an older browser, plain http on a non-localhost address, a disabled or blocklisted gpu, some virtual machines and remote desktops), or a content security policy blocking the worker | check `isSecureContext` and `"gpu" in navigator`, then the console for a policy error; see [deployment.md](deployment.md) |
| `BACKEND_UNAVAILABLE` during preparation with a `data:` CSP error in the console | a content security policy without `data:` in `connect-src`, with the built-in worker | add `data:` to `connect-src`, or switch to hosted files |
| preparation fails with a webassembly compilation CSP error | `script-src` without `'wasm-unsafe-eval'` | add it |
| `listen()` or `record()` rejects with `AbortError` "Unable to load a worklet's module" | a content security policy blocking the built-in `data:` worklet | add `data:` to `script-src`, or use `assetBaseURL` |
| `NETWORK` | the model hosts are unreachable: offline, a proxy or ad blocker, or `connect-src` | check the network panel; allow the hosts, or serve the model with `modelBaseURL` |
| `INTEGRITY` | a mirror or proxy served different bytes | download the files again from the pinned revision, unmodified |
| the model downloads on every visit | `cache: false`, a private window, or cleared or evicted site data | keep `cache` on; consider `navigator.storage.persist()` |
| `NotAllowedError` from `listen()` or `record()` | the user or a policy denied the microphone, or a cross-origin iframe lacks `allow="microphone"` | explain how to allow the microphone in the site settings; add `allow="microphone"` to the iframe |
| `NotFoundError` from `listen()` or `record()` | no microphone is connected | say so and offer file upload instead |
| `CAPABILITY` from `listen()` or `record()` | the page is not https or localhost, the browser lacks a capture api, or a supplied stream has no audio track | serve over https; pass a stream with an audio track |
| `CAPABILITY` from `transcribe()` | `timestamps` set to `"segment"` or `"word"` | remove the option; timestamps are not available |
| `AUDIO_INVALID` | a codec or container this browser cannot decode, an empty or corrupt file, a bare `Float32Array` without `sampleRate`, `sampleRate` passed with another input, or a `MediaStream` passed to `transcribe()` | convert to wav or a format the browser plays; give raw pcm its rate; send streams to `listen()` or `record()` |
| `BACKPRESSURE` | live audio queued faster than it could be transcribed, often speech that started while the model was still downloading, or a slow gpu | await the preparation before listening; raise `draftIntervalMs` or set it to `0`; start a new session |
| `GPU_LOST` or `GPU_ERROR` | the gpu device reset: sleep, a driver update, memory pressure or too many gpu-heavy tabs | offer a retry; the next call prepares the model again |
| `DECODE_LIMIT` | a safety limit stopped decoding, and no partial text is returned | try the audio again or in shorter pieces; report it if it repeats |
| `DISPOSED` | a call on an instance after `dispose()` | create a new instance; with the shared module, call `disposeSpeech()` and then `getSpeech()` |
| `ABORTED` right after mounting in development | react strict mode unmounted the component and its cleanup disposed the instance | keep the instance in the shared module, and do not dispose it in a component cleanup |
| empty text | silence, a muted or wrong microphone, raw pcm with the wrong `sampleRate`, or, live, audio below `speechThreshold` | check `onLevel`, the input device and the rate |
| live text stays in draft | the speaker has not paused | settled text arrives after `silenceMs` of quiet, or at `maxUtteranceMs`; `stop()` settles the rest |
| `Worker is not defined` or similar during server rendering or tests | a model call outside a browser | move calls into effects and event handlers; mock `src/lib/speech.ts` in unit tests |
| `ERR_PACKAGE_PATH_NOT_EXPORTED` | `require("@karanganesan/vocule")`: the package is es modules only | use `import`, or an esm-capable test runner such as vitest |
| the same audio gives slightly different text as mp3 and as wav | compressed formats and resampling change the samples | pass wav or pcm when exact text matters |

## error codes at a glance

| code | show the user | then |
| --- | --- | --- |
| `ABORTED` | nothing; the app or user cancelled | nothing |
| `AUDIO_INVALID` | the file could not be read as audio | let them pick another file |
| `BACKEND_UNAVAILABLE` | this browser or device cannot run on-device speech recognition | hide or disable the feature |
| `CAPABILITY` | the option or capability is not available | fix the call, or explain the missing browser feature |
| `NETWORK` | the model could not be downloaded | offer a retry |
| `INTEGRITY`, `MODEL_FORMAT` | speech recognition could not start | check the model source and the content security policy |
| `BUSY` | still working | wait for the running call |
| `BACKPRESSURE` | speech recognition fell behind | start a new session |
| `DISPOSED` | nothing | create a new instance |
| `GPU_LOST`, `GPU_ERROR` | something went wrong with the graphics processor | offer a retry |
| `DECODE_LIMIT` | the audio could not be transcribed | try again or in shorter pieces |
