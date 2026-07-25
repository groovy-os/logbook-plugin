# The seven principles of UX depth

These are the bar every front-end logbook item is held to. They exist because teams keep
shipping **thin v0 surfaces** — a button per endpoint, a rectangle per record, no states,
no hierarchy, no product thinking. Each principle below names the failure mode it prevents,
with a worked example.

The gauntlet in `SKILL.md` is just these seven principles turned into questions. The spec you
write is the answers. A reviewer should be able to hold the built surface against each
principle and say "yes" — if they can't, the spec wasn't deep enough.

Where a principle says "cite the project's canon", it means *this* repo's design docs,
patterns, and primitives — found the way Step 2 of `SKILL.md` describes. If the project has
none, cite an exemplar surface from the codebase instead. Never cite a doc that doesn't exist.

---

## 1. Design the job, not the endpoint

> Start from **what the user is trying to accomplish**, never from the API surface.

The single most common failure. A spec that reads `create / link-order / submit round /
record verdict / approve|reject` is a list of backend routes wearing buttons. The builder
renders exactly that: one button per endpoint, in endpoint order, with endpoint names.

**The tell:** a sample-approval tab that shipped with `Submit to buyer`, `Verdict: approve`,
`Approve w/ comments`, and `Verdict: reject` as four sibling buttons on an empty round —
because the brief was the `/rounds/{id}/submit` and `/rounds/{id}/verdict` endpoints, not the
*job*. The real job is **"get a sample approved by the buyer so the order can be accepted"** —
a multi-step, multi-day collaboration with photos, comments, iterations, and a clear "what's
blocking acceptance right now" answer. None of that is an endpoint; all of it is the job.

**How to apply:** write the job as a sentence — *"As a [role], I'm trying to [outcome] so
that [why it matters]."* Then map the flow that accomplishes it. Endpoints are an
*implementation note at the bottom of the spec*, never the structure of it.

---

## 2. Every state is a screen

> Empty, loading, partial, error, zero / one / many, and no-permission are **not edge
> cases — they are the surface.** Design each one.

Thin surfaces only design the happy, populated path. But a user's *first* encounter with any
feature is the **empty state**, and that's where you teach them what it's for and how to
start. A feature with no empty state is a feature nobody can onboard onto.

**The tell:** four grey image placeholders and a `—` where a record's content should be. That's
the populated component rendering *nothing*, because the empty/pending state was never
designed.

**How to apply:** for every surface, enumerate and spec all of these that apply:
- **Empty** — no records yet. This is the onboarding screen: explain the feature, show the
  primary action prominently, set expectations. Never a blank panel.
- **Loading** — use the project's skeleton pattern, not a spinner-on-blank.
- **Partial / in-progress** — the most common real state. A record mid-flow, a form
  half-configured. What does "you're 2 of 4 steps in" look like?
- **Zero / one / many** — one record looks different from forty. Does it paginate, group,
  filter, collapse?
- **Error & retry** — use the project's error-state component and retry affordance. A raw
  `Failed to fetch` must never reach the UI.
- **No-permission / blocked** — the action exists but the user or the entity's state can't do
  it yet. Show *why*, don't just hide it (see Principle 5).

---

## 3. Render records, not roll-ups (the data hierarchy IS the design)

> When the underlying data is a **list of records**, render the records. Decide what's
> primary, secondary, and tertiary — that decision *is* the component.

Don't render `2/3 approved` or `4/4 ready` when there's a list behind it; render the list,
with each record's status scannable at a glance. A roll-up is a summary of information the
user came here to act on — it forces a click to recover what you already had.

**The tell:** "a bunch of fancy looking rectangles, no real substance." Rectangles with no
information hierarchy — everything the same weight, nothing scannable, the actual records
(what happened, when, by whom, what changed) hidden behind decoration. A timeline's job is to
answer *"what happened and what do I need to act on"* at a glance; uniform rectangles answer
nothing.

**How to apply:** for the surface, write the **data hierarchy** explicitly:
- What is the **primary** thing the eye should land on? (the record's identity + its status)
- What's **secondary** (scannable on the same row without clicking — the shortfall, the
  "blocks acceptance" flag, the overdue-by)?
- What's **tertiary** (revealed on drill-in)?
- Can the user see *which specific record* is short / pending / overdue / blocked **without
  navigating away**? If not, you've built a roll-up.

If the project documents a granularity rule, cite its section; the rule above is the floor
either way.

---

## 4. Progressive disclosure — right altitude first, drill for depth

> Show the right amount at each level. A surface with one button is as wrong as one with
> forty. Lead with the answer; let the user drill into detail.

