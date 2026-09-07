---
name: jjhub-cli
description: Drive a running JJHub instance (a Jujutsu-native change/stack/operation overlay on GitHub) from the command line with the `jjhub` CLI, or as an agent over MCP, ACP, A2A, or WebMCP — create and land changes, manage stacks and bookmarks, resolve conflicts, coordinate local jj workspaces, and undo/redo via the operation log. Use when scripting or automating repository operations against a JJHub server, or wiring an agent client (Claude Code, Zed, a Claude/ChatGPT connector, an A2A peer) up to one.
---

License: MIT (see the public mirror erisera-code/jjhub-skills, LICENSE).

# jjhub — the JJHub CLI and agent surfaces

`jjhub` is a GitHub CLI (`gh`)-like wrapper with Jujutsu-native superpowers. It is a thin
client over JJHub's HTTP API: every command reads or mutates state on a running JJHub
server, the same server backing the web UI. Repo context is inferred from the current
directory's git `origin` remote (override with `--repo`), and any top-level command that
isn't one of `jjhub`'s own nouns is passed straight through to the real `gh` binary.

The CLI is one of five interchangeable surfaces over the same server state and the same
op log — pick whichever fits the client:

| Surface | What it is | Entry point |
|---|---|---|
| CLI | `jjhub`, a `gh`-alike | `npx jjhub@latest <command>`, or `jjhub` once installed |
| MCP | 132 structured tools | remote: `<server>/mcp` (OAuth 2.1, RFC 9728 auto-discovery); local: `npx jjhub@latest mcp serve` (stdio) or `--http <port>` |
| ACP | Agent Client Protocol v1, slash-command style | `npx jjhub@latest acp serve` (stdio) — Zed's agent panel and friends |
| A2A | Agent2Agent v1.0, one skill per registry command | agent card `<server>/.well-known/agent-card.json`; JSON-RPC `POST <server>/a2a`; REST `<server>/a2a/v1` |
| WebMCP | 133 tools, progressively loaded | in-browser only, from an open JJHub tab — no separate connection |

This file documents the CLI in depth (it is the most complete reference for every verb,
since ACP/A2A commands are the same verbs under the same names) and points at each other
surface's own guide for its protocol-specific mechanics. See "Install this skill" below
for how to wire any of these into an agent client.

## Setup

See the [top-level README](../../README.md) for full project setup (`npm install`,
tests, requirements). This section covers just the CLI.

0. **No install needed**: `npx jjhub@latest <command>` fetches and runs the CLI on demand.
   `npm i -g jjhub` gives a standalone `jjhub` binary if you'd rather not re-fetch every
   invocation.
1. **Start the JJHub server** (if not already running, and you're not pointing at a
   deployment): `npm start` from the repo root. Defaults to `http://localhost:3000`; set
   `PORT` to change it, `JJHUB_DB` to point at a different SQLite file.
2. **Point the CLI at it**: `export JJHUB_URL=http://localhost:3000` (this is the default,
   so only needed if the server is elsewhere). Against a real deployment, sign in first —
   `jjhub auth login --url https://<your-jjhub>` (GitHub device flow) saves a bearer token
   to `~/.config/jjhub/config.json`, and every subsequent command picks it up with no env
   vars needed. For a local dev server started with `JJHUB_API_KEY` set, instead
   `export JJHUB_API_KEY=<the same key>` — every request otherwise gets `401 unauthorized`.
   Precedence when both a token and a key are present: `JJHUB_TOKEN` (bearer, e.g. from
   `auth login`) beats `JJHUB_API_KEY` (operator key, sent as `x-api-key`) beats the saved
   config file. Exporting a login token as `JJHUB_API_KEY` does NOT work — the server
   rejects it (401). `JJHUB_CONFIG_DIR` overrides `~/.config/jjhub` for the CLI and every
   agent surface below, if you need an isolated config (a scratch credential, a test
   harness, a second identity).
