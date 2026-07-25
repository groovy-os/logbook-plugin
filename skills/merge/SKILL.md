---
name: merge
description: Plans and executes the merge of a batch of READY logbook pull requests into the base branch without conflict or multi-head cascades — diffs each PR's files, builds a collision graph, linearizes the migration chain so the batch produces one head, produces a conflict-aware merge order, and (optionally) drives the merges with the merge-base-in + scoped-test + CI-gate recipe, resolving cascading conflicts as siblings land. Use at the end of a /logbook:loop batch, or when the user says "merge these PRs", "what order do I merge", "linearize the migrations", "resolve the merge conflicts across these branches", "these PRs all conflict", or runs /logbook:merge.
argument-hint: "[PR numbers | LOG-321 LOG-322 | --all-ready] [--drive]"
---

# logbook:merge — the end-of-run merge planner & driver

A clean PR is not a **mergeable** PR once several land against a moving base branch. This skill turns a set of READY PRs into a single linear base branch with every conflict solved — without the multi-head + conflict cascade that an un-planned 20–30-PR merge produces.

It is the back half of the pipeline: **… build → review → [merge-order/conflict map → solve] → ready to merge.** The human does final product review and clicks merge (or authorizes you to); the *plan and the conflict resolution* are this skill's job.

## When to use
- At the end of a `/logbook:loop` batch, once items are READY.
- Any time you face several open PRs that target the same branch and will collide — shared files, or each adding a migration off the same parent.

## The core lessons this skill encodes

Learned from an un-planned 30-PR batch that took a full extra session to untangle:

