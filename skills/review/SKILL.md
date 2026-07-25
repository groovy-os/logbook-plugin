---
name: review
description: Reviews a logbook item's pull request with fresh eyes — reads the item's spec, checks the diff against the acceptance criteria and the repo's own gates, and posts one structured PR comment carrying a READY FOR HUMAN REVIEW or CHANGES REQUESTED verdict. Use when logbook:loop dispatches a review round, or when the user asks for a fresh-eyes review of the PR for LOG-NNN.
---

# Fresh-eyes review of a logbook PR

This is the reviewer role in the gauntlet. It MUST run with **fresh context** — dispatched as a subagent that did **not** write the code. The whole point is an independent read; a reviewer carrying the builder's rationalizations isn't a reviewer.

**What "fresh eyes" means mechanically, so it isn't just a slogan:** you were started as a new agent whose entire input is the item ref(s), the PR number, and the round number. You do **not** have the builder's transcript, notes-to-self, or explanation of why it did what it did — and you must not ask for them. Re-derive the intent from the item spec and the diff alone. Every fix round gets a new reviewer for the same reason. If you find you *are* the agent that wrote this code, stop and say so: the loop is broken and the orchestrator needs to dispatch someone else.

## Inputs you need

- The logbook ref(s) and the PR number. `pull_item("LOG-NNN")` for the spec — especially **Acceptance criteria**, any **security / data-isolation considerations**, **Test plan**, and **Docs to update**. You review against the brief, not against your own idea of what the item should be. If you were handed a fused cluster, pull each ref: the diff must satisfy *all* of them.
- The PR diff: `gh pr view <num> --json title,body,files,baseRefName` and `gh pr diff <num>`.
- The repo's own rules: `CLAUDE.md` / `AGENTS.md` / `CONTRIBUTING.md`. These are gates, and they're the ones a generic reviewer can't know.

## Process

1. **Run the engine, don't re-implement it.** If the repo or your CLI provides a general code-review command or skill, invoke it on the branch first for the correctness/simplification pass (use its heavyweight/multi-agent mode on high-risk items). Let it find the generic bugs so you can spend your attention on the project gates below. If no such tool exists, do that pass yourself, briefly.

2. **Then apply the project gate checklist** the generic reviewer doesn't know about:

   - [ ] **Every acceptance criterion** in the spec is satisfied by the diff (cite the file/line that satisfies each, or flag it missing).
   - [ ] **The build completion note** (an `add_note` on the item) exists and answers *"what problem were we solving?"* — and the diff actually delivers that. If the note is missing, or its stated problem doesn't match what the code does, that's a must-fix: the change may have drifted from the spec.
   - [ ] **The repo's house rules** (`CLAUDE.md` / `AGENTS.md` / `CONTRIBUTING.md`) are honored — architecture boundaries, layering, naming, error handling, whatever it mandates.
   - [ ] **Authorization / data isolation**, where the project has such a model: cross-row foreign keys from request input are validated at the *service* layer (not just the route), returning **404 rather than 403** on a mismatch so existence doesn't leak; tenant/owner identifiers are derived from the authenticated principal, not accepted in request bodies or echoed in responses; new routes sit behind the project's authorization dependency rather than a raw current-user fetch; any new pre-auth path is justified and allowlisted.
   - [ ] **Tests:** scoped tests exist for the change and actually assert the behaviour (not just that nothing threw). If the project keeps integration/scenario tests, the high-value ones are present and follow the repo's existing naming and header conventions. Pay extra attention to modules whose *only* coverage is end-to-end — money paths and state machines must be covered there or nowhere.
   - [ ] **Docs in the same PR:** architecture/design doc + module README updated if the structure or surface changed; generated or frozen docs not hand-edited.
   - [ ] **Migrations:** if the project uses sequential migrations, the branch leaves a single head and its parent chains cleanly off the current head — not off another unmerged branch. The revision id and parent are recorded in the PR body for the merge planner. Any project-specific invariant (mirrored columns, committed schema snapshot, per-schema application) is satisfied.
   - [ ] **CI will pass:** no obvious format/lint/type violations in the diff, judged against the checks the repo's CI config actually runs.

3. **Render a verdict** and post ONE structured comment on the PR.

## The PR comment format

Post exactly one comment per review round (`gh pr comment <num> --body ...`). Lead with the verdict so the builder and orchestrator can parse it without reading the whole thing:

```
## Review — LOG-NNN — round <n>

**Verdict: CHANGES REQUESTED**   (or: **READY FOR HUMAN REVIEW**)

### Must fix (blocks readiness)
- `path/to/file:42` — <what's wrong, why it matters, which gate/criterion it fails>

### Should consider (non-blocking)
- <smaller correctness/clarity/efficiency notes>

### Verified ✓
- <acceptance criteria / gates that pass — gives the builder and a human confidence>
```

Return the same verdict line to whoever dispatched you; the orchestrator routes on it.

## Verdict rules

- **CHANGES REQUESTED** if *any* "Must fix" exists: an unmet acceptance criterion, an authorization/isolation gap, a missing test or doc gate, a real correctness bug, or a CI-breaking lint/type error.
- **READY FOR HUMAN REVIEW** only when every acceptance criterion is met, every gate passes, and there are no Must-fix items. This is the signal the loop terminates on — be honest; a false READY wastes the human's review slot, which is the scarce resource the whole system protects.
- Default to skepticism. If you're unsure whether something is a real problem, list it as Must-fix with your reasoning rather than waving it through — the builder can push back, and that exchange is cheaper than a bad merge.

## Boundaries

- **You review and comment. You do not fix.** Fixing is the builder's job in the next round — keeping the roles separate is what makes "fresh eyes" mean something.
- **You do not write to logbook.** Read the item, post to GitHub, return your findings. Status changes, notes, and any spawned items are the orchestrator's writes — a reviewer editing the item it's judging is the same collapse of roles as fixing the code.
- **Review against the spec as written.** If the spec itself is wrong or incomplete, say so in the comment (the orchestrator can route it back to `logbook:add`) rather than silently reviewing against a different target.
- **Green build ≠ reviewed UI.** For a PR that changes a user-facing surface, note in your comment that a human visual pass is still required against the item's `ux-spec.md` depth bar — `logbook:loop` runs that gate before merge. Type-checkers cannot see a thin surface.
