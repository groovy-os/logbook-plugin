---
name: sweep
description: Runs an orchestrated cleanup audit of the logbook backlog — finds items that are already done (merged-but-uncrossed), obsolete, duplicated, or stale-briefed, closes them with evidence-backed resolutions, and produces a bill of work for the human. Runs as a director/worker system with model tiering and single-writer discipline. Use when the user says "clean the backlog", "sweep the backlog", "audit the logbook", "what's stale", "which of these are already done", "is the backlog lying to me", or runs /logbook:sweep.
argument-hint: "[--tag <label> | --mine | --all] [--dry-run]"
---

# Sweeping the logbook: an orchestrated staleness audit

Backlogs rot in predictable ways: work merged straight to the base branch never gets crossed off (the status lies), architecture moves out from under an item (the brief targets code that no longer exists), the same problem gets filed twice from two sessions, and evidence/line-refs drift. `logbook:sweep` audits the whole open backlog against **reality** — git history, PR state, current code — and leaves behind a smaller, truthful backlog plus a bill of work for the human.

If the plugin's `webhookWired` setting is **false**, nothing closes items automatically — expect the DONE bucket to be the biggest finding of the sweep, and say so in the report.

## Architecture: director + tiered workers

**You (the invoking agent) are the DIRECTOR.** You never delegate judgement you can make yourself from returned evidence, and you are the **single writer to logbook** — workers research and return structured findings; only you call `update_item(s)` / `create_item(s)` / `add_note` / `link_items`. This prevents duplicate notes and racing writes when many workers run in parallel.

