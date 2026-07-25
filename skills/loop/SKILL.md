---
name: loop
description: Runs a batch of logbook items through the full engineering gauntlet — triage, conflict-clustering, build, fresh-eyes review, a capped fix loop, and a merge-order plan — until every item is a clean, conflict-mapped PR awaiting human review.
argument-hint: LOG-321 LOG-322 ... | --tag <label> --count <n> | --mine | [--status <status>] [--mode sequential|parallel] [--max-rounds 3]
---

# logbook:loop — the development gauntlet

You are the **orchestrator**. Given a set of logbook items, drive them through the full pipeline:

> **triage (cluster by file-overlap) → build → fresh-eyes review → fix → merge-order/conflict map → solve conflicts → ready for human review**

The human's job is product + final GitHub review/merge; yours is everything up to a clean, conflict-mapped, mergeable PR set.

Arguments: `$ARGUMENTS`

**Why the triage step exists (read this first).** Measured the hard way: a 30-PR run where items were built as individual standalones produced a large conflict cascade at merge time — siblings that edited the *same* service/schema/registry/model files (or each added a migration off the same parent) all conflicted, and untangling them took a full extra session of merge-base-in + re-resolve + re-CI cycles. **The fix is at the top of the funnel: items that will touch the same files must be built together as ONE branch/PR/review unit, not N standalones.** Step 1.6 (cluster) and the merge-planning step (delegates to `logbook:merge`) are the structural prevention + cure. Do not skip them.

## Context economy (how this loop stays affordable)

You are running a batch. If you pull every item's full payload into *your* context, you will be out of room before the first build finishes — and the orchestrator is the one agent that must survive to the end.

The pattern, in order:

1. **`backlog_stats` for shape.** Counts by status/label/priority. Cheap. This tells you what the batch even looks like before you name a single ref.
2. **`list_backlog` for scope.** Index rows (ref, title, status, priority, labels, size) — enough to select, rank, and cap. Select the target set from index rows, not from full items.
3. **Dispatch subagents that call `pull_items` in THEIR context** and return *structured findings* — a compact table, not the payloads. The triage pass (Step 1.6), each build, and each review all read their own items. Their token cost dies with them.
4. **Expand one item yourself only when a decision genuinely turns on something the index lacks** — a single `pull_item` / `get_document` when you must adjudicate a tier or a fusion call. One item, not the batch.

**Single-writer rule.** Only the orchestrator writes to logbook. Triage, review, and planning subagents are **read-only against logbook** — they return findings as text and *you* record them (`update_item`, `add_note`, `create_item`). The one sanctioned exception is `logbook:build`, which owns the item it was dispatched for while it holds it (its own `complete_task` / `add_links` / `add_note` / `status=stuck`). Everything else routes through you. Two agents writing the same item is how progress bars start lying.

## Step 0 — Resolve this repo's conventions (do this before anything else)

Nothing below is allowed to assume a stack. Resolve, in this order, and report what you resolved:

- **Base branch** — the branch feature branches cut from and PRs target:
  `git symbolic-ref --short refs/remotes/origin/HEAD` (e.g. `origin/main` → `main`).
  If that fails (no HEAD ref set): try `git remote show origin | sed -n 's/.*HEAD branch: //p'`, then the plugin's `baseBranch` config, then `main`. If the repo has an integration branch (`develop`, `staging`) that PRs actually target, the `baseBranch` config is the override — trust it over the symbolic ref.
