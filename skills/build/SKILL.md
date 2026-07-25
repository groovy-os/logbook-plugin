---
name: build
description: Implements one logbook item (or one fused cluster of items) end-to-end — pulls and claims it, branches off the repo's base branch, builds it test-first, runs the scoped tests plus a CI pre-flight, opens a pull request, and wires the branch and PR back to the item. Use when the user says "build LOG-NNN" or "work this item", and when logbook:loop dispatches the build stage for a unit.
---

# Building one logbook item

This is the per-item builder — the inner unit the `logbook:loop` orchestrator drives once per build unit. It encodes the *discipline*; the stack-specific commands are discovered at runtime (see Step 0) and the generic mechanics are delegated (see "Delegate, don't duplicate").

**Building a fused cluster?** If `logbook:loop` Step 1.6 fused several items that touch the same files into one build unit, you may be handed **multiple refs** (e.g. "build LOG-360 + LOG-366 + LOG-379 together — all edit the campaign/project pair"). Treat them as **one coherent change on one branch → one PR**: pull/claim each, satisfy every item's brief in the same diff, name the branch/PR for the cluster (e.g. `claude/scope-cluster-soft-delete-guards`), reference all refs in the PR body, and attach the one PR to *each* item. Building fused items together is the whole point — it resolves their file overlap *here*, with full context, instead of leaving it for a blind merge-time agent. The gates below apply to the combined change.

## Step 0 — Resolve the repo's conventions before you touch anything

The orchestrator normally hands these to you. If you're running standalone, resolve them yourself and say what you resolved:

- **Base branch** — `git symbolic-ref --short refs/remotes/origin/HEAD` (`origin/main` → `main`); fall back to `git remote show origin`, then the plugin's `baseBranch` config, then `main`. Branch from it and target your PR at it. Never assume `main` when the repo integrates on `develop`/`staging`.
- **Test command (scoped and full)** — the plugin's `testCommand` config if set; otherwise detect from `package.json` scripts, a `Makefile`, `pyproject.toml`/`tox.ini`/`noxfile.py`, `Cargo.toml`, `go.mod`, and above all **the checks `.github/workflows/*.yml` actually runs** — CI is the authoritative gate, so match it.
- **Format / lint / type commands** — same detection order. These are the cheap checks that otherwise fail CI in the first ten seconds.
- **Project gates** — read the repo's contributor guide (`CLAUDE.md` / `AGENTS.md` / `CONTRIBUTING.md`) and honor the gates it defines. Those house rules outrank anything generic below.

If you can't detect a test command and none is configured, say so and ask rather than inventing one.

## Checklist (create a TodoWrite todo per step)

1. **Pull & claim.** `pull_item("LOG-NNN", claim: true)` — one call returns the problem, description, task checklist, notes, and links with live PR state, and assigns you + sets in_progress. If the item is thin (no real build brief), **stop and flesh it out first** with `logbook:add` — do not build from a one-liner.

   **🛑 UX gate for surface items.** If the item renders a user-facing surface (page/tab/panel/drawer/flow — usually carrying a front-end/UI label), it MUST have a `ux-spec.md` document attached before you build (the `pull_item` payload lists it in the `documents` array). If it's missing, **stop and invoke `logbook:ux` first** to spec the surface (job, states, hierarchy, affordances, copy) and attach it — *then* build to that spec. When it's present, **`get_document(number, "ux-spec.md")` to read the spec body** (the `documents` array is metadata only — no content), and build the UI to satisfy its **depth bar** + the depth tasks, not just the endpoints. Building a surface-shaped item without a UX spec is how thin v0s ship; don't. (A fix with no visual surface can carry a one-line `N/A` `ux-spec.md`, which satisfies the gate.)

