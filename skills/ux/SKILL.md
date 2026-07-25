---
name: ux
description: Specs the UX/UI depth of a front-end logbook item BEFORE it is built — inverts a thin, endpoint-shaped brief into a real product surface (job-to-be-done, flows, every state, data hierarchy, affordances, real microcopy) and attaches that spec to the item as `ux-spec.md` so the build has depth instead of shipping a v0. Use when the user types "/logbook:ux", says "spec the UX for LOG-NNN", or flags a front-end item as too thin to build. Also fires as the hard gate from logbook:loop and logbook:build — they block any front-end-labelled item whose documents carry no `ux-spec.md` and run this first.
argument-hint: LOG-NNN (a front-end logbook item to spec)
---

# The UX depth gauntlet

Front-end logbook items keep shipping as **thin v0 surfaces** — a button per endpoint, a
rectangle per record, no empty states, no hierarchy, no product thinking. The cause is
upstream: the *brief* describes the API surface, so the builder honestly renders the API
surface. Four equal-weight buttons named after four routes, live on an empty record, is not
a build failure — it is a brief failure.

This skill fixes it at the source. It runs the item through a **gauntlet of UX questions**,
producing a spec deep enough that `logbook:build` builds a product people will actually use.
**We accept this takes longer** — depth is the goal, not speed. A 3× longer spec pass that
yields an adopted feature beats a fast pass that yields another dead v0.

> This is to front-end items what `logbook:add` is to all items: the thinking a human would
> otherwise do mid-build, done up front and written into the item. `logbook:add` makes the
> item *buildable*; `logbook:ux` makes the front-end surface *good*.

## When this runs

- **The hard gate (primary path).** `logbook:loop` and `logbook:build` block any item
  carrying the **`front-end`** label whose `documents` array (from `pull_item`) has **no
  `ux-spec.md` attachment**. They invoke this skill first. **Do not build a front-end item
  without a UX spec.**
- **On request.** `/logbook:ux LOG-NNN`, "spec the UX for LOG-NNN", or the user flags an item.

**Re-running is safe and expected.** `attach_document` takes a `replace` flag that defaults
to `true`, so re-running the gauntlet on a log overwrites the existing `ux-spec.md` cleanly
rather than accumulating duplicates. If a spec is thin, run it again — don't work around it.

**Scope check first.** This is for items that render a **surface** (a page, tab, panel,
drawer, flow). If a `front-end`-labelled item is genuinely chrome-only — a copy tweak, a
color fix, a prop rename, a dependency bump — it doesn't need the full gauntlet. Attach a
one-line `ux-spec.md` of `N/A — <why: e.g. copy-only fix, no new surface>` (see Step 5) so
the gate is satisfied, and move on. Don't ceremonially spec a one-line fix; don't wave a real
surface through.

## The seven principles

The whole gauntlet is these turned into questions — read **`references/ux-principles.md`** in
full; it carries each principle's failure mode and a worked example. In brief:

1. **Design the job, not the endpoint.** Start from the user's outcome; endpoints are an appendix.
2. **Every state is a screen.** Empty / loading / partial / error / zero-one-many / no-permission.
3. **Render records, not roll-ups.** The data hierarchy IS the design.
4. **Progressive disclosure.** Right altitude first; one button is as wrong as forty.
5. **Affordances must be legible & honest.** Preconditions gate availability; outward/irreversible actions confirm.
6. **Words are UI.** Real button verbs and copy — no leaked enums like `Verdict: approve`.
7. **Earn the feature.** State why it beats the spreadsheet/email/chat thread it replaces.

## Process

Create a TodoWrite todo per step for anything non-trivial.

### 1. Pull the item & read the existing surface
- `pull_item("LOG-NNN")` — get the problem, description, tasks, labels. Confirm it's
  `front-end`-labelled and surface-shaped (see scope check above).
- If the item already has documents, `get_document` the relevant ones — an existing
  `ux-spec.md` is what you're revising, not something to duplicate.
- **Read the existing component.** The item's brief usually names a file. Read it and the
  data/API module it calls. You're looking for what exists and where the depth gap is — the
  thin happy path with no states, the endpoint-shaped buttons, the roll-up hiding a list.

### 2. Ground in the project's design canon (don't reinvent the design system)

**Find the canon before you write; never import one from another codebase.** Look, in order,
and stop when you have something concrete:

1. **What the repo's own agent instructions point at.** Read `CLAUDE.md` / `AGENTS.md` — the
   root one and any nested one living next to the front-end code. If they name design docs,
   page templates, a pattern library, or component rules, that is the authoritative canon.
2. **Conventional locations.** A `docs/design/`, `design/`, or `front-end/docs/` directory; a
   `DESIGN.md` or `STYLEGUIDE.md`; a design-system package's README; a Storybook config.
3. **The code as de facto canon.** The shared UI primitives directory (error states, data-
   fetching hooks, badges, skeletons, tables) and one or two **exemplar surfaces** — sibling
   pages in this repo that already have real depth.

**Degrade gracefully.** If the project has no documented design canon, say so in the spec
(`Canon: none documented — matched <exemplar file> instead`) and derive the bar from the best
existing surfaces you can find. Do **not** invent a house style, cite doc paths that don't
exist, or assert layout policy this repo has never adopted. An undocumented convention
guessed at is worse than an honestly-named exemplar.

Whatever you find, **name it by exact path in the spec** — the template, the pattern, the
primitives, the exemplar. Naming the reusables is what keeps the build on-canon instead of
freelancing a second design system.