- **Test + lint/format/type commands** — the plugin's `testCommand` config if set; otherwise detect from the repo's manifest or CI config, in this order: `package.json` `scripts.test`/`lint`/`typecheck`, `Makefile` targets, `pyproject.toml` / `noxfile.py` / `tox.ini`, `Cargo.toml`, `go.mod`, then the checks actually run by `.github/workflows/*.yml` (the CI file is the most honest source — it is the gate that must pass). Record the *scoped* form too (how to run one module/package's tests), since the inner loop uses it.
- **Whether a merge webhook is wired** — the plugin's `webhookWired` config. This decides who closes items (Step 3.5). Do not guess it from the presence of a PR link.
- **Collision hotspots** — see Step 1.6; derived from history, never hardcoded.

If detection fails for the test command, say so plainly and ask, rather than inventing a command that will fail in every build.

## Step 1 — Resolve the target set

Parse `$ARGUMENTS` into a list of item refs, staying at index level:

- **Explicit refs** (`LOG-321 LOG-322`) → that's your set.
- **Filter** (`--tag costing --count 10`, `--mine`, a status) → `list_backlog` with `label` / `assignee_email` / `status` + `limit`, ranked by priority.
- **Nothing / "the next N"** → `backlog_stats` for shape, then `briefing`, then `list_backlog` for the top unclaimed items by priority.

**Claim the set** once it survives Steps 1.5–1.6: `update_items` to set `status=in_progress` + assignee across the batch in one call. Claim *after* tiering and fusing, so refused and deferred items stay unclaimed for the next run.

**Sanity-check each item is a real build brief.** The triage pass (Step 1.6) reports thin items — one-liner, no acceptance criteria, no affected-files section. Do not build from guesswork: flesh it out with `logbook:add` or surface it to the user, and drop it from the batch until it's a brief.

## Step 1.5 — Classify each item by size/risk, then enforce batch caps

A tag is a *theme*, not a size filter — `--tag X --count 15` will happily grab multi-day features as if they were quick fixes. Before queuing, classify every resolved item into a tier and enforce the caps. Prefer the **`Size / risk tier`** line if the item's spec carries one (the `logbook:add` template adds it); otherwise derive it from the signals below.

| Tier | Signals | Max queued per run |
|---|---|---|
| **S — Simple** | Single module/package; no schema change or migration; no shared foundation code; bugfix or small endpoint; a handful of acceptance criteria. | **15** |
| **M — Moderate** | 1–2 modules; may include one migration; more involved logic but bounded scope. | **10** |
| **L — Large** | New module, schema *design*, cross-module change, or security/auth-touching; feature-sized. | **4–5** |
| **XL — Major overhaul** | Architectural change, broad refactor across many modules, or rework of the shared foundation the whole repo imports. | **Do not automate.** |

Rules:

- **XL items are refused.** Do not build them in this loop. Surface them to the user with: *"LOG-NNN looks like a major overhaul (<why>). Recommend engineer-in-the-loop development rather than the automated gauntlet."* Drop it from the batch.
- **Enforce the per-tier cap.** If the requested count exceeds a tier's cap, take the highest-priority items up to the cap and **report exactly what was deferred** (ref + tier) in your message to the user — never silently truncate. **No silent caps.** Deferred items stay unclaimed for the next run.
- **A mixed batch is capped per tier independently**, and lean conservative: a session heavy on L items is where supervision matters most — prefer fewer, and tell the user the composition before starting (e.g. "queuing 8 S + 2 M; 1 XL refused").
- When unsure between two tiers, pick the **larger** one — under-queuing is cheap, an overnight run that bites off a feature it can't finish is not.

## Step 1.6 — Conflict-cluster the batch (triage at the top of the funnel)

**This is the single highest-leverage step for avoiding the merge cascade.** Before choosing execution mode, predict which items will collide and **fuse the colliders into one build unit**. Two items that edit the same file are far cheaper to build as one branch/PR/review (the author resolves the overlap *while building*, with full context) than as two standalones that a merge-time agent must reconcile blind.

**Run this as a triage subagent**, not inline: dispatch one agent with the batch's refs, have it `pull_items` in its own context, and return only the table below. That keeps the payloads out of yours.

**1. Predict each item's file footprint.** For every item, derive the set of files/modules it will touch:
- Primary source: the spec's **`Affected files`** / **`Affected modules`** section and the file refs in its **`Evidence (file refs verified against code)`** lines (the `logbook:add` template captures these — if an item lacks them, that's a thinness signal; flesh it out first).
- Fallback: the module/area label ≈ that module's directory — its service/handler, schema/type, route/controller, README, and its test directory. Resolve the actual layout with a quick `git ls-files <module>` rather than assuming a naming convention.

**2. Build the collision graph.** Two items collide if their footprints share any file. **Derive this repo's high-collision files from history rather than assuming a list** — every codebase has them, and they are never the ones you'd guess:

