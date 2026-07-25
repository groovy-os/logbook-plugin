---
name: plan
description: Plans the optimal build path through the open logbook backlog — clusters open items (assigned or unassigned) into logical batches by file-collision, module affinity, and dependency order ("if you're doing LOG-123, do 122 and 124 in the same pass"), sequences the batches, and emits paste-ready /logbook:loop commands. Runs as a director/worker system with model tiering and single-writer discipline. Use when the user says "plan the backlog", "what should we batch", "what order should these be built", "come up with batches for logbook:loop", "which of these collide", or runs /logbook:plan.
argument-hint: "[--tag <label> | --mine | --all] [--horizon today|week]"
---

# Planning the logbook: batch composition for the loop

`/logbook:loop` executes batches well but doesn't choose them — and batches chosen badly (colliding items built as standalones, dependents built before their foundations) are what produces a merge cascade at the end of a big run. `logbook:plan` is the upstream brain: it looks at the whole open backlog and answers *"what should be built together, and in what order?"* — producing batches you can hand straight to `/logbook:loop`.

Run it on a **clean** backlog — ideally right after `/logbook:sweep`. If you hit obviously-stale items mid-plan, don't silently plan around them: flag them and suggest a sweep pass.

## Architecture: director + tiered workers

**You are the DIRECTOR and the single writer to logbook** — workers research and return structured data; only you write (`link_items`, `add_note`, `update_item`). This prevents duplicate notes and racing writes when many workers run in parallel.

