# Logbook item template

Paste this into the item `description` (markdown) and fill every section. Each section header notes *what it prevents* — if a section truly doesn't apply, write "N/A — <why>" rather than deleting it, so the builder knows it was considered, not forgotten.

---

## Size / risk tier
<!-- One of: S (simple — single module, no schema/migration, no shared core) |
     M (moderate — 1–2 modules, maybe one migration) |
     L (large — new module, schema design, cross-module, or security/auth-touching) |
     XL (major overhaul — architectural / broad refactor / core rework).
     The logbook:loop orchestrator reads this line to enforce per-tier batch caps
     (roughly S<=15, M<=10, L<=4-5 per run) and to REFUSE XL items (recommend
     engineer-in-the-loop). Be honest — under-tiering is how an unattended run
     bites off more than it can finish.
     This must match the `size` argument you pass to create_item, which is
     REQUIRED and validated (S | M | L | XL). -->
**Tier:** <S | M | L | XL>  — <one line of rationale>

## Problem
<!-- THE load-bearing field. The first principle of a good brief is "what problem
     are you trying to solve?" — answer it here crisply: what is broken / missing /
     needed, and who feels the pain. The builder uses it for judgment calls the
     acceptance criteria don't cover, and the build COMPLETION NOTE will restate and
     answer this exact problem. A spec whose Problem is vague produces a completion
     note that can't prove the change worked. -->

## Context & background
<!-- Links and prior art: related logbook items (LOG-NNN), the PR/commit that
     introduced the issue, the relevant section of the project's agent instructions
     (CLAUDE.md / AGENTS.md) or contributor docs, any design doc.
     Prevents the builder re-deriving history you already know. -->

## Acceptance criteria
<!-- Checkable bullets. Each becomes a task on the item. "Returns 404, not 403,
     for an id belonging to another account" — not "handle permissions correctly".
     The reviewer renders READY/CHANGES against THIS list, so vague = unverifiable. -->
- [ ]
- [ ]

## Affected modules & files
<!-- Name the boundary and the specific files you expect to change, using this
     repo's actual layout (read it — don't import a layout from another project).
     Turns a search problem into a lookup. If you're unsure, say "likely X, confirm".
     Call out any high-collision shared file (central models/schema, a route or
     event registry, a top-level architecture doc) — logbook:plan clusters on this
     footprint to avoid merge collisions. -->

## Test plan
<!-- Name BOTH:
     - The scoped tests to run in the inner loop (the narrowest path that covers
       the change).
     - The test(s) to add or extend, with intended file names.
     Derive the command instead of guessing: use the configured `testCommand`
     plugin setting if set; if blank, detect it from the repo — the package
     manifest's script block, pyproject/Makefile/justfile, or the CI workflow's
     job steps — and prefer its scoped form.
     If the change touches shared core code or adds a migration, note that the
     FULL suite is required in the inner loop. -->

## Authorization / isolation considerations
<!-- Only if this project documents an isolation, tenancy, or authorization gate
     in its agent instructions (CLAUDE.md / AGENTS.md) or contributor docs.
     If so, answer it explicitly and cite the specific rule: which routes, which
     identity dependency, what a cross-boundary lookup must return, whether any
     response field must be withheld.
     If the project documents no such gate, write "N/A — no documented
     authorization gate in this repo" rather than inventing one. Guessed security
     rules are worse than none. -->

## Docs to update
<!-- Same PR as the code. Check the project's own docs policy (its agent
     instructions file usually names the hard gates) rather than assuming a set.
     Typically: an architecture/overview doc if structure changed, the module's
     own README if its surface changed, and the logbook resolution note on
     completion. Note any doc or index the project marks as frozen/generated. -->

## Migration notes
<!-- If a schema change is involved: name the migration tool this repo uses, check
     for chained/conflicting heads before writing one, and note any invariant the
     schema must preserve (mirrored columns, backfill order, rollback path).
     N/A if no schema change. -->

## Out of scope
<!-- What this item explicitly does NOT cover, so the builder doesn't scope-creep
     and the reviewer doesn't flag missing-but-intentional gaps. -->
