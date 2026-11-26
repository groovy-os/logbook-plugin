# The overview — the short read in front of the brief

The `description` is the thorough spec, and it is written for the agent that builds it. That is deliberate and it stays: the brief may get longer, never thinner. But a person opening the log should not have to read a build spec to learn what it is. The **overview** is what they read first, and what a reviewer holds the result against.

**One source of truth, one derived view.** The spec in `description` is the truth. The overview is derived from it, after it, every time — never written first, never carrying anything the spec does not say. If the two disagree, the spec wins and the overview is wrong.

It is **not** `summary`. `summary` stays the one-line, max-120-char index row it has always been. The overview is its own argument, `overview`, on the same tools.

## Only when the server offers it

Look at the input schema of the tool you are about to call (`create_item`, `create_items`, `update_item`, `update_items`) as it is loaded in this session. If it lists an **`overview`** argument, write one. If it does not — an older server, or one where this has not been deployed yet — **file exactly as you always have and say nothing about it**: no overview pasted into the description, no apology, no mention to the person.

Decide from the schema alone. Never find out by sending the argument and watching what comes back: an unknown argument can be dropped without an error, so a probe proves nothing and the person is told about an overview that was never stored.

## The shape (fixed)

Markdown, in this order, and nothing else:

```
<What this is, in one plain sentence.> <Why it matters and who feels it, in one plain sentence.>

**Done when**
- <acceptance criterion 1, word for word>
- <acceptance criterion 2, word for word>

**Not included:** <what the spec's Out of scope says — leave the line out if it says nothing or N/A>

**Open questions**
- <a question only a person can answer — leave the block out if there are none>
```

- **Two sentences, plain words.** Write them for someone on *this* board who will never read the spec. On a board that is not about code — a supplier review, a hiring round, an event — that means the board's own vocabulary and no engineering terms at all: no "endpoint", "PR", "module", "tier", "migration". On a code board, still say what changes for the person using the thing before you say how.
- **The acceptance criteria are copied, never rewritten.** Same wording, same order as the spec's Acceptance criteria section — the same strings you pass as `tasks`. The reviewer checks the build against the spec's list; a person must be reading that same list, not a friendlier cousin of it. If a criterion reads badly in the overview, it reads badly in the spec — fix it there, then copy it. If there are many, list them all; never trim, merge, or summarise them.
- **Not included** restates the spec's Out of scope in a line or two. **Open questions** are the spec's own open questions (see the template's Open questions section) — things waiting on a person, not things you could have looked up.
- **Nothing else.** No file paths, no implementation plan, no test commands, no size or labels, no restating the title.
- **Short.** Aim for under about 120 words not counting the criteria list. The server refuses an overview over 4,000 characters — and in `create_items` that refusal aborts the whole batch. A log whose criteria alone approach that is almost always an epic (see "Logs vs epics"); if it really must be filed as one log, leave the overview off rather than cut the criteria, and tell the person in one line that you did.

A log that is itself only a few lines — a reminder for a human, not a brief — needs no overview. Skip it.

An **epic** has no acceptance criteria of its own — its children carry them. Its overview is the two sentences, plus Not included and Open questions where the epic's description has them; no **Done when** list.

### Example — a code board

```
People who follow an old invite link currently see a blank error page. This makes an expired invite say so, and offer a way to ask for a new one.

**Done when**
- An expired invite token renders the "This invite has expired" page, not a 500
- The page shows the inviter's name and a "Request a new invite" button
- Requesting a new invite emails the inviter once per token per hour
- Regression test added for the expired-token path

**Not included:** Changing how long invites last; redesigning the invite email.
```

### Example — a board that is not about code

```
We need to decide which of our current suppliers to keep for next year. This collects each supplier's first-quarter delivery and quality record in one place and gets a keep, renegotiate or replace decision from the buying team.

**Done when**
- Every active supplier has a one-page scorecard covering on-time delivery, quality issues and price changes for January to March
- The buying team has met and recorded a keep, renegotiate or replace decision for each supplier
- Each supplier marked renegotiate or replace has a named owner and a date

**Not included:** Running the renegotiations themselves; finding replacement suppliers.

**Open questions**
- Should suppliers we started using in March be scored, or held over to the next review?
```

## Keeping it in step

The server stamps an overview against the `description` and `problem` it was written beside. Change either one later without sending a new overview and the log reads back `overview_stale: true`, and people see "may be out of date" next to it. So:

- **Whenever you change a log's `description` or `problem` and the log already has an overview, send the rewritten `overview` in that same call.** Derive it again from the spec as it now stands — do not patch the old one.
- **A log read back with `overview_stale: true` has an overview you cannot trust.** Work from the description. If you are already updating that log's spec, refresh the overview in the same call.
- **A log with no overview stays that way unless someone asks.** Do not add overviews to logs you happen to pass through, and never in bulk. Fleshing out a thin log is the exception: that is writing its spec, so it gets its overview like any newly filed log.

## Writing one for an existing log

When someone asks — "add an overview to LOG-NNN", "LOG-NNN needs the short version" — `pull_item` it, derive the overview from the description exactly as above, and `update_item` with **only** `overview`. Do not touch the description. If the log has no acceptance criteria to copy, it is thin: say so and offer to flesh it out first, rather than inventing criteria for the overview.
