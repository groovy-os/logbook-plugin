# logbook

The command suite for [logbook](https://www.logbk.dev) — a hosted MCP backlog for agentic
engineering.

This plugin is the **client**. It ships the slash commands and skills that Claude Code uses to
file work, spec it, plan it, build it, review it, and merge it. Your board, your logs, and
their documents live in the hosted logbook service and are reached over MCP — the plugin does
not store anything itself.

---

## Install

```sh
claude plugin marketplace add groovy-os/logbook-plugin
claude plugin install logbook@logbook --scope user
```

Then connect the MCP server for the project whose board you want to drive:

```sh
claude mcp add --transport http logbook \
  https://www.logbk.dev/api/mcp/<project-slug> \
  --header "Authorization: Bearer lgbk_…"
```

The project slug and the `lgbk_…` token both come from **Settings** in the logbook app. The
token is scoped to your organization; the URL is what binds the connection to one project, so
point it at the board you actually want this repo writing to.

> **Restart Claude Code after installing.** Newly installed commands do not load into a
> running session — this is verified behaviour, not a caveat. Quit and relaunch Claude Code,
> then `/logbook:add` and friends will show up. If a command "doesn't exist" right after an
> install or update, restart before debugging anything else.

---

## The commands

| Command | What it does | Reach for it when |
|---|---|---|
| `/logbook:add` | Turns a rough idea or a bug you just hit into a meaty, self-contained build brief on the board. | Anything worth building or fixing needs to be captured. |
| `/logbook:explore` | Investigates a question or a suspected problem across the codebase and files what it finds as logs. | You suspect the issue is wider than one item, or you don't yet know the blast radius. |
| `/logbook:ux` | Runs a front-end log through the UX depth gauntlet and attaches a `ux-spec.md` build contract to it. | A front-end log is endpoint-shaped and would otherwise ship as a thin v0. |
| `/logbook:plan` | Groups ready logs into collision-aware batches by footprint, size, and risk. | You have a pile of ready work and want a sane build order. |
| `/logbook:loop` | The orchestrator: pulls a batch and drives build → review → fix → PR for each log. | You want work actually done, mostly unattended. |
| `/logbook:merge` | Lands finished PRs in dependency order and handles the merge cascade. | Branches are green and stacked up waiting to go in. |
| `/logbook:sweep` | Backlog hygiene — reconciles stale statuses, dedupes, archives what's done. | The board has drifted from reality. |
| `/logbook:jot` | Writes a private note on your notepad. | A thought with no home yet — a hunch, a follow-up. |
| `/logbook:recap` | What shipped recently, with resolutions and pull requests. | Standup, weekly summary, changelog draft. |

`logbook:build` and `logbook:review` are **internal skills the loop dispatches**, not commands
you type. `/logbook:loop` invokes them per log; you never call them directly.

---

## How the loop works

```
file → explore → spec → plan → loop (build → review → fix) → merge
```

You file work with `/logbook:add` (or let `/logbook:explore` file a cluster of it). Front-end
logs pass through `/logbook:ux`, which is a **hard gate** — the loop refuses to build a
front-end log that has no UX spec attached. `/logbook:plan` batches what's ready. `/logbook:loop`
then works the batch: for each log it builds on a branch, reviews the diff with fresh eyes,
fixes what review found, and opens a PR. `/logbook:merge` lands them.

The orchestrator **delegates to subagents** — the build, the review, and the fixes each run in
their own context and report back a summary. That is deliberate: the main session stays small
and coherent across a long batch instead of drowning in file dumps.

---

## Configuration

The plugin declares three user settings:

| Setting | Type | Default | Effect |
|---|---|---|---|
| `baseBranch` | string | `"main"` | The branch feature branches cut from and PRs target. |
| `testCommand` | string | `""` (blank) | The test command for a scoped change. Left blank, the skills detect it from the repo (package manifest scripts, tool config, CI workflow). |
| `webhookWired` | boolean | `true` | Whether a GitHub merge webhook is wired to logbook. When `false`, the loop closes logs itself instead of waiting for a merge webhook to do it — otherwise finished work is never marked done. |

Set them at install time with a repeatable `--config` flag:

```sh
claude plugin install logbook@logbook --scope user \
  --config baseBranch=develop \
  --config testCommand="npm test --" \
  --config webhookWired=false
```

Or change them later from `/plugin`.

---

## Requirements

- A **logbook account** at [logbk.dev](https://www.logbk.dev) — for the project board, the MCP
  endpoint, and an `lgbk_…` token.
- **Claude Code**, with the logbook MCP server connected as shown above.
- The **`gh` CLI**, authenticated — the PR and merge commands shell out to it.

---

## License

Proprietary. See [LICENSE](./LICENSE). Use is permitted only in connection with a logbook
account; no redistribution, derivative works, or resale.