Depth does **not** mean "dump every field and action on one screen." It means a deliberate
ladder: summary → record → detail → action. The thin-surface failure and the
everything-at-once failure are the same mistake — no altitude decision was made.

**How to apply:** decide the levels. For an approvals surface: the tab answers "which items
block acceptance" (altitude 1) → each item shows its current round + status (altitude 2) →
expanding an item shows its round history, photos, comments (altitude 3) → acting on a round
opens the focused submit/verdict flow (altitude 4). Each level earns the next click by
answering a question. Use the project's detail/list page template when there's genuine depth,
rather than inventing a layout.

---

## 5. Affordances must be legible and honest

> Every action says **what it does** and **when it's allowed**. Preconditions gate
> availability. Irreversible actions confirm. No button appears before it can be used.

**The tell:** `Submit to buyer` AND `Verdict: approve` AND `Approve w/ comments` AND
`Verdict: reject` all live and clickable on a round that was *just created and has no
content*. Why is there an approve button **before** the thing has been sent? That's a state
machine's full transition table dumped onto the screen as sibling buttons — dishonest,
because most of them can't validly fire yet.

**How to apply:** for every action, spec:
- **Precondition** — what state must the entity be in for this to be valid? (Can't record a
  verdict before a round is submitted.) Disable or hide accordingly, and when disabled, say
  why on hover or inline.
- **Placement** — primary action prominent; destructive/secondary de-emphasized. One clear
  primary per context, not four equal-weight buttons.
- **Confirmation** — irreversible or outward-facing actions (submit *to an external party*,
  reject, delete) confirm first. "Submit to buyer" sends something to someone outside the
  org — that's not a one-click-no-undo button.
- **Feedback** — optimistic update or pending state, plus the project's toast/notification
  pattern on success and failure. The user must know it worked.

---

## 6. Words are UI

> Microcopy, labels, button verbs, and empty-state guidance carry as much product logic as
> components. Write them like a human, not like an enum.

**The tell:** `Verdict: approve`. That's a serialized backend status (`verdict="approve"`)
leaked straight onto a button face. A human would write **"Approve sample"** or **"Mark
approved."** Every leaked enum (`in_progress`, `PROTO`, `1 iter`, `no verdict`) is a place
where product thinking was skipped.

**How to apply:** the spec names the real copy for:
- **Button verbs** — action + object: "Submit to buyer", "Request changes", "Approve sample".
  Never bare enums, never "Submit" alone.
- **Empty-state copy** — one line of what this is + how to start.
- **Status labels** — human phrasing for each backend state (`in_progress` → "In progress",
  `awaiting_buyer` → "Waiting on buyer").
- **Error messages** — what happened + what to do, not "Failed to load X".
- **Helper text** — the one sentence that prevents a support question.

---

## 7. Earn the feature

> State **why this surface exists** and what makes a user choose it over a spreadsheet, an
> email, or a chat thread. If you can't answer that, the spec isn't done.

The first question of any brief is *"what problem are you trying to solve?"* A UX spec that
can't say why the feature beats the status quo will produce a surface nobody adopts — a worse
version of the email thread they already use.

**How to apply:** close the spec with the **"why this beats the workaround"** line. For an
approvals surface: *"Today buyers and suppliers track approval rounds over email and chat
photos, and nobody can answer 'is this order blocked on an approval?' without asking three
people. This surface makes the blocking answer a glance and the next action one click."* That
sentence is the depth bar — every flow, state, and affordance in the spec should serve it.

---

## The depth bar (use this as the reviewer's checklist)

A front-end item is **ready to build** only when its spec answers all of these:

- [ ] **Job** — stated as an outcome, not an endpoint list (P1)
- [ ] **Users & context** — who, when, how often, how urgent, what device (P1)
- [ ] **Flows** — the happy path *and* the branches, step by step (P1, P4)
- [ ] **States** — empty / loading / partial / error / zero-one-many / no-permission all
      specced (P2)
- [ ] **Data hierarchy** — primary / secondary / tertiary decided; records rendered, not
      roll-ups (P3)
- [ ] **Altitude ladder** — summary → record → detail → action; no dump, no one-button
      surface (P4)
- [ ] **Affordances** — every action has a precondition, placement, confirmation rule, and
      feedback (P5)
- [ ] **Copy** — real button verbs, empty-state text, status labels, errors — no leaked
      enums (P6)
- [ ] **Why it beats the workaround** — one sentence (P7)
- [ ] **Reuses the canon** — names the actual template, patterns, and primitives it composes
      from, by path; or, when the project documents none, names the exemplar surface it
      matches and says so explicitly

If any box is empty, send it back through the gauntlet before it goes to `logbook:build`.
