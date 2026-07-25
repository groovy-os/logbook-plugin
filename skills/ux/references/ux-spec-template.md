# UX/UI spec template

This is what the gauntlet produces and attaches to the logbook item as a document
(`attach_document(number, "ux-spec.md", <this filled body>)`) — plus the depth tasks it adds
and a one-line pointer in the item's `description`. Fill **every** section; if one genuinely
doesn't apply, write `N/A — <why>` so the builder knows it was considered, not forgotten. Each
section maps to a principle in `ux-principles.md`.

`attach_document` replaces by default, so re-attaching a revised spec under the same filename
is the correct way to iterate — never a second document.

Keep it concrete. ASCII wireframes beat prose — the builder builds what it can see.

---

~~~markdown
## UX/UI Spec

<!-- Written by /logbook:ux on <date>. This is the build contract for the surface.
     The reviewer holds the built UI against the depth bar at the bottom. -->

### Job to be done   <!-- P1 -->
As a **<role>**, I'm trying to **<outcome>** so that **<why it matters>**.
This surface is NOT "the UI for the <X> endpoints" — it's how the user does <job>.

### Users & context of use   <!-- P1 -->
- **Who:** <roles — operator / customer / admin / reviewer>
- **When & how often:** <e.g. multiple times a day during pre-production>
- **Urgency:** <is this blocking money/shipment? routine?>
- **Device / surface:** <desktop dashboard / mobile scan / drawer>

### Why this beats the workaround   <!-- P7 -->
Today users do <X> via <spreadsheet / email / chat thread>. This surface wins because <one line>.

### Core flow(s)   <!-- P1, P4 -->
Happy path, step by step:
1. <user lands, sees …>
2. <does …>
3. <result …>
Branches / alternates:
- <rejection path, multi-round path, error path …>

### Information architecture & data hierarchy   <!-- P3 -->
- **Primary (eye lands here):** <record identity + status>
- **Secondary (scannable, same row):** <the actionable signal — "blocks acceptance", "short −131">
- **Tertiary (on drill-in):** <history, photos, comments, metadata>
- **Records, not roll-ups:** the user can see WHICH <record> is <blocked/pending> without
  navigating away.

### Altitude ladder   <!-- P4 -->
L1 <summary — the one question this surface answers> →
L2 <per-record view> →
L3 <record detail> →
L4 <focused action>

### Layout / wireframe   <!-- compose from this repo's canon -->
- **Canon source:** <exact path to the design doc/template used — or "none documented;
  matched <exemplar file>">
- **Template:** <the page shape being used — detail / list / builder / form / dashboard>
- **Patterns:** <the documented patterns this composes — granularity, tabbed sections,
  skeletons, toasts, drawers — cite section refs if the repo has them>
- **Primitives/components to reuse:** <real paths — the error-state component, the fetch hook,
  badges, tables …>

```
+-- ASCII wireframe of the populated state ------------------+
| <draw it — the builder builds what it can see>            |
+-----------------------------------------------------------+
```

### States   <!-- P2 — design each that applies -->
- **Empty:** <onboarding copy + primary action — what the user sees with zero records>
- **Loading:** <skeleton shape>
- **Partial / in-progress:** <the mid-flow state — usually the most common>
- **Zero / one / many:** <how it scales — grouping, pagination, filtering>
- **Error & retry:** <the project's error-state component + retry>
- **No-permission / blocked:** <show why, don't just hide>

### Affordances (actions)   <!-- P5 -->
| Action | Precondition (when valid) | Placement / weight | Confirm? | Feedback |
|---|---|---|---|---|
| <Approve sample> | <round submitted> | <primary> | <no> | <toast + status flip> |
| <Submit to buyer> | <round has content> | <primary> | <yes — outward-facing> | <pending + toast> |

### Copy   <!-- P6 — real words, no leaked enums -->
- **Buttons:** <"Submit to buyer", "Request changes", "Approve sample">
- **Empty state:** <one line>
- **Status labels:** <in_progress → "In progress"; awaiting_buyer → "Waiting on buyer">
- **Errors:** <what happened + what to do>