Route each role to the cheapest model that fits (the Agent tool's `model:` option). The tiers are **relative capability**, not product names — map them onto whatever model classes your harness exposes, and collapse tiers if it exposes only one.

| Role | Model class | Work |
|---|---|---|
| **Retriever** | a fast, cheap model — mechanical retrieval, no judgement | Per item: `git log origin/<base> --grep "LOG-NNN" --oneline`, `gh pr list --state merged --search "LOG-NNN"`, do the files the item cites still exist (`ls`/`Glob`), does the item's branch still exist on the remote. Returns raw findings only. |
| **Code checker** | a mid-tier coding model — reads code, confirms claims | For items the retriever couldn't resolve: open the cited files and check whether the described problem still exists in current code (the missing guard is still missing, the hardcoded value is still hardcoded). Returns "problem still present / problem gone (what changed) / can't tell" with file:line evidence. |
| **Judge** | a frontier model — cross-item judgement, whole set in one context | Verdict inference on mixed evidence, plus the passes that need the whole set in view: duplicate clustering, theme detection ("these 6 items are all superseded by the LOG-804 planner epic"). Returns verdicts with confidence + reasoning. |
| **Director** | you, the orchestrator | Target-set resolution, dispatch, the final call on everything the judge marks uncertain, all logbook writes, the bill of work, the report. |

Don't over-dispatch: if the sweep is small (≤ ~10 items), the director can run the retrieval-tier checks inline and only spawn workers for the code-check and judgement stages.

## Step 0 — Discover the repo (once, cheap)

```bash
git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null | sed 's|^origin/||'
```
That is the **base branch** every history query below runs against. If it is unset, fall back to the plugin's `baseBranch` setting (default `main`). Fetch it once before any fan-out so every worker reads fresh history:

```bash
git fetch origin <base>
```

## Step 1 — Scope the sweep (director) — index rows only

- **`backlog_stats` first** — the shape of the backlog in one small object. It sizes the audit before you spend anything on it, and its `unsized` count is itself an audit finding worth reporting.
- **Then `briefing` for the scoreboard, and `list_backlog` at index verbosity** across the open statuses (backlog, in_progress, stuck). One line per item — ref, title, summary, size, status, assignees, labels — is enough to bucket provisionally and to shard the workers. Batch tools, never per-item loops.
- **Do not call `pull_items` in your own context.** The specs — problem, description, tasks, notes, links with live PR state, documents — are what the retrievers and code-checkers read in *their* contexts. Give each worker a slice of refs; it calls `pull_items` itself and returns structured evidence. Holding the whole backlog here defeats the tiering the rest of this skill is built on.
- Expand a single item yourself (`pull_item`) **only** to adjudicate a verdict the workers disagree on, or to write its resolution — and then only that item.
- Do **not** claim — this is an audit, not a pickup.
- Record the raw count. Every item must land in exactly one verdict bucket at the end — **no silent drops, no silent caps.**

## Step 2 — Evidence pass (cheap-model fan-out)

For each item, a retriever gathers the mechanical facts (batch several items per worker — one worker per ~10 items is plenty). For > ~15 items, pipeline with whatever workflow tool your harness offers; below that, parallel `Agent` calls in a single message.

Each worker returns, per item: merged-PR hits, base-branch history hits (`git log origin/<base> --grep`, plus `--grep` on distinctive strings from the item's title/acceptance criteria — the LOG ref alone misses work merged without a ref), cited-file existence, linked-PR state.

**There is no logbook tool that lists pull requests.** PR state comes from the `gh` CLI, or from the links already carried on the item that the worker pulled.

**Triage on return (director):**
- **Hard DONE evidence** (merged PR implementing it, or a base-branch commit whose diff covers the acceptance criteria) → verdict bucket **DONE**, no further stages.
- **Hard OBSOLETE evidence** (the code surface the item targets was removed/replaced, identifiable commit) → bucket **OBSOLETE**.
- Everything else → Step 3.

## Step 3 — Code-check pass (mid-tier, only for unresolved items)

Code checkers open the cited code and answer: does the described problem still exist? Items whose problem is **verifiably gone** get promoted toward DONE/OBSOLETE *only if* the worker can name the commit or refactor that removed it — otherwise the finding is "problem gone, cause unidentified", which is **propose-list** material, not auto-close.

## Step 4 — Judgement pass (frontier judge)

Give one judge the **full annotated set** (all items + all evidence) for the cross-item work:
- **Duplicate clusters** — same problem filed twice; pick the survivor (older/meatier), mark the rest DUPLICATE.
- **Superseded-by-direction** — items an epic or decision has made moot (check the epics and the resolutions of recently-closed items).
- **Stale briefs** — problem still real, but evidence/approach has drifted.
- A per-item verdict + confidence for everything still unresolved.

If the set is too large for one context, shard by label/module and run a final merge pass.

## Step 4.5 — Epics rot differently

An **epic** is a container; its state follows its children, so the usual evidence bar does not apply. Bucket them by what their children say:

- **All children done, epic still open** → the commonest epic rot. Close it with a resolution summarising what the set delivered, citing the children.
- **Some children done, rest stale or obsolete** → the epic is not done. Fix or close the stale children first; the epic follows.
- **No children at all** → this is a plan that never happened. Do not auto-close it; it is a NEEDS-HUMAN decision (revive with children, or abandon).
- **Children that outlived their epic** → a log whose parent was deleted keeps working (the link is cleared, not the log). Flag orphans; they are usually fine, but a cluster of them means a plan was dropped mid-flight.

Never close an epic on the evidence you would use for a log — a merged pull request closes a child, never the container.

## Step 5 — Verdicts and writes (director only)

Every item lands in one bucket. **The auto-close evidence bar is mechanical, not vibes:** an item may auto-close ONLY if at least one of — (a) a **merged PR** implements it; (b) an identified **base-branch commit** implements it or removed its target code; (c) it is a **duplicate** of a specific surviving item, named. A judge's "looks done, high confidence" without (a)/(b)/(c) goes to the propose list. When in doubt, propose.

| Bucket | Action (director) |
|---|---|
| **DONE** | `update_items` with `status=done` + a changelog-quality `resolution` **citing the evidence** (PR # / commit sha, what shipped where). |
| **OBSOLETE** | Close with `resolution` naming what superseded it (commit/epic/decision). |
| **DUPLICATE** | Close the twin with `resolution="duplicate of LOG-NNN"`, `link_items` to the survivor, and move any unique detail from the twin onto the survivor as a note. |
| **STALE BRIEF** | Keep open. Correct the item per the `logbook:add` discipline (fresh evidence refs, corrected approach) or, for a big blast-radius question, flag it for `/logbook:explore`. `add_note` what was corrected. |
| **VALID** | Keep open, untouched — this is `/logbook:plan`'s input. If it lacks a size/risk tier, add one (you have the evidence in hand). |
| **PROPOSE** | Ambiguous — assemble the kill list with per-item evidence and present it to the user for **one batched approve/reject**, then process approvals as above. |
| **NEEDS-HUMAN** | A product/priority decision only the human can make → the bill of work. |

Use `update_items` for bulk closes (items process independently — check per-ref `ok`/`error` and retry only failures).

With `--dry-run`, nothing closes: every bucket collapses into PROPOSE and the report is the deliverable.

## Step 6 — The bill of work (logbook item, assigned to the human)

File **one dated item** — `create_item` titled "Bill of work — <date>", assigned to the user's email, labelled `product` — with a **task checklist**, one task per action only they can take, ordered by leverage:
1. PRs already open and awaiting their review (from the link payloads workers returned in Step 2).
2. Approve/reject the PROPOSE kill list (if they haven't inline).
3. NEEDS-HUMAN decisions, each task naming the item and the specific question.
4. XL / engineer-in-the-loop items that are blocking clusters of other work.

`link_items` the bill to every item it references. File standalone items only for genuinely separate new work discovered during the audit (full `logbook:add` discipline).

## Step 7 — Report

Close with the accounting: raw count swept → per-bucket counts, the closed list (ref · title · evidence one-liner), the propose-list verdict, the bill-of-work item ref, and the surviving VALID count — with the suggestion to run `/logbook:plan` on it next. Offer `archive_done_items` as an optional final sweep.

## Guardrails

- **Single writer.** Workers never call logbook mutation tools. If a worker reports it "filed an item", that's a defect — re-check for duplicates.
- **Evidence bar is non-negotiable.** No close without (a)/(b)/(c) above. Resolutions cite shas/PRs — they become documentation.
- **Closes are recoverable, so bias to action *within* the bar** — a wrongly-proposed item costs one human glance; a wrongly-closed one loses backlog signal until someone notices.
- **Context economy is a rule, not a preference.** Index rows in the director; full payloads only in worker contexts.
- **Don't fix code.** The sweep finds work; it never does it. Discovered bugs get filed (`logbook:add` discipline), not patched.
- **Uncommitted and worktree state is out of scope** — audit against `origin/<base>`, and say so in the report so "in progress in someone's worktree" isn't mistaken for stale.
