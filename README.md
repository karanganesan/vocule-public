# vocule

private speech to text that runs entirely in the browser.

this repository is where issues, questions and feedback about [vocule](https://www.npmjs.com/package/vocule) live, along with an [agent skill](#agent-skill) for building with it. vocule's source code is not open source; the package, its documentation and its benchmarks are on npm:

```sh
bun install vocule
pnpm install vocule
npm install vocule
```

- **website:** [karanganesan.com/vocule](https://karanganesan.com/vocule)
- **documentation:** [vocule on npm](https://www.npmjs.com/package/vocule)
- **bugs and feature requests:** [open an issue](https://github.com/karanganesan/vocule-public/issues/new)

## agent skill

[`skills/vocule`](skills/vocule/SKILL.md) is an [agent skill](https://agentskills.io) that teaches coding agents to add vocule to a web app: one shared instance, file transcription, live dictation, recording, download progress, errors, content security policy, and examples for react, next.js, vue, nuxt, svelte, sveltekit, angular, astro and plain typescript. it works in claude code, codex, cursor, github copilot, gemini cli, opencode and any other agent that reads the open skill format. install it with the [skills](https://github.com/vercel-labs/skills) cli:

```sh
npx skills add karanganesan/vocule-public
```

the cli finds the agents on your machine and asks where to install; `-a claude-code -a codex` picks agents, `-g` installs for your user instead of the project, and `npx skills update` fetches newer versions. `npx skills install` is the same command. to copy it by hand, put the `skills/vocule` folder in `.claude/skills/` for claude code, or in `.agents/skills/` for codex, cursor, github copilot, gemini cli and opencode.

then ask for what you want, such as "add live dictation to the notes page with vocule". the skill loads when a request fits; to call it by name, type `/vocule` in claude code or `$vocule` in codex.

## let's talk

for anything that doesn't fit an issue, such as questions, ideas or something you are building with vocule, write to [hey@karanganesan.com](mailto:hey@karanganesan.com).
