# vocule

private speech to text that runs entirely in the browser.

this repository is where issues, questions and feedback about [vocule](https://www.npmjs.com/package/@karanganesan/vocule) live, along with an [agent skill](#agent-skill) for building with it. vocule's source code is not open source; the package, its documentation and its benchmarks are on npm:

```sh
bun install @karanganesan/vocule
pnpm install @karanganesan/vocule
npm install @karanganesan/vocule
```

import the package as `@karanganesan/vocule` too; the unscoped `vocule` name is not published on npm.

- **website:** [karanganesan.com/vocule](https://karanganesan.com/vocule)
- **documentation:** [vocule on npm](https://www.npmjs.com/package/@karanganesan/vocule)
- **bugs and feature requests:** [open an issue](https://github.com/karanganesan/vocule-public/issues/new)

## agent skill

[`skills/vocule`](skills/vocule/SKILL.md) is an [agent skill](https://agentskills.io) that teaches coding agents to add vocule to a web app: one shared instance, file transcription, live dictation, recording, download progress, errors, content security policy, and examples for react, next.js, vue, nuxt, svelte, sveltekit, angular, astro and plain typescript. it works in claude code, codex, cursor, github copilot, gemini cli, opencode and any other agent that reads the open skill format. install it with the [skills](https://github.com/vercel-labs/skills) cli:

```sh
npx skills add karanganesan/vocule-public --skill vocule
```

the cli finds the agents on your machine and asks where to install; `-a claude-code -a codex` picks agents, `-g` installs for your user instead of the project, and `npx skills update vocule` fetches a newer version. to copy it by hand for a project, put the `skills/vocule` folder in `.claude/skills/` for claude code or `.agents/skills/` for codex and other agents that read that path. for a user-wide codex install, put it in `~/.codex/skills/vocule/`.

then ask for what you want, such as "add live dictation to the notes page with vocule". the skill loads when a request fits; to call it by name, type `/vocule` in claude code or `$vocule` in codex.

## let's talk

for anything that doesn't fit an issue, such as questions, ideas or something you are building with vocule, write to [hey@karanganesan.com](mailto:hey@karanganesan.com).
