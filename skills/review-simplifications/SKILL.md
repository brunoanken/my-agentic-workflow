---
name: review-simplifications
description: Review changed code purely for simplification — cleaner implementations of the current approach, and structural alternatives (data structures, state handling, module boundaries, algorithms) when a real tradeoff exists. Applies unambiguous, no-tradeoff wins directly and re-verifies them against tests/lint/types; presents anything with a real tradeoff as options instead of picking one. Explicitly excludes database schema, migrations, and query design. Use when asked to simplify code, find a cleaner approach, reduce complexity, or make code more elegant/declarative/functional — outside of bug-hunting or DB review.
metadata:
  author: Bruno Zaninello
  version: "1.0.0"
---

# Review Simplifications

Find the simplest code that still does the job — first by cleaning up the current approach,
then by asking whether the current approach is even the right shape. Two outputs, kept
strictly separate: things you just fix, and things you ask about.

## Relationship to other skills

This skill has one job: simplification. It deliberately does not do the jobs of its
neighbors — invoke them instead of duplicating their ground:

- **`simplify`** (built-in) — mechanical reuse/efficiency/altitude cleanups, applied
  directly, no presentation step. This skill is for when a finding is bigger than that: a
  real tradeoff worth the user's input, or a structural alternative (different data
  structure, different state-handling approach, different module boundary) that changes
  the shape of the solution, not just its wording. If everything you find is a
  no-tradeoff mechanical win, `simplify` was the right tool — say so rather than padding
  the review to look more substantial.
- **`enhance-code`** — the broad "find everything and fix it" scanner: bugs, security,
  performance, safety, *and* over-engineering/maintainability. Don't chase correctness,
  security, or raw performance here even if you notice it — flag it in one line and point
  at `/enhance-code` or `/code-review`, then move on. Mixing categories is exactly what
  this skill exists to avoid.
- **`database-change-modeler`** / **`review-postgres-schema`** — anything about table
  design, migrations, indexes, constraints, or query shape belongs to them, not here. Do
  not review `tables.py`, migration files, or SQL/query-builder logic for simplification.
  If a change touches those files, skip them and note the exclusion; don't silently
  review them anyway because they happened to be in the diff.

## Scope

- Analyze the staged diff (`git diff --staged`; fall back to `git diff` if nothing is
  staged) plus the full files containing those changes — same anchor `enhance-code` uses,
  so the two skills compose cleanly on the same change.
- If the user instead points at a file, module, or "this whole feature," use that as the
  target and skip the diff step.
- **Excluded regardless of target**: migration files, table/schema definition files
  (e.g. `tables.py`, `models.py` ORM row definitions where the change is column/constraint
  shape rather than application logic), and raw SQL or query-builder code. If a changed
  file mixes application logic with a query, review the logic and skip the query.
- Read the full file, not just the hunk — a diff-local view misses the sibling
  implementation three functions up that already establishes the idiom (see Process).

## Process

1. **Gather the target.** Diff or explicit target per Scope above. List the files in
   scope and the files excluded by the DB rule, so the exclusion is visible rather than
   silent.
2. **Read for precedent before proposing anything.** Look at how the same package/domain
   solves adjacent problems — a sibling service, pipeline stage, or module doing something
   structurally similar. An alternative that aligns with the codebase's own established
   pattern needs less justification than one that introduces a new idiom; note which case
   you're in for each finding.
3. **Classify every candidate finding as one of two tiers** (see the rubric below) before
   writing it up. Don't let a finding's *size* decide its tier — a one-line change can
   still hide a real tradeoff (e.g. changing a lookup from a loop to a comprehension is
   tier 1; changing it from list-of-structs to a dict keyed by id, changing what's
   memoized, or moving state from instance attributes to a passed-in context is tier 2
   even if the diff is short).
4. **Apply tier 1 findings directly**, one at a time. After each one, re-run the project's
   tests, linter, and type checker on the affected files before moving to the next finding
   — a change that looks behavior-preserving on read can still break an invariant a quick
   read won't surface (object identity, exception type, ordering, mutation vs. copy). If a
   "tier 1" turns out to break a test, that is signal it was actually tier 2: revert it,
   move it to the proposals list with the failing test as the reason it's not obvious.
5. **Write up tier 2 findings as options**, not recommendations-with-an-asterisk — see
   Output Format. Do not implement any of them.
6. **Report.** Summarize what was applied (with the verification that confirms it's safe)
   separately from what's being proposed.

## The tier rubric

**Tier 1 — apply directly.** All of these must hold:
- Behavior is unchanged for every input, including edge cases already covered by existing
  tests (empty input, duplicates, error paths) — not just the happy path.
- There's exactly one reasonable way to do it; a second engineer reading the diff wouldn't
  reach for a coin flip.
- It doesn't change a public signature, a stored data shape, or anything another module
  depends on.
- Verification (tests/lint/types) after applying it stays green.

**Tier 2 — present as options.** Any of these puts it here:
- Changes the data structure or state-handling approach (e.g. list to dict, mutable
  accumulator to a pure fold, instance state to passed-in parameters, sync to
  event-driven).
- Changes an architectural boundary (where logic lives, which layer owns a decision,
  splitting/merging modules or services).
- Trades one axis for another (readability vs. performance, flexibility vs. simplicity,
  fewer lines vs. fewer concepts) where reasonable engineers could land differently.
- Requires assumptions about future requirements, scale, or usage patterns the current
  code doesn't state.

When unsure which tier, default to tier 2. A wrong "obvious" fix costs a revert and trust;
a wrong "worth asking" costs one skipped question.

## Output Format

### Applied

For each tier 1 fix: one line naming the file, what changed, and the verification that
passed (e.g. "ran `pytest tests/unit_tests/providers/`, `ruff`, `mypy` — all green").
Show the diff if it's non-trivial to picture from the description alone.

### Proposed (needs your input)

For each tier 2 finding, in order of how much it would actually simplify the code (not
file order):

1. **What's there now** — one or two sentences, with a `path:line` anchor.
2. **Why it's a real tradeoff** — the specific axis being traded, in this codebase's
   terms (not generic "simplicity vs. flexibility" — say what flexibility, needed by what,
   evidenced how).
3. **Options**, 2–3 max, each with: a short label, what it would look like concretely
   enough to implement on a "yes, do that" reply, and its cost/benefit versus the current
   code and versus the other options. Mark one as the default recommendation if you have
   one, but don't hide the others behind it.

Keep each option to the length that lets the user decide, not a full design doc — if an
option needs more than a paragraph to explain, it's probably two options collapsed into
one and should be split, or it's actually out of scope for a "simplification" review (e.g.
a rewrite that changes product behavior, which isn't this skill's call to propose).

If nothing in scope clears tier 2, say so plainly instead of manufacturing a tradeoff to
fill the section — "no structural alternatives worth the tradeoff; see Applied above" is a
complete, successful review.
