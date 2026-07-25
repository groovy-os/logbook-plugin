---
name: add
description: Turns a rough idea, or an issue discovered mid-work, into a meaty self-contained build brief on the logbook board — problem, size/risk tier, acceptance criteria, affected modules and files, test plan, doc-update list. Use whenever work is about to be filed: any time create_item, create_items, or update_item is about to be called; the moment a bug, gap, or regression surfaces while running tests, reviewing a PR, or coding something else (capture it before the context is lost); or when the user says "log this", "add it to the logbook", "file an item", "put that in the backlog", "make a log for this", or "/logbook:add". Also use to flesh out an existing thin item before it is built. Fires on its own from the surrounding task — it does not require the slash command.
argument-hint: "<rough idea / bug / problem statement>" | LOG-NNN (to flesh out an existing thin item)
---

# Authoring logbook items that build well

A logbook item is the prompt the entire `logbook:loop` gauntlet builds from. **Garbage in, garbage out** — a one-liner produces a guessing agent; a complete brief produces a clean PR on the first pass. Your job here is to do the *thinking* a human would otherwise have to do mid-build, up front, so the builder and reviewer never have to guess.

## When to use this

- The user hands you a rough idea and wants it logged.
- **You discover a bug/gap/regression mid-work** — running tests, reviewing a PR, or coding something else — and want to capture it. This is the highest-value moment: you have the repro, the stack trace, and the exact failing surface *right now*. Log it before that context evaporates. This fires automatically (you don't need anyone to type `/logbook:add`).
- An existing item is thin and you're about to send it through `logbook:loop` — flesh it out first.

## Triggered automatically vs. via the command

This skill is the engine; `/logbook:add` is just a thin front door that calls it. The skill activates **on its own** whenever the surrounding task involves adding to the logbook — e.g. an agent driving a test suite that finds a 500 should invoke this to file the item, no command required. The command exists only for when *you* want to start an authoring pass deliberately.

## Capturing a discovery without derailing

When the trigger is a discovery mid-work (not a deliberate authoring pass), you're in the middle of something else — don't rabbit-hole. But DO capture the context that's only available right now, because it's pure gold for the eventual builder:

- **Exact repro** — the request/command that triggered it, the input, the env.
- **Observed vs. expected** — the actual error/stack trace/status code vs. what should have happened.
- **Where it broke** — the endpoint/module/file:line if you saw it in the trace.

Drop those into the template's Problem + Context + Test plan sections. A bug captured *with* its repro at discovery time is often a more complete brief than one written cold later. Then return to what you were doing. If you genuinely can't tell the size/risk tier from where you are, mark it provisional (`Tier: ? — discovered during <x>, needs triage`) rather than stalling.

## The rule

**Never file a one- or two-line item that will be built by an agent.** If the item is a throwaway reminder for a human, a thin note is fine. If an agent will build from it, it MUST carry every section in `references/item-template.md`. The cost of writing the brief is paid back many times over by not having a build/review loop thrash on missing context.

## Process

1. **Check for an existing item FIRST (deduplicate before you write).** This is the highest-leverage step in the auto-trigger path — the same failure recurs across runs and across sessions, and a backlog full of duplicate "X is broken" items is worse than useless. Before authoring anything, search the logbook by the issue's distinctive signature — the module name, the endpoint, the error/symptom, a logbook ref if one is mentioned. Use `pull_items(search: "...")` and/or `list_backlog` (filter by `label` and `status`); search **open AND recently-closed** items, not just the open backlog. Then decide:
   - **Open duplicate exists** → do NOT create a second item. If you have new context (a fresh repro, a second affected surface, a clearer trace), `add_note` it onto the existing item and stop. Tell the user you found and enriched LOG-NNN instead of filing a dup.
   - **Closed/done item covers it, but the bug is back** → this is a regression. Create a new item, but link it to the old one with `link_items` (and reference the ref in Context) and say "regression of LOG-NNN" — a reopened-bug history is signal, a silent re-file is noise.
   - **Near-miss (related but distinct)** → file it, but cross-reference the related item in Context so the connection isn't lost.
   - **Genuinely new** → proceed.
   When filing a batch with `create_items`, dedup each entry against the backlog and against the others in the same batch.
2. **Interrogate the idea before writing.** Read the relevant code first — don't author from imagination. Find the module(s) involved, the existing patterns, the tests that already exist. A spec that names the wrong file is worse than no file reference. Use Grep/Glob/Read or dispatch an `Explore` agent for breadth.
3. **Tag from the live taxonomy — reuse, almost never create.** Tagging is how the whole system finds work: `logbook:loop --tag X` keys off labels, so a mis-tagged or untagged item is effectively invisible to the gauntlet. Get this right.
   - **Always call `list_labels` first** — it returns the managed taxonomy with item counts. That live set is the source of truth; do not tag from memory (labels evolve, and this skill deliberately carries no hard-coded list that would go stale against your board).
   - **Tag across the dimensions the taxonomy uses**, picking the existing label that fits each that applies — typically 2–4 labels total. Most boards end up with roughly these axes; read the actual label names off `list_labels` rather than inventing them:
     - **Layer** — which side of the stack the work lands on.
     - **Domain/area** — the module, feature area, or subsystem.
     - **Nature** — defect vs. hardening vs. feature vs. chore.
   - **Strong bias against new labels.** `create_label` is a *taxonomy change*, not a tagging convenience — a backlog with 40 near-synonym labels is unsearchable. Watch the counts `list_labels` returns: a label sitting at count 1 is the sprawl to avoid. If no existing label is a clean fit, pick the **closest existing one** rather than minting a variant. Only when a genuinely new *category* is needed (not a synonym of an existing one) do you `create_label` — and on the autonomous path, **propose the new label to the user and get a yes before creating it** rather than spawning it unsupervised. An agent inventing labels is the same failure as filing duplicate items: taxonomy noise that makes the board lie.
4. **Fill the template** in `references/item-template.md`. Every section earns its place — see that file for what each one prevents.
5. **Write it via MCP.** `create_item` with `problem`, `title`, `size`, `summary`, `description` (the filled template as markdown), `priority`, `tasks: [...]` (the acceptance criteria as a checklist), and `labels`. For several at once use `create_items` (all-or-nothing validation — fix and retry safely).
   - **`size` is required** — the call fails validation without it. It is one of `S` / `M` / `L` / `XL`. Use the same judgement as the template's `Size / risk tier` line, so the tier the item is *filed* with is the tier `logbook:loop` reads back when it enforces batch caps. An item whose size and tier line disagree will get batched wrongly.
   - **`summary` is one line, max 120 chars, imperative.** It is what every index row shows, so it is what a planner or a teammate scanning fifty items actually reads — the title alone is rarely enough to decide whether an item belongs in a batch. Optional to the API, effectively mandatory in practice.

   A minimal well-formed call looks like:

   ```
   create_item(
     title: "404 on cross-account subject lookup",
     summary: "Return 404 instead of 403 when a subject id belongs to another account",
     size: "S",
     problem: "<the sharp problem statement>",
     description: "<the filled template, markdown>",
     priority: "medium",
     tasks: ["Returns 404 for a foreign subject id", "Regression test added"],
     labels: ["backend", "bug"]
   )
   ```
6. **Set `assignee_emails` only when the owner is obvious.** Use `list_team` if you need to resolve a name to an email. Most items going through the gauntlet are unassigned until an agent claims them.

## What makes an item "meaty" (the high-leverage sections)

Read `references/item-template.md` for the full template. The sections that most change build quality:

- **Problem** — the load-bearing field. The first principle of a good brief is *"what problem are you trying to solve?"* Answer it crisply: a sharp problem statement is what the build's completion note restates and proves it solved. Vague problem in → unverifiable outcome out.
- **Size / risk tier (S/M/L/XL)** — declare it. `logbook:loop` reads this to enforce per-tier batch caps and to refuse XL (major-overhaul) items, steering those to engineer-in-the-loop. An honest tier is what keeps an unattended batch from biting off more than it can finish.
- **Affected modules + files** — turns a search problem into a lookup, **and is the signal `logbook:plan` clusters on**. Name the directory and the specific files you expect to change, using this repo's actual layout — read it, don't assume one. Be precise: triage fuses items whose footprints overlap into one build/PR to pre-empt the merge cascade, so a vague or missing footprint means two items that *will* collide get built as standalones that fight at merge time. Flag explicitly if the item adds a schema migration or touches a high-collision shared file (a central model/schema file, a route or event registry, a top-level architecture doc).
- **Test plan** — name the scoped tests to run in the inner loop AND which new test(s) to add. Derive the command rather than guessing: use the configured `testCommand` plugin setting if it is set; if it is blank, detect it from the repo (the `scripts` block of the package manifest, the tool config in `pyproject.toml`/`Makefile`/`justfile`, or the job steps in the CI workflow) and prefer the scoped form of it. Pre-stating this means the builder doesn't have to infer coverage.
- **Authorization / isolation considerations** — if this project documents an isolation, tenancy, or authorization gate in its own agent instructions (`CLAUDE.md` / `AGENTS.md`) or contributor docs, answer it explicitly here and cite the specific rule. If the project has no such documented gate, write "N/A" rather than inventing one — guessed security rules are worse than none.
- **Doc-update list** — which docs must change in the same PR. Check the project's own docs policy (its agent instructions file usually names the hard gates) instead of assuming a fixed set.

## Anti-patterns

- **Filing without searching first** → duplicate items. The auto-trigger path makes this the most likely failure mode (same test failure, many runs). Always dedup first; enrich an existing item rather than spawning a twin.
- Authoring from memory without reading the code → wrong file refs, wrong patterns.
- Acceptance criteria that aren't checkable ("make it better") → the reviewer can't render a verdict.
- Omitting the test plan because "the builder will figure it out" → it will, slowly, after a review bounce.
- Importing conventions from another codebase — file layouts, test commands, migration tools, review gates differ per repo. Read this repo's agent instructions and manifests; if a convention isn't documented here, don't assert it in the brief.

## Related skills

`logbook:explore` (is this problem wider than one item?), `logbook:plan` (batching), `logbook:build` / `logbook:review` / `logbook:merge` (the build gauntlet), `logbook:loop` (drives the whole loop), `logbook:sweep` (backlog hygiene), `logbook:ux` (UX specs).
