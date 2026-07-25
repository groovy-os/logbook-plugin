---
name: jot
description: Writes a note on the user's private notepad — the personal column on the logbook board, not a note attached to a log. Use for a thought with no home yet: a hunch, a follow-up to look at later, something noticed in passing while doing something else. Invoke when the user says "jot that down", "note that", "remind me", "put that on my notepad", "keep that somewhere", or "/logbook:jot". Do NOT use for actual work — that is a log, filed with the logbook:add skill — and not for something about a specific log, which is add_note.
argument-hint: "<the thought>"
---

# Jotting a note

The notepad is the margin of the board. It exists so a passing thought does not have to become a log before it is allowed to be written down.

## When this is the right home

| The thought | Where it goes |
|---|---|
| "Rate limiting on invites looks weak" — a real problem | a **log** (`logbook:add`) |
| "Ask Mira why the revalidation batch is 50" | **jot** |
| "This item's approach is wrong because…" | `add_note` on that item |
| "Try the token bucket idea on the next auth item" | **jot** |

The test: if someone could pick it up and build it, it is a log. If it only makes sense to you, it is a jot. Notepads are per-user and private — nobody else on the team sees them.

## Steps

1. **Write it as the user said it.** Do not expand a one-line thought into a paragraph; the value is that jotting is cheap. Keep their words.
2. `jot(body)`. Markdown is fine.
3. Confirm in one line and stop. Do not offer to turn it into a log unless they ask — the point of a jot is that it is *not* yet work.

## If it is really a log

When the thought is plainly a piece of work — it has a problem and someone could act on it — say so once, briefly, and offer `logbook:add` instead. Then do whichever they choose. Filing a real problem as a private note is how work gets lost, which is the thing this board exists to prevent.
