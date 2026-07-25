---
name: explore
description: Investigates a logbook item before it gets built — does the same problem exist elsewhere in the codebase (other pages, modules, endpoints, similar code paths), and is the item's proposed fix still accurate against the current code? Treats the item's problem as a hypothesis, researches the blast radius, then either files batchable sibling items for the other occurrences, corrects the brief, or certifies it isolated with a note that shows its work. Use when the user says "explore LOG-NNN", "does this happen elsewhere", "is this widespread", "check the other pages for this", "validate the fix for LOG-NNN", or "/logbook:explore". Read-heavy research — it does not write code or open PRs.
argument-hint: LOG-NNN [hint about where else to look, e.g. "check the other dashboard pages"]
---

# Exploring a logbook item: blast radius + fix validation

A logbook item describes one observed problem. But the same *shape* of bug often recurs — the missing guard that's absent on five other routes, the anti-pattern copied across eight pages, the stale assumption baked into a dozen callers. And the item's proposed fix was often written against code that has since moved. `logbook:explore` is the investigation stage that answers two questions before anyone builds:

1. **Does this problem exist elsewhere?** (blast radius / recurrence)
2. **Is the proposed fix accurate against the *current* code?** (brief validation)

The output is one of three outcomes — file batchable siblings, correct the brief, or certify isolated — always recorded back on the item so the work isn't repeated.

## When to use this

- The user suspects an item is the tip of an iceberg ("I think LOG-429 happens on other pages too").
- Before a build, to confirm an item's scope and that its evidence/line-refs still hold.
- Mid-`logbook:loop`, when a builder smells that the problem it's fixing is one instance of a wider pattern.

## The investigation process

1. **Pull the item and extract the problem SIGNATURE.** `pull_item(LOG-NNN)`. Read the problem, evidence, and proposed fix. Then name the *shape* of the bug in a form you can search for — the missing guard, the called function, the anti-pattern, the convention that's violated. "InboxPage hardcodes the badge count" → signature = "components reading a hardcoded count instead of the live client." A vague signature produces a vague search; sharpen it first.

2. **Form hypotheses about where else it could live.** Think in the codebase's own structure: sibling files (other pages/components in the same dir), analogous layers (other routes/services with the same responsibility), callers of the same function, the same pattern in a different module. Write down the candidate surfaces before searching, so "I found nothing" later means "I checked these specific places," not "I glanced."

3. **Search multiple angles — breadth matters here.** A single grep is not exploration. Hit it from several directions: by symbol (Grep the function/import), by pattern (the anti-pattern's structural signature), by sibling convention (Glob the peer files and Read them), by analogous feature. **For a large surface (e.g. "every page" / "every route"), fan out parallel `Explore` agents by area** — each blind to the others — rather than one serial sweep. Keep a running list of *what you searched and where*.

4. **Validate the proposed fix against current code.** Open the files the item cites. Do the line refs still point at the right thing? Does the recommended approach still apply, or has the code moved (a guard already added, a refactor that changes the fix)? Compare against the current state of the base branch — detect it rather than assuming: use the configured `baseBranch` plugin setting, or resolve it with `git symbolic-ref refs/remotes/origin/HEAD`. Briefs drift; catching that here saves a whole build/review cycle.

## The outcome — always record it (these can combine)

**A. Widespread — the problem recurs.** For each *genuinely new* occurrence:
- **Dedup first** (search open + recently-closed items with `pull_items(search: ...)` / `list_backlog` — reuse the `logbook:add` discipline) so you don't file a twin of something already tracked.
- **File it with the full `logbook:add` discipline** — meaty problem, size/risk tier, affected files, test plan, tags from the live taxonomy (`list_labels`, reuse over invent). A sibling item that's a thin one-liner just moves the scoping problem downstream. Use `create_items` for the batch, and remember `size` is required and `summary` is what every index row shows.
- **Link the cluster** with `link_items` so the relationship is visible, and put a **shared label** on them so `logbook:loop --tag <that>` can batch the whole family in one run.
- **Update the original item** with `add_note` summarizing the blast radius: "Explored — found N other occurrences (LOG-aaa, LOG-bbb…), filed and linked; suggest batching under tag `<x>`."

**B. Brief is inaccurate — the problem is real but the fix/evidence is stale.** `update_item` to correct the evidence (current line refs) and the recommended approach, and `add_note` explaining what changed and why, so the builder gets the corrected brief.

**C. Isolated & validated — found nothing else, fix checks out.** `add_note` a **trustworthy negative**: state the conclusion AND how you reached it — "Explored — confirmed isolated. Searched: <signatures/symbols>, across <surfaces/areas>; the proposed fix at <file:line> still applies. No sibling items needed." A negative result is only useful if a human can judge that the search was thorough — so document the search, never just "all good."

## Rigor rules

- **A "found nothing" must show its work.** Document the signatures searched and the surfaces covered. A shallow search that certifies "isolated" is worse than no exploration — it creates false confidence and stops anyone looking again.
- **Be adversarial about isolation.** Try at least two or three independent search angles before concluding the problem is a one-off; one weak match is not "widespread," and one grep returning nothing is not "isolated."
- **Dedup before you create** — same threat as `logbook:add`: don't spawn duplicate or near-twin items.
- **Don't silently file XL work.** If exploration reveals the problem is actually a major overhaul (architectural, broad refactor), say so on the original item and recommend engineer-in-the-loop — don't quietly mint a giant item the gauntlet would refuse anyway.
- **Read this repo's conventions, don't import them.** Layout, test command, review gates and any documented authorization rules differ per project. Ground the investigation in this repo's agent instructions (`CLAUDE.md` / `AGENTS.md`) and its actual file tree.
- **You investigate and report. You do not write code or open PRs.** Your deliverables are logbook items, edits, notes, and links — the build comes later via `logbook:loop`.

## Reporting back

Tell the user, in plain prose, which outcome you reached, the refs you created or updated, and the search you actually performed. The note on the item is the durable record; the message to the user is the summary.

## Composition

- **Creating siblings** → follow the `logbook:add` skill's discipline (dedup, tier, tag, meaty brief).
- **Breadth search** → dispatch parallel `Explore` agents by area; each returns occurrences, you synthesize and dedup.
- **Then** → the resulting cluster feeds `logbook:plan` for batching and `logbook:loop --tag <shared>` to build them together.