Route each role to the cheapest model that fits (the Agent tool's `model:` option). The tiers are **relative capability**, not product names — map them onto whatever model classes your harness exposes, and collapse tiers if it exposes only one.

| Role | Model class | Work |
|---|---|---|
| **Extractor** | a fast, cheap model — mechanical retrieval, no judgement | Per item, extraction from the spec payload the worker pulls itself: affected files/modules, size/risk tier line, labels, explicit dependency mentions ("blocked by", "after LOG-NNN", `related_items`), migration-likely?, front-end?, does it carry a UX spec in `documents`? Batch ~10 items per worker. |
| **Footprint verifier** | a mid-tier coding model — reads code, confirms claims | Verify each item's file footprint against the current tree: do the cited files exist, expand a module/package label into its real touch-set (the sibling files that make up a unit of that kind in this repo, plus its tests), flag any footprint that lands on a **collision hotspot** (Step 0). |
| **Graph builder** | a frontier model — cross-item judgement, whole set in one context | Build the **collision graph** (shared files → fuse or flag) and the **dependency graph** (schema before consumer, endpoint before UI surface, foundation/keystone before leaves, epic children ordered per the epic's notes). Propose clusters + an ordering with reasons. |
| **Director** | you, the orchestrator | Final batch composition and sequencing, tier-cap enforcement, the handoff commands, the report, all logbook writes. |

For a large backlog (> ~15 items), pipeline extraction → verification with whatever workflow/pipelining tool your harness offers; otherwise dispatch parallel `Agent` calls in a single message. The graph pass wants the whole set in one context, so it runs after a barrier.

## Step 0 — Discover the repo (once, cheap)

Nothing below is hardcoded to a stack. Derive it, and only fall back to plugin config when discovery fails.

```bash
# Base branch — what feature branches cut from and PRs target
git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null | sed 's|^origin/||'
```
If that is unset (a common state on fresh clones), use the plugin's `baseBranch` setting (default `main`). Say which one you used in the report.

```bash
# Collision hotspots — DERIVED, not a hardcoded file list.
# The files the most commits touch are this repo's "everything edits it" files.
git log --since=6.months --name-only --pretty=format: \
  | grep -v '^$' | sort | uniq -c | sort -rn | head -30
```
Drop lockfiles, generated artefacts, and changelogs from the result; what remains is the hotspot set — the registry / models / barrel-export / schema files of *this* codebase. To test a specific pair, ask which files travel with a given file: `git log --since=6.months --name-only --pretty=format:'%H' -- <file>`.

Two items whose footprints intersect **only** on a hotspot get a *conflict edge*, not a fuse (Step 4) — hotspots are shared by everything, so fusing on them serializes unrelated work.

```bash
# Versioned migrations — detect the tool, don't assume one
git ls-files | grep -Ei '(^|/)(migrations?|alembic|db/migrate|prisma/migrations|supabase/migrations|liquibase|flyway)/' | head
```
If the project uses **sequential/versioned migrations where each revision names its predecessor** (Alembic `down_revision`, Django `dependencies`, Rails/Prisma/Flyway ordinals, golang-migrate pairs), then every item that adds a migration off the current head is a future multi-head. Flag them as a migration set for `/logbook:merge` — it linearizes the chain so the batch produces **one head**.

## Step 1 — Scope the backlog (director) — index rows only

Retrieve in widening steps and stop at the narrowest one that answers the question. The tiered workers exist so that *you* never have to hold what they can hold for you — pulling every open item's full payload here spends the entire budget the tiering was meant to save, before a single worker is dispatched.

1. **`backlog_stats` first.** The shape of the whole backlog in one small object — counts by status, size and label, per-assignee open counts, the `unsized` set, the oldest in-progress item. This tells you how big the planning problem is before you spend anything on it.
2. **`list_backlog` at index verbosity** across the open statuses (backlog / in_progress / stuck), assigned **and** unassigned. One line per item — ref, title, summary, size, status, assignees, task progress, labels — which is enough to scope batches, spot anchors, and shard the workers.
3. **Do not call `pull_items` in your own context.** Specs are what the extractors and verifiers in Steps 2–3 are for. Hand each worker the refs in its slice; it calls `pull_items` in *its* context and returns structured findings. You receive footprints and verdicts, never bodies.

Don't claim anything. Items already `in_progress` with a live branch/PR are **anchors** — plan around them, don't re-batch them; the index row carries the status you need to identify them.

Expand a single item yourself (`pull_item`) only to adjudicate a worker disagreement or when a batching decision genuinely turns on something the index doesn't carry — and then only that item.

## Step 2–3 — Extract, then verify footprints

Pipeline the items through extraction and footprint verification (no barrier between the two stages — an item can be verified while others are still extracting). An item with no derivable footprint and no useful spec is a **thinness signal**: set it aside for `logbook:add` fleshing rather than guessing it into a batch.

## Step 4 — Graphs and clusters (frontier judge)

One judge gets every item + verified footprint and returns:

1. **Fuse sets** — items sharing a *primary* file (the same unit's implementation/schema/route files, the same function or class) → must be ONE build unit in `/logbook:loop` (the "if you do LOG-123, do 122 and 124" answer). Same-file-different-functions is a judgement call: fuse when they touch the same function/class, otherwise co-batch.
2. **Conflict edges** — items sharing only a Step 0 hotspot, a shared doc/registry file, or "each adds a migration" → keep separate but must not run in *parallel* batches; carry as a conflict map to `/logbook:merge`.
3. **Dependency edges** — A must merge before B starts (schema → consumer, endpoint → UI surface, keystone → siblings). Cite the evidence for each edge; invented dependencies serialize work for nothing.
4. **Theme affinity** — items in the same module/epic batch naturally even without file overlap (warm context, one review narrative).

## Step 4.5 — Respect existing epics

An epic is a decomposition someone already made. Treat it as evidence, not as noise to re-derive:

- **Never batch the epic itself.** It is a container with no diff; `logbook:loop` refuses it. Batch its children.
- **Its children are a theme with a stated order.** The epic's description and notes usually say what comes first; prefer that ordering over one you infer, and cite it as the evidence for the dependency edge.
- **Do not scatter one epic across many batches** without saying why. Children of one epic share context, and reviewing them together is cheaper. Split only when file collisions or a real dependency force it — and then say which batch carries which children.
- **A part-built epic is an anchor.** If some children are already in progress with live branches, plan the rest around them rather than re-clustering the whole set.

An epic with no children is a planning gap: report it as NOT BATCHED with that reason rather than silently skipping it.

## Step 5 — Compose and sequence the batches (director)

From the graphs, build the plan:

- **Batch = a set of build units that can run as one `/logbook:loop` invocation.** Respect the loop's tier caps (S ≤ 15, M ≤ 10, L ≤ 4–5 per run; re-tier fused units — 3 fused S ≈ 1 M). **XL items are never batched** — they go to the human as engineer-in-the-loop work.
- **Sequence by dependency**: batches containing foundations/keystones first; a batch that adds migrations should note the shared-parent risk for `/logbook:merge`.
- **Gates are pre-steps, not blockers to planning**: a front-end item with no UX spec document gets an explicit `/logbook:ux LOG-NNN` pre-step in its batch (the hard UX gate); a thin item gets a `logbook:add` fleshing pre-step.
- Two batches with a conflict edge between them run **sequentially**, not in parallel sessions.

## Step 6 — Persist and hand off

- `link_items` the members of each fuse set / batch so the relationships survive the session. Reuse existing labels if a shared tag helps (`list_labels` first); don't mint per-batch labels — **explicit refs are the primary handoff**.
- `add_note` on each batch's keystone item: which batch it's in, what precedes it, why.
- Emit the plan report:

```
BATCH 1 — <theme> (run first: contains the keystone)
  pre-steps: /logbook:ux LOG-431
  ▶ /logbook:loop LOG-122 LOG-123 LOG-124 --mode sequential
  fused: {122,123} (both edit the orders service); 124 co-batched (same module)
  tier mix: 2 S + 1 M · conflicts with batch 3 via <hotspot file> — do not run in parallel

BATCH 2 — ... (independent of batch 1 — can run in a parallel session)
  ...

NOT BATCHED
  LOG-88 (XL — engineer-in-the-loop) · LOG-91 (thin brief — needs logbook:add) · ...
```

Every open item appears exactly once — in a batch, a pre-step, or NOT BATCHED with a reason. **No silent drops and no silent caps**: if a tier cap pushed items out of a batch, name every deferred item and why. Close by offering to kick off Batch 1.

## Guardrails

- **Plan, don't build.** This skill never branches, edits code, or opens PRs — its output is batches for `/logbook:loop`.
- **Single writer** — workers never touch logbook mutation tools.
- **Context economy is a rule, not a preference.** Index rows in the director; full payloads only in worker contexts.
- **Don't over-fuse**: a mega-unit that serializes unrelated work is as costly as a conflict cascade. Fuse on shared *primary* files; conflict-map the rest.
- **Anchors are immovable**: in-progress items with live branches keep their scope; plan around them.
- **Every edge cites evidence** (a file, a spec line, an epic note) — a plan the human can't audit isn't a plan they can approve.
