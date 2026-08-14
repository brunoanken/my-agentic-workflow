# Timing log

Only applies when the run contract set timing to **on**. If timing is off, none of this happens —
don't create the file, don't mention it.

## Location

`<work-folder>/timing-log.md`, unless the invoking text names a different path. Create it if it
doesn't exist, with this header:

```markdown
| Item | Start (UTC) | Finish (UTC) | Duration | PR |
|---|---|---|---|---|
```

## The gate

Your first action after the run contract resolves is a **file write** — not a `date` command, not a
mental note. Append the start row with Finish/Duration/PR left as `—`:

```markdown
| Story 3 — Ledger reconciliation | 2026-07-30T14:02:40Z | — | — | — |
```

Only once that row exists on disk may you do anything else — reading the source doc, `git log`,
dispatching sub-agents, any of it.

**GATE:** if you are about to read the source and no start row exists, stop and write it first.

## What "start" means

Start is **the moment you begin working** — i.e. when `/story-loop` was invoked. It is **not**
branch checkout, and it is not "after research finished."

If the run contract needed questions in Step 0, the row still records the invocation time; the few
seconds spent asking are part of the work. Write the row the instant the answers land.

If an older note in an existing `timing-log.md` defines start differently, **that note is wrong** —
correct it rather than following it.

## Closing a row

At PR time (or at the end of the item for non-PR git policies), edit **the same row** to fill in:

When the run contract turned **PR self-review** on, "PR time" means *after the review fixes are
pushed* — the review and its fixes are part of the item's work, so the row stays open through them.
Still one row: don't open a second one for the review pass.

- **Finish** — UTC timestamp
- **Duration** — human-readable, e.g. `1h 12m`
- **PR** — the PR link, or `—` if the git policy created none

One row per item per contiguous stretch of work. Never fold two items into one row.

## Blocking stops the clock

When you hit a genuinely blocking question (see `DECISIONS.md`), fill in Finish + Duration on the
current row **before** you stop and ask. Time spent waiting on an answer is not billable.

**On resume:** add a **new row** with a fresh start time. Do not reopen or extend the closed row.
Same item title is fine — two rows for one item is the correct record of what happened.

## Multi-item runs

Each item gets its own row, written when that item starts, closed when that item's PR lands. Don't
batch them at the end — a session that dies mid-run should leave an accurate partial record.
