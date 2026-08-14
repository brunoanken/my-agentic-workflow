# Intake — producing the work list

The loop consumes exactly one thing: **an ordered list of items, each with acceptance criteria.**
Intake's only job is to produce that list from whatever was handed to you, and to establish the
**work folder** everything else hangs off.

## The work folder

One folder per body of work. Everything lives there: the source doc (or derived stories), the
decisions doc, the timing log, `progress.md`.

- Source doc exists on disk → its folder. `docs/2026_07_25_ledger/user-stories.md` →
  `docs/2026_07_25_ledger/`.
- No source doc on disk (Linear issue, PR diff, bare idea) → create
  `docs/<YYYY_MM_DD>_<short-slug>/`.

Never skip creating it. "There's no doc" is not a reason to lose the decision trail.

## Chase the provenance first — but only upward

A source rarely stands alone. Before deriving or picking anything, walk the chain **up** as far as
it goes and read each level:

```
story  →  PRD  →  Linear issue  →  initiative / project
PR     →  linked Linear issue    →  PRD
```

Each level up tells you *why*, which is what makes autonomous judgment calls defensible. Stop
climbing when a level adds no constraints you didn't already have. Don't walk *down* into sibling
stories you weren't asked to do.

Use the Linear MCP tools when a ticket is referenced (`mcp__linear-server__get_issue`,
`get_document`, `list_comments`). Comments often carry decisions the description never got updated
with — read them.

## Branch A — the source already enumerates items

User-stories docs, `/review-pr` findings, checklists, a Linear issue with a task list, an epic with
sub-issues.

1. **Normalize** into an ordered list: id, title, acceptance criteria, dependencies. Preserve the
   source's own ordering and ids — the loop writes status back into the source doc later, so the ids
   must match.
2. **Check real repo state**, not the status column: `git log`, `gh pr list --state open`,
   `git branch -a`. A doc that says "Story 3: Not started" while an open PR implements it is a
   conflict, not a starting point. Status columns go stale; the repo does not.
3. **Confirm the pick** — state which item you intend to take and why, and wait. Skip this only if
   the invoking text already named the item or said "everything remaining".
4. If the source defines its own **execution contract** (ordering rules, status conventions, an
   "Agent Execution Contract" section) — follow it exactly. It overrides this skill.

No sign-off is needed on the list itself in this branch. The list was already someone's decision.

## Branch B — the source must be derived

PRD, Linear issue, backend PR diff, or a bare idea. You are inventing scope, which is
high-variance — but the skill's whole point is autonomous execution, so derivation is **not** a
checkpoint. Treat the item list itself as exactly the kind of call `DECISIONS.md` describes:
non-blocking by default, log it, keep moving.

1. **Read the source completely**, plus everything up its provenance chain.
2. **Derive the items.** For anything beyond a couple of obvious units, invoke the
   `write-user-stories` skill — it produces `docs/YYYY_MM_DD_feature_name/user-stories.md` with
   acceptance criteria and test specs already in the right shape, sequenced so dependencies flow
   forward. For a genuinely small source (one Linear ticket, one bug), a short inline list in the
   work folder is enough; don't ceremony it up.
3. **Write the list to disk** in the work folder. The derived doc *becomes* the source doc: the loop
   updates its status column, and the decisions doc and timing log sit beside it.
4. **Post the list, then go — do not wait for sign-off.** State it compactly in one message (item
   titles, one line each, plus anything you deliberately excluded and why) so it's visible and
   correctable, and immediately continue into implementation in the same turn. This is a decision
   you are equipped to make (you just read the source and its provenance chain) — the fact that it
   is high-variance is precisely why `DECISIONS.md`'s tripwire applies: if you can defend a pick,
   posting it and moving is correct, not interrupting. The user can redirect at any point after —
   that costs less than a blocked turn costs on every run that didn't need redirecting.
   Only stop for real sign-off first when scope is genuinely unrecoverable if wrong — e.g. the
   derived work would touch billing/money-movement code paths with no reversible undo, or the
   source itself is ambiguous about *whether work should happen at all* (not just *what shape it
   takes*). That is rare; default to posting and going.
5. Then proceed as Branch A from step 3 — including that branch's own instruction not to pause for
   confirmation on the pick.

### Deriving from a backend PR (the frontend-adaptation case)

The ask is usually "here's a backend change, work out what the frontend must become." Read the diff
for the contract change, not the implementation:

- **Contract surface** — endpoints added/removed/renamed, request and response shapes, new required
  fields, changed nullability, new enum values, changed status codes, changed pagination.
- **Semantic changes that don't show in types** — a field that now means something different, an
  operation that became async, an error that's now recoverable.
- **Removals** — the most dangerous class. Anything the frontend currently calls that no longer
  exists, or no longer returns what it did.

Then find every frontend site that touches the changed surface, and split the work into:

- **Adaptation** — the frontend must change or it breaks. Non-negotiable, sequence first.
- **New capability** — the backend now enables something the UI doesn't expose yet. This needs
  actual product and UX thinking, not a mechanical binding: what's the flow, where does it live in
  the existing IA, what are the empty/loading/error states, what does it displace. Say so in the
  story rather than leaving it implied.
- **Cleanup** — dead frontend code for removed backend surface.

Flag anything that is a **backend gap** rather than frontend work (the frontend can't build X
because the endpoint doesn't return Y) — that goes to the decisions doc as follow-up, not into a
story you'll fail to complete.

**Optional fan-out:** if the impact surface is large — a big PR against a large frontend, where
missing a call site means shipping a break — dispatch parallel read-only `Explore` sub-agents, one
per angle (by API client module, by route/page, by shared type, by test file). One sequential
explorer is cheaper but has a real miss rate here, and this is the one phase in the whole skill
where breadth beats depth. Use judgment; default to a single explorer for a small diff.

## What intake does not do

- It doesn't implement anything.
- It doesn't expand the list beyond the source's intent. Adjacent work you discover goes in the
  decisions doc as a candidate follow-up story — see `DECISIONS.md`.
- It doesn't narrow the list to dodge hard items.