3. **Invoke commands**: `npx jjhub@latest <command> [args]` or `jjhub <command>` once
   installed; from inside this repo in development, `npm run jjhub -- <command> [args]`.
4. **Initialize the overlay** for a repo before using any other command:
   `jjhub repo init [owner/name]` (infers `owner/name` from the cwd's git origin if omitted).

The same JJHub server also backs a web UI (`npm run web:dev` from the repo root, in a
second terminal — see [`web/README.md`](../../web/README.md)); the CLI and the web UI are
two interchangeable clients over the same state, not separate systems.

## Global flags

- `--repo <owner/name|repoId>` — override the inferred repository context.
- `--json` — emit machine-readable JSON instead of a human-readable table.
- `--url <baseUrl>` — override the server URL for this invocation (same as `JJHUB_URL`).

## Commands

```
repo init [owner/name] [--github-token pat]
                                        create the JJHub overlay for this repo
repo create owner/name --public|--private [--description text] [--default-branch name]
                                        create a NEW repo on GitHub (via the GitHub App) then
                                        overlay it — --public/--private is required, no default
repo list                              list repositories you can access
                                        (permissions come live from GitHub)
repo archive | unarchive               retire/reactivate (archived = read-only, sync skipped)
repo delete                            hard-delete incl. op log (ops credential only)

user list                              list known users (identity is GitHub-federated)

change new <title> [--parent CHG-x] [--stack id] [--route same-repo|unprojected|fork:<owner/name>]
                                        --route (issue #277): same-repo is the default; unprojected
                                        creates the change with no GitHub branch/PR ever; fork:<owner/name>
                                        is recorded but refused at projection time until real fork-PR
                                        projection ships
change list
change show <changeId>
change describe <changeId> [--title t] [--description d]
change amend <changeId>
change squash <changeId>
change abandon <changeId>
change restack <changeId>              rebase a needs-restack change onto its rewritten ancestor
change request-review <changeId>       move a draft change into review and mark its PR ready for review
change return-to-draft <changeId>      return an in-review change and its PR to draft
change context <changeId>              compact orientation snapshot: status, diff stat, landability,
                                        checks, conflicts, review health, and derived nextActions in one call
change commit <changeId> [<file...>] [-m msg] [--delete <path>]... [--expect-head <sha>]
                                        commit working-tree files onto the change's branch;
                                        --delete removes a path in the same commit (repeatable);
                                        --expect-head refuses (409) if the change moved since <sha>
change propose <title> [<file...>] [--parent CHG-x] [--stack id] [-m msg]
                        [--delete <path>]... [--expect-base <sha>]
                                        issue #374: create + commit in one call, printing the diff
                                        summary and PR link; --expect-base checks --parent's current
                                        commit; on partial failure the error names the created change
                                        id (still listed/abandon-able)
change diff <changeId>                 per-file diffs of what the change contains
change files <changeId>                list locally-stored files (contentMode "local" repos only)
change land <changeId> [--force] [--expect-head <sha>]
                                        waits for real CI triggered by its own ready flip,
                                        re-syncs a behind branch, then merges; --force bypasses;
                                        --expect-head refuses (409) if the change moved since <sha>
change enqueue <changeId>              add a review-ready pull request to GitHub's merge queue;
                                        GitHub owns the merge from here, JJHub marks the change
                                        landed once GitHub reports it merged
change dequeue <changeId>              withdraw a JJHub-queued pull request back to explicit review
change queue-status <changeId>         print the merge-queue receipt (position/state/merge-group
                                        sha/checks/removed reason) plus whether GitHub auto-merge
                                        is armed on the pull request
change split <changeId> <newTitle>
change checks <changeId>               list GitHub check runs for a change
change actions <changeId>              show bounded GitHub Actions workflow and job evidence
change actions rerun <changeId> <runId>
                                        ask GitHub to re-run a workflow run shown by "change actions"
change actions cancel <changeId> <runId>
                                        ask GitHub to cancel a workflow run shown by "change actions"
change deployments <changeId>          show bounded GitHub deployment evidence
change security <changeId>             show bounded GitHub code-scanning evidence
change issues <changeId>               show linked GitHub issue context for a change
change reviewers <changeId>            reviewer-routing audit trail: every request/remove that
                                        actually succeeded against GitHub, who did it, and when
change recommend-reviewers <changeId>  advisory-only reviewer suggestions from CODEOWNERS, prior
                                        review history, and recent file authorship — never requests
                                        a reviewer; use "change reviewers" to see actual requests
change review-health <changeId>        live GitHub review-collaboration summary: state, requested
                                        reviewers, stale/dismissed approvals, unresolved threads,
                                        CODEOWNERS evidence, and GitHub's effective merge requirement
change landability <changeId>          show the effective GitHub landability policy report — every
                                        gate's state and resolution, plus the overall verdict
change projects <changeId>             show bounded GitHub Projects v2 evidence for linked issues
change handoff <changeId>              durable task-handoff log (issue #373): findings, validation
                                        evidence, outstanding work, or a verified commit SHA another
                                        agent (or the same one later) can pick up without reconstructing
                                        a chat transcript; works against a landed or abandoned change too
change handoff <changeId> --add <kind> -m <body>
                                        append one entry (never updates/deletes an existing one);
                                        kind: note | status | question | decision | blocker | plan
change deps <changeId>                 list a change's declared read-only external dependencies
                                        (issue #277); with no flag, lists current state
change deps <changeId> --set kind:owner/repo#ref [--set ...]
                                        replace-all declaration of external dependency edges
                                        (kind is issue|pull_request|release|bookmark); never itself
                                        reads GitHub and never gates landing (advisory only)
change deps <changeId> --refresh       best-effort live GitHub read of every declared dependency's
                                        current state (open/closed, released/draft, exists/missing)
change evolution <changeId>            auditable logical-change timeline (issue #275): kind, related
                                        change ids, before/after evidence, remote recovery limits
change recover <changeId> --keep left|right|both
                                        resolve a recorded divergence (issue #275) between two live
                                        changeIds that both claim the same underlying jj change id;
                                        left/right abandons the other side, both marks it resolved
                                        without abandoning either; refuses (409) if either is landed

stack create <title> [--id x] [--changes CHG-1,CHG-2]
stack list
stack land <stackId> [--force]         same check-run gate as change land
stack enqueue <stackId>                queue a LINEAR stack root->tip via change enqueue's own path;
                                        refuses (409) before any GitHub call if the stack isn't linear
                                        (a member with more than one parent, an unprojected member, or
                                        a base that doesn't chain onto the previous member); prints each
                                        member's own outcome (queued/skipped/failed) rather than an
                                        all-or-nothing result
stack delete <stackId>                 disband — members become loose changes (undoable)
stack reorder <stackId> <changeId,changeId,...>
stack checks <stackId>                 every member's GitHub check runs, one batched round trip
stack issues <stackId>                 linked issue and external-blocker summary for a stack
stack review-health <stackId>          per-member review-health summary, one batched round trip
stack landability <stackId>            per-member effective GitHub landability policy report plus
                                        the overall verdict, one batched round trip; a member blocked
                                        behind an earlier, not-yet-landable member carries its own
                                        "stack-order" requirement

project rules list                     list this repo's Projects v2 lifecycle->field mapping rules
project rules set --project <projectNodeId> --field <fieldNodeId> --option <optionId>
                  --when <draft|in_review|queued|landed|abandoned|blocked> [--id <ruleId>] [--disabled]
project rules delete <ruleId>
project preview <changeId>             preview what "project apply" would do right now, without writing
project apply <changeId>               evaluate and write matching rules now; automatic dispatch already
                                        runs this on land

bookmark list
bookmark delete <name> [--force]       --force required for the default (trunk) bookmark
bookmark set <name> <tracked|view> [--target CHG-x]

workspace list                         live local jj workspace agents reporting into this repo
                                        (see "jjhub daemon"), with their last-seen topology observation
workspace status                       agents, which is primary, and any active lease holder
workspace primary <workspaceId>        mark a workspace as this repository's primary
workspace route [<workspaceId>|clear]  show/set/clear the suggested local-write target — a SUGGESTION
                                        only, never applied implicitly
workspace connect [aliasOrPath]        start a local daemon for a workspace (= "daemon start")
workspace disconnect [aliasOrPath]     stop a local daemon for a workspace (= "daemon stop")

view save <name> <revsetExpression>    save a named jj revset expression for later reuse (storage
                                        only — evaluate it with "jjhub revset")
view list                              list saved views in this repo
view delete <name>

op log                                 "@" marks the current head operation
op diff <fromOpId> <toOpId>            structural diff between two operations' immutable snapshots
                                        (which changes/stacks/bookmarks/conflicts were created,
                                        modified, or removed); works for adjacent or far-apart ops
undo [--force]                         refused when the head op is a "land" (real merge can't be unwound); --force reverts local state anyway
redo

conflict list                          ACTIONS column: the states each conflict can still take;
                                        "historical" = its change already landed, nothing to resolve
conflict resolve <conflictId> <state>
    state: unresolved | left | right | edited | preserve_unresolved
    ("superseded" is set by JJHub itself when the change lands; it cannot be chosen)

sync                                   push overlay state to GitHub

auth login [--url <baseUrl>] [--scope read|write] [--repos o/a,o/b]
                                        sign in with GitHub via the device flow (saves an
                                        OAuth token to ~/.config/jjhub). --scope read mints
                                        a read-only token; --repos fences it to specific
                                        repositories. --api-key <key> instead saves the
                                        legacy ops shared secret
auth sessions                          list your live sessions; auth revoke <id> kills one
auth logout                            forget the saved connection/token
auth status                            who the saved credential authenticates as

ai models --provider <p> [--api-key <key>] [--base-url <url>]
                                        live model list for one AI provider (issue #472); the key
                                        is used for this one request only and never saved. Without
                                        --api-key, falls back to your already-saved AI connection's
                                        key for that same provider, if any. Always succeeds — a
                                        live-lookup failure shows source: static_fallback with a
                                        detail message rather than an error, since a free-text model
                                        id always works when configuring AI, listed or not
ai status                               where AI is coming from for you: your own saved connection,
                                        the server operator's default, or not configured at all

mcp serve                              MCP server over stdio (132 tools) for agent clients
mcp serve --http <port>                the same tools over Streamable HTTP (POST /)
acp serve                              Agent Client Protocol v1 agent over stdio (Zed's agent
                                       panel, ...) — modes (read-only/ask/auto), a working-
                                       repository + dry-run config option, persistent sessions,
                                       plans and diffs — every operation as a slash command
agents-md                              print an AGENTS.md section teaching coding agents to
                                       prefer jjhub over raw git/gh (jjhub agents-md >> AGENTS.md)

telemetry flows [--since <ISO8601>] [--surface webmcp|cli|browser|mcp|acp|a2a|api] [--limit N]
    ongoing telemetry for the read -> patch -> verify -> land -> disconnect -> recover flow
    across every client surface — per-step p50/p95 duration and error rate

daemon <localRepoPath> [--once] [--interval 5] [--engine auto|subprocess|isomorphic] [--two-way]
    sync a local jj working copy into JJHub (inbound; --two-way mirrors JJHub edits back)
daemon start [alias|localRepoPath] [--interval 5] [--engine ...] [--two-way]
    spawn the same sync as a detached background process
daemon stop [alias] | daemon status [alias]

topology <localRepoPath> [--engine subprocess]
    inspect explicit local remote/bookmark topology; never selects or rewrites a push route

revset <localRepoPath> <expression> [--engine subprocess]
    evaluate the installed jj binary's revset in exactly the named workspace;
    this is local, read-only discovery rather than a server-side approximation
revset <expression> --workspace <workspaceId> [--repo owner/name] [--limit N]
    REMOTE form (issue #274): relays the query through the JJHub server to
    that exact workspace's own "jjhub daemon" and polls for the result — use
    when you're not on the machine with the workspace checked out. Slower
    (bound by the daemon's sync interval) and needs that daemon healthy;
    same result shape/error taxonomy as the local form above

content preview <localRepoPath> <split|absorb|squash|duplicate|restore>
    --source <jj-change-id> [--destination <jj-change-id>] [--path <relative-path>]...
content apply <localRepoPath> <split|absorb|squash|duplicate|restore>
    same selection plus --expected-operation <jj-operation-id> returned by preview
    local-only jj content curation; checked against jj's immutable policy and
    never implemented as a browser patch transform; records a durable receipt
    on the JJHub server (best-effort — see "content receipts" below)
content preview|apply --workspace <workspaceId> <split|absorb|squash|duplicate|restore>
    --op <json> | (--source <jj-change-id> [--destination <jj-change-id>]
    [--path <relative-path>]... [--title <title>]) [--expected-operation <id>]
    REMOTE form (issue #276): relays the preview/apply through the JJHub server to
    that exact workspace's own "jjhub daemon" and polls for the result — use
    when you're not on the machine with the workspace checked out. jj's own
    immutable-commits check and JJHub's landed-change guard both apply; a
    successful apply also appends a durable content_op_receipts entry
content receipts [--repo owner/name|repoId]
    list this repo's content-op receipt trail: every content apply invocation,
    successful or rejected, with its kind, revisions, and outcome
```

Anything else (`pr`, `issue`, ...) is passed through to `gh` — e.g. `jjhub pr view 7`. `jjhub auth`
is jjhub's own command (it manages the *JJHub server* connection); use `gh auth` for GitHub auth.

### The other agent surfaces: ACP and A2A

`jjhub acp serve` and the server's A2A endpoint are not CLI subcommands (A2A has no CLI
entry point at all — it's always part of the running server), but they run the exact same
verbs as the table above, through a shared command registry
(`src/agents/commands.ts`, 127 commands: every operation, the hand-written extras `whoami`,
`land_change`, `land_stack`, `get_land_job`, `delete_bookmark`, `sync_github`,
`raise_conflict`, plus registry-only composites `stack_status`, `get_conflict`,
`get_landability`) — so anything documented above as a CLI command is also an ACP slash
command and an A2A skill under the same snake_case name.

- **ACP** (`jjhub acp serve`, stdio, Agent Client Protocol v1): sessions have modes
  (`read-only` refuses mutations; `ask`, the default, confirms force-land/trunk-deletion/
  force-undo through `session/request_permission` or `elicitation/create`; `auto`
  auto-approves everything else), a `repository` config option so `/get_change CHG-3` works
  without repeating the repository id, a `dry_run` config option, and persistent sessions
  (`session/load`/`list`/`resume`/`close`/`delete`, stored in
  `~/.config/jjhub/acp-sessions.json` — `JJHUB_ACP_SESSIONS_PATH` overrides the file,
  `JJHUB_CONFIG_DIR` moves the whole config dir). A dev server with no `JJHUB_API_KEY` needs
  `JJHUB_ACP_ALLOW_ANONYMOUS=1` on the agent process. Full Zed setup and everything the
  session exposes: [`docs/guides/COMMON-TASKS.md`](../../docs/guides/COMMON-TASKS.md#drive-jjhub-from-an-editor-agent-panel-acp-eg-zed).
- **A2A** (spec v1.0 only — the v0.3 compatibility layer was removed; no v0.3 clients):
  agent card at `/.well-known/agent-card.json` (optionally signed via
  `JJHUB_A2A_CARD_SIGNING_JWK`, verifiable against `/.well-known/a2a-jwks.json`), JSON-RPC at
  `POST /a2a`, REST at `/a2a/v1` — one task store shared by both bindings, so a task started
  on one can be fetched/listed/cancelled on the other. Call a command as a text part
  (`/list_changes <repositoryId> status=in_review`) or a data part
  (`{"command": "commit_files", "args": {...}}`); confirmable mutations pause the task in
  `input-required` (a `confirm` message resumes it), plans stream as `working` updates, and
  `CancelTask` can interrupt an in-flight skill at its next safe checkpoint. Full detail and
  the `npm run a2a:exercise` / `npm run acp:exercise` client harnesses (including the shared
  `--mutate --stress N --land` scratch-repository stress scenario):
  [`docs/guides/COMMON-TASKS.md`](../../docs/guides/COMMON-TASKS.md#drive-jjhub-agent-to-agent-a2a).

### Server connection: flags, env vars, or a config file

Precedence is `--url` > `JJHUB_URL` > `~/.config/jjhub/config.json` for the server, and
`JJHUB_TOKEN` (bearer token from `jjhub auth login`) > `JJHUB_API_KEY` (operator key, sent as
`x-api-key`) > the saved config (token, then apiKey) for the credential; the config file is
written by `jjhub auth login`. Run `jjhub auth login --url http://localhost:3000 --api-key <key>`
once and every subsequent command in that shell (any shell, any day) picks it up with no env
vars needed. `jjhub auth status` confirms what's currently in effect.

### Identity & permissions (issue #43: GitHub-federated)

A JJHub user is a GitHub account. `jjhub auth login` runs an OAuth device flow against the
JJHub server's own authorization server: open the printed URL, sign in with GitHub, confirm
the code, and the CLI saves a JJHub-issued OAuth token. Every mutation is attributed to that
identity (visible in `jjhub op log`).

What you can do on a repo is asked of GitHub live at request time (cached ~60s):
GitHub `admin` → owner, `maintain`/`write` → writer, `triage`/`read` → reader. There is no
internal membership table and nothing to manage in JJHub — manage collaborators on GitHub.

The legacy shared secret (`--api-key` / `JJHUB_API_KEY`) still exists as an ops/emergency
credential: full access, RBAC bypassed, no attribution. Anonymous requests (no server-side
`JJHUB_API_KEY` configured at all) remain fully open — dev mode, unchanged.

### GitHub admin verification (optional, separate mode)

This RBAC is entirely JJHub-internal — a JJHub "owner" isn't checked against real GitHub
permissions at all. If the server has `JJHUB_REQUIRE_GITHUB_VERIFICATION=1` set,
`jjhub repo init` requires `--github-token <pat>` (or falls back to `$GITHUB_TOKEN`): the
server verifies that token has real `admin` access to the exact `owner/name` being created
via a live GitHub API call, rejecting creation (403) otherwise. The verified token also
becomes the new repo's GitHub connection automatically. See the top-level README's "GitHub
admin verification (optional mode)" section.

For exact, always-current syntax straight from the CLI binary, run `jjhub help` — this file is
an expanded, example-rich companion to that output, not a replacement for it.

## Install this skill

Three independent ways to pick this skill up — use whichever fits the agent you're
configuring. `GET /.well-known/skills/` on any running JJHub server (including the live
instance) returns all three as an `installation` object, e.g.
`curl https://jjhub.erisera.com/.well-known/skills/ | jq .installation`.

### (a) Teach a repo's coding agents to prefer jjhub (AGENTS.md)

[`agents-snippet.md`](./agents-snippet.md) is a short, imperative AGENTS.md
section — "version control in this repo goes through JJHub; prefer `jjhub
change/stack/undo` over raw git/gh mutations" — the same pattern as GitHub's
`gh skill install github/gh-stack`. Append it to a tracked repo's own
AGENTS.md so any AGENTS.md-aware coding agent (Claude Code, Copilot, Cursor,
Codex, ...) picks up the workflow automatically:

```bash
npx jjhub@latest agents-md >> AGENTS.md
```

### (b) Fetch this skill directly (`skills.sh`, or a plain curl/`.well-known` fetch)

The skill content itself (this file plus `agents-snippet.md`) is mirrored, read-only, to the
public repo [`erisera-code/jjhub-skills`](https://github.com/erisera-code/jjhub-skills) on
every change (`.github/workflows/publish-skills.yml`, built by
`scripts/build-skills-dist.ts` — `npm run build:skills` reproduces it locally). Install from
there with the [skills.sh](https://skills.sh) installer:

```bash
npx skills add erisera-code/jjhub-skills
```

Or mirror it by hand into a local `skills/` directory — the same shape either the live
server's `GET /.well-known/skills/` or the mirror's own `index.json` returns:

```json
{
  "skills": [
    { "name": "jjhub-cli", "url": "/.well-known/skills/jjhub-cli/SKILL.md", "agentsSnippet": "/.well-known/skills/jjhub-cli/agents-snippet.md" }
  ],
  "installation": {
    "skillsCli": "npx skills add erisera-code/jjhub-skills",
    "claudeCodeMarketplace": "/plugin marketplace add erisera-code/jjhub-skills",
    "agentsMd": "npx jjhub@latest agents-md >> AGENTS.md",
    "wellKnown": "https://jjhub.erisera.com/.well-known/skills/",
    "repository": "https://github.com/erisera-code/jjhub-skills"
  }
}
```

```bash
mkdir -p skills/jjhub-cli
curl -s https://jjhub.erisera.com/.well-known/skills/jjhub-cli/SKILL.md -o skills/jjhub-cli/SKILL.md
curl -s https://jjhub.erisera.com/.well-known/skills/jjhub-cli/agents-snippet.md -o skills/jjhub-cli/agents-snippet.md
```

`erisera-code/jjhub-skills` also carries a `.claude-plugin/marketplace.json`, so Claude Code
can add it directly as a plugin marketplace:

```
/plugin marketplace add erisera-code/jjhub-skills
/plugin install jjhub-cli@jjhub-skills
```

### (c) Wire up an MCP/ACP client, or an A2A peer

- **Claude Code** (MCP): `claude mcp add jjhub --http https://jjhub.erisera.com/mcp`
  against a deployment (OAuth 2.1 discovered automatically via RFC 9728 — the CLI walks you
  through sign-in), or point at a local stdio server:
  ```json
  { "mcpServers": { "jjhub": { "command": "npx", "args": ["jjhub@latest", "mcp", "serve"] } } }
  ```
- **Zed** (ACP), `~/.config/zed/settings.json`:
  ```json
  {
    "agent_servers": {
      "jjhub": {
        "command": "npx",
        "args": ["jjhub@latest", "acp", "serve"],
        "env": { "JJHUB_URL": "https://jjhub.erisera.com", "JJHUB_TOKEN": "<token from jjhub auth login>" }
      }
    }
  }
  ```
  (Running `jjhub auth login` on the same machine first means the `env` block can be
  omitted entirely — the agent picks up the saved credential from `~/.config/jjhub`.)
- **A Claude.ai or ChatGPT connector** (remote MCP): add a custom connector pointing at
  `https://<your-jjhub>/mcp` — both discover the OAuth 2.1 flow automatically (RFC 9728
  protected-resource metadata); no manual token entry.
- **An A2A peer**: fetch `https://<your-jjhub>/.well-known/agent-card.json` for the skill
  list and the two binding URLs (JSON-RPC `/a2a`, REST `/a2a/v1`); see "The other agent
  surfaces: ACP and A2A" above for the auth model and confirmation flow.

## Example workflows

### Create and land a change

```bash
npm run jjhub -- repo init owner/name
npm run jjhub -- change new "Fix: add user validation"
npm run jjhub -- change list                              # note the CHG-n id
npm run jjhub -- stack create "Fix bundle" --changes CHG-1
npm run jjhub -- stack land "Fix bundle"
```

A single change not part of a stack lands directly: `npm run jjhub -- change land CHG-1`.

### Use GitHub's merge queue instead of a direct land

When the target branch requires it, add a review-ready change to GitHub's
merge queue rather than merging it directly — GitHub owns the merge from
there, and JJHub only marks the change landed once GitHub reports it merged:

```bash
npm run jjhub -- change request-review CHG-1
npm run jjhub -- change enqueue CHG-1
npm run jjhub -- change queue-status CHG-1   # position/state/merge-group evidence,
                                              # removal reason if GitHub removed it,
                                              # and whether GitHub auto-merge is armed
npm run jjhub -- change dequeue CHG-1        # withdraw back to explicit review
```

A change JJHub knows is queued cannot be rewritten (`describe`/`amend`/
`restack`/...) or landed directly until it is dequeued — `change land`
refuses with a message naming the queue.

### Sync a local jj working copy into JJHub

```bash
cd /path/to/local/jj/repo
jj bookmark track main --remote=origin   # REQUIRED once per clone (issue #28):
                                         # without it, already-landed commits look
                                         # like new local work and the daemon could
                                         # try to re-project them
npm run jjhub -- repo init alice/my-project
jj new -m "Implement feature X" && jj commit
npm run jjhub -- daemon . --once      # one-shot sync; omit --once to loop every 5s
npm run jjhub -- op log               # verify the sync landed as a new operation
```

### Discover local jj changes with a revset

```bash
npm run jjhub -- revset . 'mutable() & descendants(@-)'
npm run jjhub -- revset . 'bookmarks() | remote_bookmarks()' --json
```

The expression is passed to the installed `jj` binary as one argument, never
through a shell. The workspace path is explicit because `@`, aliases, and
immutable boundaries are facts about that specific local workspace. This first
read-only form does not send local commits or filesystem paths to JJHub.

### Curate local change content

```bash
npm run jjhub -- content preview . split \
  --source <jj-change-id> --title "Extract validation" --path src/checkout.ts
npm run jjhub -- content apply . split \
  --source <jj-change-id> --title "Extract validation" --path src/checkout.ts \
  --expected-operation <operation-id-from-preview>
```

The non-interactive local surface delegates `split`, `absorb`, `squash`,
`duplicate`, and `restore` to the installed jj binary with structured argument
arrays. A preview's jj operation id is mandatory for apply, and immutable
targets are refused without `--ignore-immutable`. `--path` selects complete
relative paths; interactive hunk selection remains jj's configured diff editor.
Run `jjhub daemon . --once` afterwards to reconcile the local graph.

Each `content apply` invocation — successful or rejected (e.g. a stale
`--expected-operation`) — records a durable receipt on the JJHub server
(best-effort: a missing repo context or unreachable server only produces a
warning, never blocks the local jj mutation). Review the trail with:

```bash
npm run jjhub -- content receipts
```

### Resolve a conflict

```bash
npm run jjhub -- conflict list                              # note the conflict id
npm run jjhub -- conflict resolve <conflict-id> left         # or: right | edited | preserve_unresolved
npm run jjhub -- conflict list                               # confirm it now reads Resolved
```

## Undo/redo caution

`jjhub undo` and `jjhub redo` move the repository's operation-log head pointer — they affect
*all* state (changes, stacks, bookmarks, conflicts), not just the last command's target.
Run `jjhub op log` first if you're not sure what the current head operation is before undoing.
