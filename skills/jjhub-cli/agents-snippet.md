# Version control in this repo goes through JJHub

This repository is tracked by JJHub (a change/stack/operation overlay on
GitHub). Use the `jjhub` CLI for version-control mutations instead of raw
`git`/`gh` — every `jjhub` action is attributed in an operation log and
reversible with `jjhub undo`, and landing is CI-gated.

- Start work: `jjhub change new "<title>"` (not a git branch).
- Describe/retitle: `jjhub change describe CHG-N --title t --description d`.
- Amend after edits: `jjhub change amend CHG-N` (descendants get flagged and
  restack with one command — no force-push ceremony).
- Stacked work: `jjhub stack create "<title>" --changes CHG-1,CHG-2`;
  reorder with `jjhub stack reorder`; land root-to-tip with
  `jjhub stack land <stackId>` (all-or-nothing, CI-gated).
- Ship a single change: `jjhub change land CHG-N`. Do NOT merge the PR with
  `gh pr merge` — that bypasses the gate and the op log.
- Made a mistake: `jjhub undo` (works across changes, stacks, bookmarks, and
  conflicts; `jjhub op log` shows what would be undone).
- Conflicts: `jjhub conflict list` / `jjhub conflict resolve <id> <state>` —
  conflicts are durable objects; you can defer them and keep working.
- Machine output: append `--json` to any command.
- Repo context is inferred from the git `origin` remote; `--repo owner/name`
  overrides.

Reads are fine with either tool. Any `jjhub` command it doesn't recognize
(`pr`, `issue`, ...) passes through to the real `gh` unchanged. Full command
reference: run `jjhub help`, or see the served skill at
`/.well-known/skills/jjhub-cli/SKILL.md` on your JJHub server.
