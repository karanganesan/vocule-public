# vocule

private speech to text, in your browser. a webgpu and webassembly inference engine for speech to text models.

this repository is where issues, questions and feedback about [vocule](https://www.npmjs.com/package/@karanganesan/vocule) live, along with an [agent skill](#agent-skill) for building with it. vocule's source code is not open source; the package, its documentation and its benchmarks are on npm:

```sh
bun install @karanganesan/vocule
pnpm install @karanganesan/vocule
npm install @karanganesan/vocule
```

this review branch contains draft guidance for the **2.0.0 release candidate**. npm still supplies the published 1.x contract until the release owner publishes 2.0.0; do not treat these examples as 1.x calls. the matching website, sdk package and skill must be released together.

from 2.0.0, every factory call requires a lowercase `model`, with no sdk default:

```ts
import { createSpeech } from "@karanganesan/vocule";

const speech = createSpeech({ model: "moondream/parakeet-redux" });
// alternative: createSpeech({ model: "fermionresearch/phonon-2" })
```

choose one instance for the feature. other options, file transcription, live dictation and recording share the same contract. selecting a model does not download it; preparation and inference start loading. audio processing stays on the device. model files download from vocule’s cdn with an immutable hugging face fallback for the same checkpoint, without a speech inference server.

**migration:** published 1.x accepts `createSpeech()` and selects redux. 2.0.0 intentionally rejects absent, unknown or mixed-case model ids synchronously with `CAPABILITY`. to keep the old checkpoint, pass `model: "moondream/parakeet-redux"` explicitly. lean imports at `@karanganesan/vocule/phonon-2` and `@karanganesan/vocule/parakeet-redux` require the matching model and exclude the other engine.

phonon-2 is an english model by fermion research; redux’s upstream card lists 25 languages, with english, spanish and french tested in 1.0.0. the models’ weights retain their own cc by 4.0 licenses and source attribution; the sdk code license does not relicense them.

the audited 2.0.0 candidate completed all 5,559 english librispeech test-clean and test-other clips through each model’s default file transcription path, with no failed transcriptions. pooled word error was **3.4633% for phonon-2** and **3.8656% for parakeet redux**, over 106,141 reference words. phonon minus redux was −0.4023 percentage points, with a paired 95% speaker interval from −0.4997 to −0.3020; the frozen accuracy checks passed. the scoring method and pinned package were independently reviewed. accuracy and speed use separate inputs and measurements.

on the tested m4 macbook air with chrome 153, the 3.4 s jane eyre complete-file call had median / p95 **56.7 / 60.02 ms for phonon-2** and **56.1 / 59.41 ms for redux**, with 15 calls per model over three matched sessions. the separate 11 s jfk file had medians of 144.3 and 141.5 ms. one 1505.81 s recording took 15.0058 and 16.9578 s, respectively: **100.3× and 88.8×** audio seconds per wall second. that was each model’s first public file call after preparation and input validation, including lazy compilation inside the call; there is no long-file p95. those two calls ran back to back, phonon-2 first; in a second, separate session with a rest before each, the same first calls took 15.5793 and 15.7538 s (96.7× and 95.6×). these are sparse descriptive observations under background load: 19 sessions and 236 timed calls. the planned performance and repeated memory qualification remains incomplete. file, pcm, live, saved-model preparation and native settings are measured separately. safari has five calls per file/pcm/preparation cell; live and long-file timings are unmeasured.

these timings are for the engine build that follows the audited candidate: the same checkpoints and weight values, with the encoder’s work laid out differently on the gpu. it is a reproducible build that has not been independently audited. it returned the same text as the audited build for every input timed here, and its encoder outputs matched bit for bit on every input checked; the word-error figures above were scored on the audited build and have not been scored again on this one. compared in one session against the audited build, its first round ran the 25-minute recording at about 88× instead of 62× for phonon-2 and about 90× instead of 67× for redux. a second pass added about 2% in chrome, measured on a quieter machine where the two builds ran it at 102× and 104× for phonon-2 and 97× and 100× for redux, and 7% in safari on eight short clips.

the source checkpoints retain their original pinned bytes and reconstructed weight values. engine and cache layout optimizations preserve those values exactly; a compact cache representation is lossless storage. speed and transcription quality are measured separately.

import the package as `@karanganesan/vocule` too; the unscoped `vocule` name is not published on npm.

- **website:** [karanganesan.com/vocule](https://karanganesan.com/vocule)
- **documentation:** [vocule on npm](https://www.npmjs.com/package/@karanganesan/vocule)
- **bugs and feature requests:** [open an issue](https://github.com/karanganesan/vocule-public/issues/new)

## agent skill

[`skills/vocule`](skills/vocule/SKILL.md) is an [agent skill](https://agentskills.io) that teaches coding agents to add vocule to a web app: required explicit model selection from 2.0.0, one shared instance, file transcription, live dictation, recording, download progress, errors, content security policy, and examples for react, next.js, vue, nuxt, svelte, sveltekit, angular, astro and plain typescript. it works in claude code, codex, cursor, github copilot, gemini cli, opencode and any other agent that reads the open skill format. install it with the [skills](https://github.com/vercel-labs/skills) cli:

```sh
npx skills add karanganesan/vocule-public --skill vocule
```

the cli finds the agents on your machine and asks where to install; `-a claude-code -a codex` picks agents, `-g` installs for your user instead of the project, and `npx skills update vocule` fetches a newer version. to copy it by hand for a project, put the `skills/vocule` folder in `.claude/skills/` for claude code or `.agents/skills/` for codex and other agents that read that path. for a user-wide codex install, put it in `~/.codex/skills/vocule/`.

the skill in this review branch targets 2.0.0. install or refresh the skill from the merged public source only after the matching npm release is confirmed; existing installed 1.x skills keep their versioned guidance. this repository does not generate the npm package or synchronize installed skills automatically.

then ask for what you want, such as "add live dictation to the notes page with vocule". the skill loads when a request fits; to call it by name, type `/vocule` in claude code or `$vocule` in codex.

## let's talk

for anything that doesn't fit an issue, such as questions, ideas or something you are building with vocule, write to [hey@karanganesan.com](mailto:hey@karanganesan.com).