2. **Branch off the base branch.** Attach the branch immediately: `update_item add_links [{kind:"branch", value:"<branch-name>"}]`. If you're in a fresh worktree, run whatever bootstrap the repo's contributor guide specifies (env files, an isolated test database, dependency install) before building.

   **🛑 Worktree guard (non-negotiable — a real run corrupted a shared checkout exactly this way).** If you are working in a git worktree, **never edit the canonical checkout**. Establish your root once and resolve every path against it:

   ```bash
   pwd && git rev-parse --show-toplevel   # this is YOUR worktree root
   ```

   Item specs cite files by repo-relative path (`api/modules/orders/service.py`). Resolve each one against **your worktree root** — never against the canonical checkout, and never against a path you remember from an earlier session. Concretely: every path you pass to Read/Edit/Write must begin with your worktree root; do not `cd` outside it; do not run `git reset --hard` or `git clean` outside it. Writing to the canonical checkout corrupts every other live session on that machine, silently, and the failure surfaces hours later in someone else's diff. The only acceptable reference to a path outside your worktree is a read-only symlink target created by the bootstrap.

3. **Build it, test-first.** Write the failing test, then the code. Honor the item's Test plan and any security/isolation section as written. Work the task checklist down with `complete_task(number, "<task substring>")` *as each piece lands* — not batched at the end (that's what keeps the board's progress bars honest). Discover a missing step → `add_task`.

4. **Honor the gates** (these are what reviews bounce on — get them right the first time):

   - **The repo's own rules first.** Whatever `CLAUDE.md` / `AGENTS.md` / `CONTRIBUTING.md` mandates — architecture boundaries, naming, error handling, commit hygiene — is a gate. Read it before writing code, not after the review bounces.
   - **Authorization / data isolation.** If the project is multi-tenant or has row-level ownership, enforce it where the project enforces it. A good default when the repo is silent: validate any cross-row foreign key coming from request input at the *service* layer (not just the route), return **404 rather than 403** on a mismatch so the endpoint doesn't leak existence, keep tenant/owner identifiers out of request and response bodies (derive them from the authenticated principal), and put new routes behind the project's authorization dependency rather than a raw "current user" fetch.
   - **Docs in the same PR.** Update the architecture/design doc if the structure changed, and the module's README if its surface changed. Respect any docs the repo marks generated or frozen — don't hand-edit those.
   - **Integration / scenario tests.** If the project keeps end-to-end or scenario tests, add the 2–4 high-value ones for this change, following the repo's existing naming and header conventions (match a neighbouring file — don't invent a scheme). *Write* them now; they *run* at pre-push/CI, not in the inner loop.
   - **Migrations.** If the project uses sequential migrations, check for a single head before and after your change, and cut yours from the current head. Apply whatever the project requires alongside it (per-schema application, a regenerated + committed schema snapshot if a CI drift gate enforces one). **Record your migration's revision id and its parent in the completion note and PR body** — `logbook:merge` reads these to linearize the chain across the batch. Do **not** pre-emptively re-point your parent onto another *unmerged* branch's revision; that breaks your own CI. The merge planner re-points at merge time.

