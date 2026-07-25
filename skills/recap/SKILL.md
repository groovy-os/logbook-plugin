---
name: recap
description: Reports what shipped recently — logs closed in the last few days, each with its resolution and the pull requests it links to. Use for a standup, a weekly summary, a changelog draft, or when the user asks "what did we get done this week", "what shipped", "what closed recently", "recap the week", or types "/logbook:recap". Read-only; it closes nothing and files nothing.
argument-hint: [days — 1 to 7, default 7]
---

# Recapping what shipped

Answer the question the board is worst at answering by eye: *what actually landed, and what did it change?*

## The seven-day ceiling is real

`recap` accepts 1–7 days and nothing longer. Done logs are auto-archived once they are a week old, and archived logs drop out of every read — so a 30-day window would silently return only the last seven days' worth and look like a quiet month. Rather than report a number that is wrong in a way nobody can see, the window is capped.

If the user asks for longer, say plainly that the board archives after a week and offer the seven-day recap. Do not fabricate the rest from git history unless they ask for that explicitly — a recap that mixes board truth with guesses is worse than a short one.

## Steps

1. `recap(days)` — default 7. One call; it returns every closed log with `resolution`, assignees, labels and matched pull requests.
2. **Lead with the shape**: how many closed, over how many days, by whom. One line.
3. **Then the entries**, newest first. Per log: the ref and title, what the resolution says changed, and the pull request with its state. Keep each to a line or two — a recap nobody reads to the end has failed.
4. **Group when it helps.** Several logs from one label or one cluster read better together than in strict chronological order; say so when you regroup.
5. **Name what is missing.** A log closed with no resolution is a gap worth flagging — the resolution is the only durable record of what the change did. Say which ones lack it rather than quietly skipping them.

## Shape

```
7 days · 6 logs closed · kristian 3, claude 2, mira 1

LOG-118  Sessions outlive org membership
         Cascade-deletes sessions on membership removal and re-checks at
         token resolve.                          acme/app#612 · merged

LOG-124  Batch the board revalidation writes
         (no resolution recorded)                acme/app#617 · merged
```

Do not editorialise about velocity, and do not compare periods unless asked. Report what landed.