### 3. (Optional) Inspect the live surface
When seeing the real thing would sharpen the spec — and the app is running — drive a browser
to screenshot the current surface and name the depth gap concretely (this is the "these are
just rectangles" instinct, made evidence). Screenshot the populated AND empty states and
capture them as "here's the v0 / here's what depth it's missing." Skip when the gap is
already obvious from code, or the app isn't up — don't rabbit-hole on getting the app
running.

### 4. Run the gauntlet — answer every question
Work `references/ux-spec-template.md` top to bottom. For each section, *answer the question*,
don't hand-wave. The high-leverage moves:
- **Invert the brief.** Whatever endpoints the item listed, push them to the *bottom* (Backend
  notes) and write the **job** at the top. This single inversion is most of the value.
- **Draw the wireframe.** ASCII the populated state. The builder builds what it can see; prose
  alone produces another thin guess.
- **Enumerate states.** Literally list empty / loading / partial / error / zero-one-many /
  no-permission and spec each — this is where v0s die.
- **Decide the hierarchy.** Primary / secondary / tertiary, records-not-roll-ups.
- **Tabulate affordances.** Every action: precondition, placement, confirm?, feedback. Kill any
  button that can appear before its precondition is met.
- **Write real copy.** Replace every leaked enum with words a human wrote.
- **Flag backend gaps.** If the UX needs data or an endpoint the API doesn't provide (e.g. an
  attachment upload), say so — and file a backend logbook item via `logbook:add` rather than
  letting the build discover it cold.

### 5. Write the spec into the item (as an attached document)
The spec lives **on the logbook item** (fetched via MCP at build time) — not in a repo `.md`
(off-branch docs rot). The long-form spec goes in a **document attachment**, not the
`description` — that keeps `list_backlog` / `pull_item` payloads small while the full spec
stays one fetch away. Four writes:

- **`attach_document(number, "ux-spec.md", content)`** — the filled spec template (the whole
  `## UX/UI Spec` body) as the document `content`. **`ux-spec.md` is the stable filename and
  the gate marker** — its presence in the item's `documents` array is what `logbook:loop` /
  `logbook:build` check. Use exactly `ux-spec.md` so `get_document` resolves it unambiguously.
  `replace` defaults to `true`, so re-running simply overwrites the previous spec — one
  document, one filename, no duplicates.
- **`update_item`** — keep the `description` short: append a one-line pointer + the job, e.g.
  *"UX spec attached as `ux-spec.md` — job: <one line>. Build to its Depth bar, not the
  endpoints."* Do **not** paste the full spec into the description (that's the document's job).
  Append; don't clobber the original problem statement.
- **`add_task`** — add one depth task per affordance/state/section the build must satisfy
  (e.g. "Empty state with onboarding copy + primary action", "Attachment upload + dropzone",
  "Disable verdict actions until the round is submitted", "Replace 'Verdict: approve' with
  'Approve sample'"). These become the checklist the builder works down and the reviewer holds
  the UI to. Keep the original endpoint tasks if accurate, but they're now subordinate to the
  depth tasks.
- **`add_note`** — a short note: "UX spec attached (`ux-spec.md`) — built depth from the job
  (<one line>). Builder: `get_document` it, then satisfy the depth tasks + the Depth bar
  checklist, not just the endpoints."

If a sibling backend gap was found and filed, put that item's ref in the note too (and
`link_items` the two).

**Chrome-only fixes still attach a marker.** For genuine chrome-only work (copy/color/prop/dep
fix), `attach_document(number, "ux-spec.md", "N/A — <why: e.g. copy-only fix, no new surface>")`
— a one-line doc that satisfies the gate uniformly. Don't skip the attach; the gate keys off
the document's presence, so an un-attached item reads as "not yet specced".

### 6. Hand off
The item is now a real build brief. If invoked under the gate, return control to
`logbook:loop` / `logbook:build` — it proceeds to build with the enriched item. If invoked
standalone, tell the user the spec is in, summarize the job + the depth it now demands, and
offer to kick off the build.

## The depth bar (don't hand off until these pass)

Hold the spec against `ux-principles.md`'s depth bar. The item is ready only when the spec
answers: job (not endpoints), users & context, flows (happy + branches), all states, data
hierarchy, altitude ladder, affordances (precondition / confirm / feedback), real copy, "beats
the workaround", and named canon reuse (or an honestly-named exemplar when there is no
documented canon). If any is empty, keep going — that's the whole point of running 3× longer.

## Anti-patterns

- **Speccing from the endpoint list.** If your spec's structure mirrors the API routes, you've
  rebuilt the thin brief with more words. Invert it — job on top, endpoints at the bottom.
- **"Happy path only."** Skipping the empty/error/partial states is exactly how v0s ship. They
  are the surface, not edge cases.
- **Roll-ups over records.** `2/3 approved` when there's a list behind it hides the record the
  user needs to act on. Render the list.
- **Leaked enums as copy.** `in_progress`, `Verdict: approve`, `PROTO · no verdict` on the
  screen = product thinking skipped.
- **Freelancing the design system.** Not naming the template/patterns/primitives → the builder
  reinvents (badly). Compose from whatever canon this repo actually has.
- **Inventing canon that doesn't exist.** Citing a design doc path this repo doesn't contain,
  or importing another project's layout policy as if it were universal, sends the builder
  chasing ghosts. Name real files or name an exemplar.
- **Ceremony on chrome.** Don't run the full gauntlet on a copy/color/prop fix — `N/A` the gate
  and move on. Depth where it matters, not everywhere.

## Related skills

`logbook:add` (file the backend gap you discover as its own item), `logbook:build` (consumes
this spec), `logbook:review` (fresh-eyes review against the depth bar), `logbook:loop` (drives
the gate), `logbook:explore` (is the depth gap wider than one item?), `logbook:plan`
(batching), `logbook:merge`, `logbook:sweep`.