5. **Test — tiered, scoped.** Inner loop: run only the tests for the module/package you touched (the scoped form of the resolved test command). **Carve-out:** if you touched shared foundation code that everything imports, or wrote a migration, run the full suite even in the inner loop.

   **Build-stage budget (~60 min wall-clock) + the test-execution escape hatch.** Don't let one item eat an unattended session. If you're approaching ~60 minutes on a single build, stop and triage *what* is slow — and treat the two causes completely differently:

   - **The test RUN is the bottleneck** (the runner hangs, deadlocks, or simply can't finish in budget — a full-suite carve-out, a database truncate deadlock, an async concurrency artifact, a zombie service holding a port). This is a **tooling** problem, not a code-correctness one. Do **not** keep grinding and do **not** block the item. Instead: run the smallest subset that proves your change (your new tests + the directly-touched code); if even that won't complete cleanly, **`add_note` on the item stating "full suite must run in CI — local run was [hung/deadlocked/over-budget]"**, call it out in the PR body and completion note, and **proceed to push + PR**. CI runs the full suite as the authoritative gate — that's its job. A slow local run must never hold up a PR.
   - **The WORK is the bottleneck** (you can't get the implementation right, the design is wrong, the environment won't come up at all, repeated *real* failures that aren't harness artifacts). This is genuine — do **not** ship it. Set `status=stuck`, `add_note` what's blocking, and stop. Don't fake a green PR to beat the clock.

   The line between these is "would this fail in CI too?" A hung or slow local harness won't (CI has a clean runner); a real bug will. Lean on CI for the former; respect the block for the latter.

6. **CI pre-flight before pushing.** Run the repo's format → lint → type-check → test chain, in that order, failing fast — the same commands CI runs, resolved in Step 0. If it passes locally, CI passes. Use the *exact* tools CI uses (a different formatter with different defaults will churn the diff and still fail the gate).

   **Format, lint, and type-check are fast and non-negotiable — they MUST pass locally before you push.** Only the *test* step may degrade to CI under the escape hatch above; the lint/type checks never do.

7. **Push & PR.** Push with an explicit refspec — `git push -u origin refs/heads/<branch>:refs/heads/<branch>` — so a worktree that tracks the base branch can't push *to* the base branch. Open the PR **targeting the resolved base branch**.

8. **Wire the PR back.** `update_item add_links [{kind:"pr", value:"<pr-url>"}]`.

   **Do not decide the item's final status yourself.** If a merge webhook is wired, it crosses the item off and writes the resolution stub when the PR merges — marking done now turns that into a no-op and loses the stub. If no webhook is wired, the orchestrator closes the item after merge. Either way your job ends at *attach the PR*; report the PR back and let `logbook:loop` finalize.

9. **Write a build completion note** (`add_note` on the item). This is the performance-tracking and changelog breadcrumb — write it like the entry you'd want to find in six months, and make it answer the first principle: **"what problem were we solving?"** Use this structure:

   ```
   ## Build complete — LOG-NNN

   **Problem solved:** <restate, in your own words, the problem this addressed —
     mirror the spec's Problem field and confirm the change actually answers it.
     If the real problem turned out different from the spec, say so.>
   **How it was fixed:** <approach + the load-bearing decisions, including any
     deviation from the brief and why (e.g. logic placed in the service, not the route).>
   **Files:** <key files changed>
   **Tests:** <what was added and what it asserts>
   **Migration:** <revision id + parent revision, if this PR adds one; else omit>
   **Spawned items:** <refs of any logbook items you filed mid-build for out-of-scope
     gaps you discovered (via logbook:add) — e.g. "LOG-431, LOG-432"; or "none". The
     logbook:loop completion card surfaces this so discovered work isn't lost.>
   **Risks / follow-ups:** <anything a human or the reviewer should scrutinize;
     any out-of-scope gaps discovered — file these as new items (above), don't bury them.>
   ```

   The "Problem solved" line is the load-bearing one. A note that lists files but never restates the problem fails the principle — the next person can't tell whether the change actually solved what it set out to. If you can't crisply state the problem the change solved, that's a signal the build drifted from the spec.

## Delegate, don't duplicate

- TDD red/green discipline, systematic debugging, worktree isolation, and evidence-before-claiming-done are generic engineering skills. If the `superpowers` plugin (or an equivalent) is installed, delegate to its `test-driven-development`, `systematic-debugging`, `using-git-worktrees`, and `verification-before-completion` skills rather than re-deriving them here.
- Blast radius of a change before you start → `logbook:explore`.
- Filing work you discover mid-build → `logbook:add`. File it; don't bury it in a note.

## Writes you own

You are the one worker permitted to write to logbook, and only for the item(s) you were dispatched for: `complete_task`, `add_task`, `add_links`, `add_note`, and `status=stuck`. Everything else — closing items, recording batch decisions, filing the merge plan — belongs to the orchestrator.

## Handoff

When the build is up and the PR is open, the next stage is review (`logbook:review`, dispatched as a **fresh subagent that did not write this code**). If you're running standalone (not under `logbook:loop`), dispatch that reviewer yourself — do not review your own diff. If you're blocked, `status=stuck` + `add_note` explaining what's blocking; notes notify assignees and are the handoff channel.