```bash
# Churn hotspots: files most commits touch
git log --since="6 months ago" --name-only --pretty=format: -- . \
  | sed '/^$/d' | sort | uniq -c | sort -rn | head -25

# Co-change: what historically lands in the same commit as a footprint file
git log --since="6 months ago" --name-only --pretty=format:'---' -- <footprint-file> \
  | sed '/^$/d' | sort | uniq -c | sort -rn | head -15
```

The top of the churn list plus anything with a high co-change count against a batch footprint file *is* your hotspot set — typically a shared model/schema module, an event or route registry, a DI/container wiring file, a shared docs table, and **any new migration** (every migration cut from the current head is a future multi-head). Report the derived hotspots; they change as the repo changes.

**3. Fuse same-file colliders into one unit.** Items that share a *primary* file (the same module's service/schema/routes — e.g. two features touching the same loader function, or five items all editing the same pair of domain objects) → **one combined branch, one PR, one review**. Give the builder all the fused items' refs and have it satisfy them together (one feature-coherent change). Re-tier the fused unit (3 fused S items in one module ≈ one M unit) and re-check the tier cap against the *fused* count.

**4. Items sharing only a cross-cutting file** (a registry, a shared model module, a docs table, or "each adds a migration") do **not** all fuse into one mega-unit — that would serialize unrelated work. Instead, keep them separate but **tag them in a `conflict map`** you carry to the merge step: `{file → [items that touch it]}` plus `{migration items → the shared parent revision they all cut from}`. The merge planner uses this to order merges and pre-empt a branched migration chain.

**5. Report the triage to the user** before building: how many raw items → how many build units after fusing, which items were fused and why (shared file), and the carried conflict map. Example: *"12 items → 8 build units: fused {LOG-360,366,379,370,377,373} into one scope unit (all edit the campaign/project pair); flagged {LOG-253,231,294} as event-registry collisions and {LOG-224,181,211} as migration-chain items for the merge planner."*

**Heuristic, not a straitjacket:** when two items are genuinely independent features that merely live in the same module but touch disjoint functions, fusing is optional — prefer fusing when they touch the *same function/class*, flag-for-merge-planner when they only share a file loosely. When unsure, fuse (one bigger coherent PR beats two that conflict).

## Step 1.7 — UI spec gate (items that render a surface)

If the batch contains items that render a user-facing surface (page, tab, panel, drawer, flow — signalled by a front-end/UI label or by a footprint under the app's UI directory), **check for an attached UX spec before queuing them.** The triage subagent's `pull_items` payload carries a `documents` array per item; have it report, per surface item, whether a `ux-spec.md` document is attached.

- **Attached** → queue it. The builder reads the body with `get_document(number, "ux-spec.md")` and builds to its depth bar.
- **Missing** → do not build it blind. Either invoke `logbook:ux` to spec the surface and attach it, or drop the item from the batch and report it as *needs UX spec*. A surface-shaped item without a spec is how thin v0s ship.

This is a real gate with a real check, not an aspiration. If the project has no UI, resolve this step to a no-op and say so once.

## Step 2 — Choose the execution mode (ASK unless --mode given)

If `--mode` is not in the arguments, **ask the user** before starting:

> Run these N units **sequentially** (one at a time, easy to supervise, low token burn) or as a **parallel dynamic workflow** (worktree-isolated, faster for big batches, higher burn)?

Use `AskUserQuestion`. Then:

- **Sequential** — process units one at a time in priority order via the per-unit loop below (dispatch each role with the `Agent` tool). This is the default supervised flow.
- **Parallel** — pipeline the units: each flows through a build stage then a review/fix loop independently, each in **its own git worktree** (`isolation: 'worktree'`) to avoid file contention. Invoking this skill is the explicit opt-in for parallel dispatch. Cap concurrency to what the local machine can sustain — a small fleet (3–4) beats a large one that thrashes.

## Step 3 — The per-unit loop

For each build unit (one item, or one fused cluster), run this loop. In sequential mode you run it inline; in parallel mode it's one unit's pipeline.

1. **Build.** Dispatch a subagent with the `logbook:build` skill, passing the unit's refs, the resolved base branch, and the resolved test/lint commands. It pulls/claims, branches off the base branch, builds per the TDD + docs + test gates, runs scoped tests + a CI pre-flight, opens a PR targeting the base branch, and attaches the branch + PR to the item(s). **Build-stage budget ~60 min:** the builder triages anything slower — if *test execution* is the bottleneck (hung/deadlocked/over-budget runner) it notes "full suite must run in CI" and ships the PR anyway (CI is the authoritative test gate; a slow local run never blocks); if the *work* itself is stuck, it sets `status=stuck` and stops rather than faking a PR.

2. **Review (fresh eyes).** Dispatch a **separate subagent** with the `logbook:review` skill.

   **How fresh eyes is guaranteed, concretely** — this is a dispatch property, not a promise:
   - The reviewer is a **new `Agent` call**, never a `SendMessage` to the builder and never the builder's own turn. A fresh agent starts with an empty context window; that is the mechanism.
   - It is handed **only** the ref(s), the PR number, and the round number. Never the builder's transcript, reasoning, notes-to-self, or "here's why I did X" — those are exactly the rationalizations the review exists to defeat. The reviewer re-derives everything from the item spec and the diff.
   - Every fix round gets a **new** reviewer agent, not a re-prompted one. A reviewer that already blessed round 1 is not fresh for round 2.
   - **You (the orchestrator) never review.** If you find yourself reading the diff to decide the verdict, you have collapsed two roles into one and the loop is broken.

   It reviews the PR against the spec + the repo's gates and posts ONE structured comment with a verdict (`READY FOR HUMAN REVIEW` or `CHANGES REQUESTED`), and returns that verdict to you.

3. **Decide on the verdict:**
   - **CHANGES REQUESTED** → hand the comment back to the builder agent. It addresses the feedback (verify, don't blindly comply), pushes, and replies on the PR. Then go back to step 2 with a **fresh** reviewer. Increment the round number.
   - **READY FOR HUMAN REVIEW** → the loop terminates for this unit.

4. **Hard fix-loop cap (anti-infinite-loop — non-negotiable for unattended runs).** Track the round count per unit. The cap is **3 fix rounds** by default (override with `--max-rounds N`). When a unit reaches the cap without a READY verdict, **STOP looping it immediately** — do not attempt another fix. Set `status=stuck`, `add_note` summarizing what kept bouncing (the recurring review objection + what was tried), leave the PR open for a human, and move to the next unit. This is the safety stop that keeps an overnight run from burning the whole session — and your token budget — on one item that two agents can't converge. **A build/review pair that hasn't converged in 3 rounds is a signal the spec or the change is wrong, which is a human decision, not more agent cycles.** Surface every capped item prominently in the final summary.

## Step 3.5 — Finalize (who closes the item depends on the webhook)

On READY: leave a final PR comment "Ready for human review" and confirm the PR + branch links are attached to the item(s). Then close out according to the resolved `webhookWired` config — **this is not optional bookkeeping; get it wrong and finished work is either double-written or never closed anywhere:**

- **`webhookWired: true`** — **do NOT set the item to done.** The merge webhook crosses it off and writes the resolution stub automatically when the PR merges; marking done now makes that a no-op and loses the auto-resolution. Leave it `in_progress` with the PR attached, awaiting human review + merge.
- **`webhookWired: false`** — **you are the only thing that will ever close this item.** Nothing downstream is listening. Once the PR is merged, `update_item` to `done` with a resolution summarizing what shipped (problem solved, how, PR link). If the human hasn't merged yet when the run ends, leave it `in_progress`, say so explicitly in the summary, and tell the user these items need closing after merge — never let "the webhook will get it" be the reason an item sits open forever.

Then **render the completion card** (see below) to the user so each finished unit lands as a satisfying, skimmable summary — not buried in tool output.

## Step 4 — Summary

When the batch is done, report a table: item ref · title · tier (S/M/L) · PR link · final verdict · rounds taken · status. Then list, separately and prominently: **capped/stuck items** (hit the fix-loop cap — need human attention), **deferred items** (dropped to honor a tier cap — still unclaimed), **refused items** (XL — recommended for engineer-in-the-loop), and **items needing UX specs** (Step 1.7). Also surface the **conflict map** carried from Step 1.6 (which READY PRs share files / form a migration chain). Close with the **batch banner** (see below), then run Step 5.

## Step 5 — Merge-order & conflict map (delegate to `logbook:merge`)

A clean PR is not a *mergeable* PR when several land against a moving base branch. After the batch is READY, invoke the **`logbook:merge`** skill to turn the conflict map into an executable plan. It:

- Diffs every READY PR's changed files (`gh pr diff --name-only`), builds the real collision graph, and detects the **migration chain** (PRs whose migrations were cut from the same parent revision → a guaranteed branched chain).
- Produces a **merge order**: independent PRs first; sibling/colliding PRs sequentialized; **if the project uses sequential migrations, the batch must produce one linear chain** — re-point each later migration onto the prior one *at merge time*, never all upfront (re-pointing onto an unmerged branch's revision breaks your own CI).
- Optionally **drives** the merges with the proven recipe (merge the base branch *into* the branch — not rebase; reset to the remote tip first; scoped-test gate; CI-green gate; regenerate any committed schema snapshot) and resolves the cascading conflicts as siblings land.

**This is where the "solve conflicts → ready to merge" half of the pipeline happens.** Whether the human or you drives the actual `gh pr merge` is their call (ask), but the *plan + conflict resolution* is the gauntlet's job, not the human's. If Step 1.6 fused well, this step is short (few collisions); if the batch was large and cross-cutting, this is where the linearization runbook pays off.

## Step 5.5 — Visual-review gate (UI PRs only — before merge)

**For any PR that changes a user-facing surface, "build + lint green" is NOT a sufficient merge gate.** Type-checkers and linters prove the code compiles and is clean; they cannot see the surface the spec optimized for — hierarchy, every state, status colors, honest affordances, real copy. A type-checker passes happily on a thin or dishonest UI. So before you recommend or drive a UI merge:

1. **Get the change in front of a *populated* app, not an empty shell.** Build an integration preview (the READY UI PRs merged into a throwaway branch off the base branch, entry-point/router conflicts resolved keep-both), run the app's dev server against a live backend with seed data, and confirm each surface actually loads. Reuse a backend that's already running where possible — don't stand up a redundant stack. A dev server with no data behind it shows only error/empty states, which is *not* the UX under review.
2. **Hand the user a per-surface visual-review checklist**, mapped to each PR's `ux-spec.md` depth bar: the exact URL + click path (which page/tab/drawer), the states to exercise (empty / populated / partial / blocked / error), and the specific depth wins to confirm (e.g. *"Approve stays disabled until a passing verdict; no leaked enum values on any button face; the banner names which record blocks the gate"*). Tell them what "good" looks like so they can sign off in seconds.
3. **The human's visual sign-off is the merge gate for UI work.** Only drive `gh pr merge` after they've reviewed (or explicitly waived it). This is the UI analogue of the human's product review — the thing the automated gates structurally cannot do.

## Continuous flow — suggest the next item (the "hit `y` to continue" loop)

The gauntlet is most useful as a conveyor: finish one, tee up the next, keep going with a light human checkpoint. After a unit's completion card (one-at-a-time flow) or after the batch banner (batch flow), **suggest the next item and pause for a one-key confirm**:

```
▶ Next up: LOG-NNN — <title>  (tier S · <one-line why this one>)
  Continue?  y = build it   ·   n = stop here   ·   or name another ref/tag
```

- On **`y`** (or Enter): claim it and loop back to Step 3.
- On **`n`** / stop: end cleanly (the banner was the closer).
- If the user names a different ref or tag: switch the target to that.

**Picking the suggestion** — rank in this order, skipping anything XL:

1. **Remaining items** in the current run's target set.
2. **Unclaimed siblings / related** of what you just built (same cluster via linked items, or same module) — momentum: the context and files are still warm, and related work batches naturally.
3. **Next highest-priority unclaimed item matching the run's `--tag`** (`list_backlog`).
4. Else the **next highest-priority unclaimed item overall** (`briefing`).

Always show *why* you picked it (priority, relation, tier) — a suggestion the user can't reason about isn't one they can confidently hit `y` on. If the only candidates are XL, say so and recommend engineer-in-the-loop rather than auto-teeing-up something the gauntlet would refuse.

**The confirm is the human checkpoint — never silently chain into the next build.** The `y` is what keeps a continuous run human-governed. For a genuinely unattended run, the user gives an explicit batch (`--count N`) up front — that authorizes the whole batch without per-item prompts, and the suggestion then fires only when the batch is exhausted ("batch done — continue with LOG-NNN? `y`").

## Completion card (render one per unit on READY)

When a unit reaches READY, print a card like this to the user (chat, not the PR). Fill it from data you already have: **Problem** from the item's spec/completion note, **Solution** from the completion note's "How it was fixed", **Nits** from the reviewer's "Should consider" notes plus any build caveats, **Spawned** from any logbook items the build filed mid-work (the completion note's spawned-items line). Keep each line to one or two lines; box alignment is best-effort — content matters more than perfect ASCII.

```
╭─ ⚙  GAUNTLET · LOG-NNN · ✓ READY FOR REVIEW ─────────────────
│
│  build ✓   review ✓ (round N)   PR #NNN → <base branch>
│
│  📋 Problem    <one line: what we set out to solve>
│  🔧 Solution   <one or two lines: the approach taken>
│  🔍 Nits       <reviewer's non-blocking notes — or "none">
│  🌱 Spawned    <new logbook items filed during the build — or "none">
│  🔗 PR         <url>   →  your turn: review on GitHub
│
╰──────────────────────────────────────────────────────────────
```

(Left-anchored, no right border — robust against emoji/content width. Don't chase a perfect right edge.)

For a **capped/stuck** item, render the same card with a `⚠ STUCK (hit 3-round cap)` header and a `🚧 Blocking` line (the recurring objection) instead of the ready framing — the human needs to see why it didn't converge.

## Batch banner (render once, at the very end)

Close the run with this banner + a one-line scoreboard. Fill in the count of READY vs stuck:

```
 _       ___    ____  ____    ___    ___   _  __
| |     / _ \  / ___|| __ )  / _ \  / _ \ | |/ /
| |    | | | || |  _ |  _ \ | | | || | | || ' / 
| |___ | |_| || |_| || |_) || |_| || |_| || . \ 
|_____| \___/  \____||____/  \___/  \___/ |_|\_\

   ✓ <R> ready   ⚠ <S> stuck   ·   PRs awaiting your review on GitHub
```

(If the ASCII banner won't align cleanly, a single tidy line — `✓ N ready · ⚠ M stuck · PRs awaiting review` — is a fine fallback. Never let the art break the summary.)

## Guardrails

- **Builder and reviewer are different agents.** If you find yourself reviewing your own build, you've broken the loop.
- **Only the orchestrator writes to logbook** (the builder is the one exception, for its own item). Workers return findings.
- **Never pull the whole batch's payloads into your context** — index rows to scope, subagents to expand.
- **Who closes the item depends on `webhookWired`** (Step 3.5). Webhook on → leave it `in_progress` with the PR attached. Webhook off → you close it, or nobody does.
- **Honor the test tiers** — scoped tests in the inner loop, full suite only at pre-push/CI. Don't run the full suite per task.
- **Stay on the resolved base branch** as the PR base — never assume `main` if Step 0 resolved something else.
- **No silent caps.** Every deferred, refused, or stuck item is named in the summary.
- **UI PRs get a visual-review gate before merge** (Step 5.5) — green build + lint never substitutes for a populated dev-server pass plus a per-surface review checklist handed to the user. UI work is not merge-ready on type-check alone.

## Related skills

`logbook:add` (file a build-ready item) · `logbook:plan` (shape a batch before running it) · `logbook:explore` (blast radius) · `logbook:ux` (spec a surface) · `logbook:build` · `logbook:review` · `logbook:merge` (merge order + conflict resolution) · `logbook:sweep` (board hygiene).