### Backend / data notes   <!-- implementation note — BOTTOM, not the structure -->
- Endpoints this consumes: <list — these are an appendix, not the design>
- Data the UI needs that the API may not return yet: <flag gaps → file a backend logbook item>

### Depth bar (reviewer checklist)
- [ ] Job stated as outcome, not endpoints
- [ ] All applicable states designed
- [ ] Data hierarchy decided; records not roll-ups
- [ ] Altitude ladder, not a dump or a one-button surface
- [ ] Every action has precondition + confirm + feedback
- [ ] Real copy, no leaked enums
- [ ] "Beats the workaround" line present
- [ ] Reuses named canon (or names the exemplar it matched)
~~~

---

## Worked exemplar — a sample-approval tab, before → after

This is the transformation the skill performs. **Before** is a real shipped brief; **after**
is what the gauntlet would have produced.

### Before (the thin brief that shipped)

> Task checklist:
> 1. Add sample write functions to the API module (create, link-order, create/submit round, record verdict, approve/reject)
> 2. Add create-sample + round + verdict UI to the samples section
> 3. Surface `unapproved_samples`
> 4. Update the component showcase

Every task is an endpoint. Result: one `+ Create & link sample` button → a round with four
grey placeholders, a `—`, and four equal-weight buttons (`Submit to buyer`, `Verdict:
approve`, `Approve w/ comments`, `Verdict: reject`) live on empty content. No empty state, no
photo capture, no comments, no round history, no "which sample blocks acceptance" answer.

### After (gauntlet output, abbreviated)

**Job:** As a **production operator**, I'm trying to **get each linked sample approved by the
buyer** so that **the order clears its acceptance gate and moves to Confirmed.**

**Why it beats the workaround:** today sample rounds live in email and chat photo threads;
nobody can answer "is this order blocked on a sample?" without asking around. This surface
makes the blocking answer a glance and the next action one click.

**Data hierarchy:** primary = each sample with its current round + status; secondary = the
"blocks acceptance" flag inline (which specific sample, which round, why); tertiary (on
expand) = round history, photos, buyer comments. *Records, not `2/3 approved`.*

**Altitude ladder:** L1 banner "1 of 2 samples blocks acceptance — SMP-0003 proto, no
verdict" → L2 a card per sample → L3 expand to round timeline with photos + comments → L4
focused "Submit round to buyer" / "Record verdict" flow.

**States:**
- *Empty:* "No samples linked yet. Link a sample to track buyer approval — required before
  this order can be accepted." + prominent `Link sample`.
- *Partial:* round created but not submitted → shows a photo-upload dropzone + "Submit to
  buyer" as the ONLY primary action (verdict buttons not yet shown — Principle 5).
- *Awaiting buyer:* "Sent to buyer 2d ago · waiting" — no operator action, shows what's
  pending.
- *Verdict in:* now the approve / request-changes affordances appear.

**Affordances (corrected):**
- `Submit to buyer` — precondition: round has ≥1 photo/attachment; **confirms**
  (outward-facing); pending state after.
- `Record verdict` — precondition: round status = submitted. (Not shown before submit.)
- No `Verdict: approve` button face — it reads **"Approve sample"**.

**Copy:** `Verdict: approve` → **"Approve sample"**; `Verdict: reject` → **"Request
changes"**; `in_progress` → "In progress"; the `PROTO · 1 iter · no verdict` enum string →
"Prototype round · awaiting first verdict".

**Backend notes (appendix):** consumes POST /samples, /link-order, /rounds,
/rounds/{id}/submit, /rounds/{id}/verdict, /samples/{id}/approve|reject. *Gap to flag:* round
photo upload — confirm the attachments endpoint exists; if not, file a backend item with
`logbook:add`.

Note how the endpoints moved to the **bottom** and the **job moved to the top**. That
inversion is the whole skill.
