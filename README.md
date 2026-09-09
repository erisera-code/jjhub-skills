# jjhub-skills

Agent skills for [JJHub](https://jjhub.erisera.com) — JJHub — a Jujutsu-native overlay atop GitHub: stable change IDs, stacked PRs, platform-wide undo, and an MCP server for coding agents.

This repository is a **generated, read-only mirror** of the `skills/` directory from the
private [jjhub](https://github.com/erisera-code/jjhub) repository, published on every
change so agent tooling has a stable public place to install from
(`.github/workflows/publish-skills.yml` / `scripts/build-skills-dist.ts` in that repo
build it; do not hand-edit files here — they will be overwritten on the next sync). Current
as of jjhub v0.3.0.

## What's in here

- [`skills/jjhub-cli`](./skills/jjhub-cli) — Drive a running JJHub instance (a Jujutsu-native change/stack/operation overlay on GitHub) from the command line with the `jjhub` CLI, or as an agent over MCP, ACP, A2A, or WebMCP — create and land changes, manage stacks and bookmarks, resolve conflicts, coordinate local jj workspaces, and undo/redo via the operation log. Use when scripting or automating repository operations against a JJHub server, or wiring an agent client (Claude Code, Zed, a Claude/ChatGPT connector, an A2A peer) up to one.

`index.json` at the repo root lists every skill in the same shape JJHub's own
`GET /.well-known/skills/` endpoint serves, plus an `installation` block.

## Install

Pick whichever matches your tooling — all three land the same skill content.

**With the [skills.sh](https://skills.sh) installer:**

```sh
npx skills add erisera-code/jjhub-skills
```

**As a Claude Code plugin marketplace** (`.claude-plugin/marketplace.json` in this repo):

```
/plugin marketplace add erisera-code/jjhub-skills
/plugin install jjhub-cli@jjhub-skills
```

**Teach your repo's coding agents to prefer `jjhub` over raw git/gh**, straight from the CLI
(no clone of this repo needed):

```sh
npx jjhub@latest agents-md >> AGENTS.md
```

## Using jjhub itself

JJHub is a Jujutsu-native change/stack/operation overlay on GitHub. The live instance is at
https://jjhub.erisera.com; full docs start at
[`docs/guides/GETTING-STARTED.md`](https://github.com/erisera-code/jjhub/blob/main/docs/guides/GETTING-STARTED.md)
in the main repository, and the CLI/agent reference lives in
[`skills/jjhub-cli/SKILL.md`](./skills/jjhub-cli/SKILL.md) right here. Agent protocol
surfaces (MCP, ACP, A2A, WebMCP) are documented in that same file's "Install this skill"
section.

## License

MIT licensed — see [LICENSE](./LICENSE). (The jjhub CLI and server are distributed separately under their own terms.)