1. **Migrations off the same parent = guaranteed multi-head.** If the project uses **sequential/versioned migrations where each revision names its predecessor**, N PRs each adding a migration pointed at the same parent produce N heads on the base branch; the 2nd and later merges fail CI with a "multiple heads" error. **Linearize the chain** — re-point each new migration's predecessor reference — **at merge time, one at a time**. Never all upfront: a re-pointed branch whose predecessor isn't merged yet can't resolve its own parent reference → red CI.
2. **Merge the base branch *into* the feature branch — do not rebase.** A 3-way merge auto-resolves additive changes (new columns in different tables, new functions, disjoint entries in a generated file) and needs **no force-push**. Rebasing five branches that all touch the same registry/model file produces a conflict cascade and risky force-pushes.
3. **Always `git reset --hard origin/<branch>` before touching a branch.** Build and fix agents push from *separate* worktrees, so a build worktree's local ref is often **stale** — missing fix-round commits. Resetting to the remote tip (rather than force-pushing the stale local) is what keeps you from silently dropping a fix.
4. **Scoped tests passing ≠ the merge is sound.** Cross-feature *semantic* conflicts (a new guard in one feature breaking another feature's lifecycle path) are caught by end-to-end / scenario suites and the full CI run, not by the touched module's own tests. Gate each resolution on scoped tests locally, then **trust the full CI suite as the authority** before merging.
5. **CI is the verifier of the things you can't see.** Schema-snapshot drift gates, migration-head checks, and lint gates catch a wrong snapshot or a missed head. Push, let CI confirm, then merge.

## Step 0 — Discover the repo (once, cheap)

```bash
# Base branch — what these PRs target
git symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null | sed 's|^origin/||'
```
If unset, fall back to the plugin's `baseBranch` setting (default `main`). Confirm against what the PRs actually target: `gh pr view <pr> --json baseRefName`.

**Test command.** Take it from the repo, in this order, and only then from config:
1. The CI workflow's test step — `.github/workflows/*.yml`, or the equivalent CI file — is the authority on what "green" means here.
2. The manifest: `package.json` `scripts.test`, `Makefile` `test:` target, `pyproject.toml` / `tox.ini` / `noxfile.py`, `Cargo.toml` → `cargo test`, `go.mod` → `go test ./...`, `Gemfile` → `bundle exec rspec`.
3. The plugin's `testCommand` setting if it is non-blank.
4. If all three fail, **ask** — don't guess a command that silently does nothing and call it green.

Note the runner prefix the repo actually uses (`npm run`, `pnpm`, `uv run`, `poetry run`, `bundle exec`, bare) — it is part of the command.

**Collision hotspots — derived, not hardcoded.** The files most commits touch are this repo's "everything edits it" files:
```bash
git log --since=6.months --name-only --pretty=format: \
  | grep -v '^$' | sort | uniq -c | sort -rn | head -30
```
Drop lockfiles, generated artefacts, and changelogs. What remains is the set to call out in the conflict map.

**Migrations.**
```bash
git ls-files | grep -Ei '(^|/)(migrations?|alembic|db/migrate|prisma/migrations|supabase/migrations|liquibase|flyway)/' | head
```
Identify the tool from what you find and learn its chaining convention before editing anything: a predecessor pointer in the file (Alembic `down_revision`, Django `dependencies`), an ordinal/timestamp prefix (Rails, Prisma, Flyway `V<n>__`), or paired up/down files (golang-migrate). Also find the tool's own head/status command — a regex over predecessor pointers **mis-reports merge revisions** (which can name more than one parent), so use the tool itself as the arbiter.

## Step 1 — Build the conflict map

Gather the READY PRs — from the run, from `gh pr list --state open --json number,title,headRefName`, or from the `links` already carried on the logbook items (`pull_items` on the batch's refs returns them). **There is no logbook tool that lists pull requests**; PR state comes from `gh` or from those links.

```bash
gh pr diff <pr> --name-only        # the files it changes
gh pr view <pr> --json mergeStateStatus,mergeable,headRefName,baseRefName
```

Construct:
- **collision graph**: PRs are adjacent if their changed-file sets intersect. Connected components = groups that must be sequentialized (or were already fused at build time by `/logbook:plan` / `/logbook:loop`).
- **migration chain**: PRs that add a file under the migrations directory found in Step 0. Read each new migration's identifier and predecessor reference. Any two sharing a predecessor (or all pointing at the current base head) are a multi-head set to linearize.
- **hotspot callouts**: which PRs land on the Step 0 hotspot files, and which shared docs/registries they co-edit.

Confirm the base branch's true single head using the **migration tool's own head/status command**, run in a base-branch worktree that you have just reset to the remote tip:
```bash
git fetch origin <base> && git reset --hard origin/<base>
<migration tool> heads          # e.g. alembic heads / django showmigrations / flyway info
```

## Step 1.5 — Triage UNSTABLE PRs (red CI ≠ unmergeable)

A PR can be `MERGEABLE` (no file conflict) yet `UNSTABLE` — mergeable, but a required check is **failing or still running**. Do **not** treat UNSTABLE as broken and skip it, and do **not** treat it as ready and merge over a real break. Triage *why* it is red before it enters the order. The decisive question is **"is this failure in the PR's own diff, or is it the harness?"**

```bash
gh pr checks <pr>                                   # which job is red? pending?
gh pr diff <pr> --name-only                         # the PR's own footprint
gh run view --job <failing-job-id> --log | grep -iE 'FAILED|assert|Error' | head -40
```

Cross the failing test files against the PR's changed files:

- **Failure is IN the diff's files (or the module it touches)** → a *real* break. The PR is **not merge-ready** — do not order it. Route it back to a build/fix pass (`logbook:build`), `add_note` what failed, and exclude it from this merge run. Faking it green wastes a CI cycle and lands a bug.
- **Failure is in UNRELATED files** (end-to-end suites the PR never touches, a different module) → usually a **harness flake**, not the diff. The classic signature is shared-fixture or seed collision under parallel test sharding: a unique-constraint violation on a seeded row, surfacing downstream as a "not found" error because the row was never created. **Confirm** it is a flake by running the named test files *alone*, serially, with no sharding, using the Step 0 test command; if they pass, the parallel collision was the cause. Then proceed — the base-branch merge in Step 3 retriggers CI, and that **fresh run is the arbiter** (lesson #5). Don't block on a stale flake.
- **Checks merely PENDING (nothing failed yet)** → not a signal. Let them finish (`gh pr checks <pr> --watch`) before ordering.

Record the verdict per UNSTABLE PR (real break → bounced; flake → cleared by fresh CI) in the Step 4 report, so a recurring flake signature becomes visible across runs instead of being re-diagnosed from scratch each time.

## Step 2 — Produce the merge order

Order by lowest collision first:
1. **Independent PRs** (no shared files, no migration) — mergeable in any order; merge first to shrink the set.
2. **Migration chain** — pick an order (touch-disjoint tables first; migrations on core/shared tables last). Chain it: PR₁ keeps its predecessor = current base head; PR₂'s predecessor → PR₁'s revision; and so on. **Apply each re-point only when its predecessor is already on the base branch.**
3. **Sibling / colliding groups** — sequentialize; expect each merge to flip the next sibling to `DIRTY` (re-resolve it then).

Show the user the order and the conflict map. Ask whether **they** merge on GitHub or **you** drive it — merging into a shared branch is outward-facing, so get the nod; an explicit "merge them as you go" is sufficient authorization.

## Step 3 — Drive the merges (if authorized)

Per PR, in order. Use a dedicated worktree (or the PR's existing build worktree).

**🛑 Worktree guard.** Operate only inside the worktree. Before any write, confirm where you are and that it is not the user's canonical checkout:
```bash
pwd && git rev-parse --show-toplevel && git rev-parse --git-common-dir
```
Never edit, `cd` into, or run `git` against the canonical checkout: absolute-path edits there corrupt other live sessions, and a stray `git reset --hard` deletes uncommitted work.

**For a PR that is currently CLEAN + green:** just `gh pr merge <pr> --merge`. Let GitHub be the source of truth — if it has a real conflict it rejects the merge; catch that and route to the resolve path.

**For a CONFLICTING / migration PR:**
```bash
cd <worktree>
git fetch origin <branch> <base>
git reset --hard origin/<branch>          # true tip incl. fix-round commits — never skip
git merge origin/<base> --no-edit         # 3-way; auto-resolves additive hunks
```
- Resolve remaining conflicts. **Additive** (different functions, different fields, separate registry entries, separate doc bullets) → keep both. **Semantic** (both edited the SAME function — e.g. one feature's guard must run before another's delete) → combine the logic so both behaviours hold; don't pick a side.
- For a **migration**: re-point the new migration's predecessor reference to the chain predecessor — and any human-readable header/docstring that repeats it, so the file doesn't lie. Then verify a single head with the migration tool's own head command.
- If the project has a **committed schema snapshot with a drift gate**, regenerate it when the 3-way merge didn't. The generator often needs database client binaries the host lacks; if the dev database runs in a container, proxy them with PATH shims rather than installing anything:
  ```bash
  mkdir -p /tmp/dbshim
  printf '#!/usr/bin/env bash\nexec docker exec -i <db-container> psql "$@"\n' > /tmp/dbshim/psql
  printf '#!/usr/bin/env bash\nexec docker exec -i <db-container> pg_dump "$@"\n' > /tmp/dbshim/pg_dump
  chmod +x /tmp/dbshim/*
  PATH="/tmp/dbshim:$PATH" <the repo's snapshot script> --check   # drop --check to regenerate
  ```
  Read credentials from the environment or the project's own env file — never inline a password into a command you write.
- **Gate on scoped tests**: run the Step 0 test command narrowed to the touched module's tests. If the suite needs its own database, give each branch an isolated one (a per-branch database name keeps concurrent worktrees from colliding) and take the connection string from the project's env file — do not compose one by hand.
  For a resolution that combines features which interact, **also run the affected end-to-end / scenario tests** — module tests miss cross-feature breaks.
- Verify no leftover markers: `grep -rn '<<<<<<<' -- . | grep -v '^\.git/'`. Then `git add -A && git commit --no-edit && git push origin <branch>` — a regular push; the merge added commits, so no force is needed.
- **Wait for green CI** (`gh pr checks <pr> --watch`), then `gh pr merge <pr> --merge`. Confirm the base head advanced as expected.
- After each merge, re-check the remaining siblings' `mergeStateStatus` (GitHub recomputes asynchronously — give it ~20–30s); re-resolve any that flipped to `DIRTY`.

**Parallelize the independent resolutions** with subagents — one per non-colliding PR (reset → merge base in → resolve → scoped test → push) — then merge them serially as CI greens. A mid-tier coding model is the right class for a resolution worker; reserve your own judgement for semantic conflicts and the order itself. Keep colliding groups **and the migration chain** sequential.

## Step 4 — Verify & report
- The base branch has a **single migration head** (linear chain) — verified with the tool, not a regex.
- Report what merged, in what order, which conflicts were semantic (and how they were combined), the UNSTABLE verdicts from Step 1.5, and anything left open (bounced or deferred PRs). Report everything deferred — no silent drops.
- **Closing the logbook items:** if the plugin's `webhookWired` setting is true, don't touch them — the merge webhook crosses them off and writes the resolution stub. If it is false, the merge would otherwise never be recorded, so `update_items` the merged batch to `done` with a resolution citing the PR number and merge commit.

## Anti-patterns (don't)
- ❌ Re-point every migration's predecessor upfront, then merge → red CI on branches whose predecessors aren't on the base branch yet.
- ❌ `rebase` branches onto each other to "stack" them → cascading conflicts in the shared registry/model files, plus risky force-pushes.
- ❌ Force-push a build worktree's stale local branch → silently drops fix-round commits.
- ❌ Trust a stale `mergeStateStatus=CLEAN` read → always let `gh pr merge` (GitHub) be the merge-time arbiter.
- ❌ Merge on green module tests alone → a cross-feature or full-suite break sails through; wait for CI.
- ❌ Treat an `UNSTABLE` PR as broken-and-skip OR ready-and-merge without checking *where* the red is (Step 1.5) → you either drop a mergeable PR or land a real break. Cross the failing tests against the PR's own diff first.
- ❌ Write a database URL, password, or absolute machine path into a command or a note → take them from the environment.
