# Decisions & open questions

**Default posture: keep moving.** Hitting an ambiguity is not a reason to stop. Write it down, make
the best call you can defend, and continue the loop. Stopping is reserved for the rare case defined
under "Truly blocking" below.

Read the repo's `CLAUDE.md` for its domain posture before deciding anything — some codebases have an
explicit tiebreaker (e.g. "always prefer correctness, even if it grows scope or adds client-side
work"). Where one exists, it settles most of these calls for you.

## The doc

One running doc per body of work, in the work folder:
`<work-folder>/decisions-and-open-questions.md`.

If it doesn't exist yet, create it. Never silently skip logging because there's no doc.

**Append entries — never rewrite or delete prior ones.** Use this shape:

```markdown
### D<N> — <short title>
- **Date:** <UTC date>  **Status:** Decided autonomously | Open — non-blocking | Open — BLOCKING | Resolved <date>
- **Context:** what I was doing and what surfaced. Include `file:line` refs.
- **Why it's ambiguous:** what the source doc/code doesn't settle.
- **Options:** each with its correctness/robustness/risk trade-off — including the ones I rejected.
- **Decision & rationale:** what I did and why. (For Open items: my recommendation.)
- **Blast radius / how to reverse:** what changes if you overrule me later.
- **Follow-up:** new story, remediation, or "none".
```

Write for a reader who wasn't in the session — enough context and insight to decide without
re-deriving anything. **This doc is a deliverable, not a scratchpad.**

## Non-blocking — the default, and the overwhelming majority

If you can pick a defensible option and keep the work correct and reversible: **log it and proceed.**

This covers naming, structure, ordering, tactical design, discovered adjacent bugs, out-of-scope
improvements, and anything where a sane default exists.

- Do **not** pause for confirmation.
- Do **not** narrow the work to dodge the ambiguity — pick the most correct option, document it,
  move on.
- Additional work you discover but shouldn't absorb (belongs to another story, or *is* its own
  story) goes in the same doc with enough detail to become a story later — then keep going.

"This would be a bigger change" and "the client would need updating" are **never** reasons to stop.

**Tripwire: if you catch yourself calling one option "Recommended," that already proves it's
non-blocking.** A call with an obvious best answer isn't ambiguity worth interrupting for — log it
under this section with that recommendation applied, and move on. Don't draft a question for
something you've already resolved in your own reasoning.

**"Surface as an open question" means write it here, not interrupt.** Task-level instructions
sometimes ask you to "surface," "flag," or "call out" a tradeoff rather than decide it silently.
Read that as pointing at this doc: log the entry, pick your recommendation, keep going. It is not
an instruction to open a live question back to the user. The three-part test below is the only
thing that justifies an actual interrupt — an inline phrase asking you to "surface" something does
not relax that bar on its own.

## Truly blocking — rare

Blocking means **all three** hold:

1. You cannot produce correct work without the answer — every path risks real harm (data-integrity
   or financial-correctness bugs, destructive or irreversible changes, or shipping something
   actively worse than the status quo), **and**
2. There is no reversible default you could take now and revisit, **and**
3. It's a product or business call that isn't yours — not a technical one you're able to reason to.

Two things sit just outside this and are also worth stopping for, per the skill's autonomy section:
a conflicting in-flight PR from the same epic, and an environment hazard that would affect
**unrelated** work.

### When it is genuinely blocking

Blocking means this **one item, or this one piece of an item**, can't proceed — it does not mean
the session stops. Keep working everything else that's independent of the answer.

1. Log the entry with **Status: Open — BLOCKING** and a clear recommendation.
2. **Stop the clock on the blocked item only** — if timing is on, fill in Finish + Duration on that
   item's current row (see `TIMING.md`). Time spent waiting is not billable, but other items keep
   their own clocks running normally.
3. **Park it, don't halt.** Set the item aside and move to the next independent item or task still
   in scope — another item in the work list, another acceptance-criterion gap, anything that
   doesn't depend on the blocked answer. A single-item run with no other independent work left is
   the only case where parking has nothing left to fall through to.
4. **Batch the question.** Don't interrupt live the moment you hit it. Accumulate blocking questions
   and present them together at the next natural checkpoint (e.g. right before a PR that depends on
   one of them) or, absent one, at end of session — alongside the usual decisions/open-questions
   summary. Surface every accumulated blocking question in that one message, not one at a time as
   they occur.
5. Stop and ask live, immediately, **only** when every remaining item in scope is also blocked (or
   depends on the same answer) and there is genuinely nothing left to make progress on.

### On resume

- Add a **new row** to the timing log with a fresh start time. Don't reopen the old row.
- Update the entry to **Status: Resolved \<date\>** with the answer received.
- Continue the loop from where you left off — through to PR, or until truly blocked again.

## End of session

Close every session with a short summary of what went into this doc: decisions made autonomously
(so they can be overruled), open non-blocking questions, and follow-up work logged. Link the doc.
Do this whether or not you were ever blocked.

The intent of the loop is autonomous work with accurate time tracking. Blocking on something small
defeats it entirely.
