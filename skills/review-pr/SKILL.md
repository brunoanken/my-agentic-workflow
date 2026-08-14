---
name: review-pr
description: Multi-lens PR review (DB/perf, code quality, completeness vs linked issue, test quality, UX) that verifies the PR against its linked ticket by default and separates blocking issues (unjustified scope gaps, new bugs/edge cases, over-engineering) from non-blocking findings, with shared context fetched once and lenses gated by what actually changed. Use when asked to review a GitHub pull request in depth.
metadata:
  author: brunozaninello
  version: "2.1.0"
---

# Review PR

You are a senior software engineer reviewing a pull request. Adapt your technical lens to the repo you're in (read its CLAUDE.md / README for stack conventions if present). Your default job is to verify the PR actually achieves the outcome it was asked to achieve — whatever's in its linked ticket, its description, or discussion comments — and to separate **blocking** issues from everything else. You are not trying to make the code perfect; you're trying to make sure it's correct, complete, and fits this codebase's design, architecture, and scalability needs. Non-blocking findings are still worth surfacing, but they must be clearly differentiated from blocking ones, never mixed in with them.

**A finding is blocking only if it is one of:**
1. **Incomplete coverage** — the diff doesn't do everything the ticket/description asked, and there's no disclosed, sound reasoning for the gap. If the PR description or comments disclose a narrowing/deferral, judge whether the stated reasoning actually holds up (given the ticket's intent and the codebase's constraints) — a disclosed-but-weak justification is still a gap. If it's genuinely ambiguous whether the reasoning holds, treat it as blocking rather than giving the benefit of the doubt.
2. **New bugs or unhandled edge cases** introduced by this diff (not pre-existing issues the diff didn't touch).
3. **Over-engineering or overcomplication** — added complexity, abstraction, or indirection that isn't warranted by what this codebase's architecture and scalability actually need right now. This is nuanced and cuts both ways: don't flag necessary complexity as over-engineering, and don't wave off genuine premature abstraction because it "might be useful later." Weigh it against this repo's own conventions and stated architecture principles, not a generic ideal.

Everything else — style, idiom mismatches, minor perf notes, cosmetic issues, nice-to-haves — is non-blocking. Database and general performance pitfalls are always worth reporting even when non-blocking, since they're expensive to catch later — but still label them non-blocking unless they meet one of the three criteria above.

## Usage

`/review-pr <pr-number-or-url>` — reviews that PR. If no argument is given, run `gh pr view --json number,url` against the current branch and confirm that's the intended PR before proceeding.

## Why this skill is structured this way

Five specialized lenses could look at the same PR, but not every lens is relevant to every PR, and spawning a subagent only to have it reply "not applicable" still costs a full copy of the shared context. So: fetch shared context exactly once, decide from the changed-file list which lenses actually apply, and only fan out to subagents when the PR is big enough that parallelism and independent judgment are worth the context-duplication cost. For small PRs, do the whole review yourself in one pass — no spawns, nothing to dedup. For larger PRs, spawn only the applicable lenses, and do the merge/scoring pass yourself using the context you already have rather than paying for a fresh agent to re-ingest it.

## Steps

### 1. Gather shared context once (do this yourself, do not delegate it)

Run in parallel:
- `gh pr view <n> --json title,body,url,headRefName,baseRefName,files,additions,deletions`
- `gh pr diff <n>`
- List changed file paths, then find the root `CLAUDE.md`/`AGENTS.md`/`README.md` plus any such file in directories the PR touches (paths only — subagents will read the ones they need).

Resolve the linked issue:
- Look for an issue key in the PR title, PR body, or `headRefName` (e.g. a Linear key like `LOOP-123`, or a Jira/GitHub issue reference). Branch naming conventions vary by repo — check recent branch names if unsure.
- If found, fetch it once via whatever issue tracker MCP/CLI is available (e.g. Linear MCP `get_issue`, `list_comments` if there's discussion worth reading).
- If no issue key is found, note that explicitly — don't guess, and tell the completeness lens (3b·c) to flag "no linked issue found" rather than silently skipping the completeness check.

Assemble this into one **shared context block** (PR diff, PR description, linked issue title/description/comments, list of relevant convention-doc paths). This is what gets used everywhere below — pasted verbatim into subagent prompts if you fan out, or just kept in your own context if you review it yourself.

### 2. Decide which lenses apply, and whether to fan out at all

From the changed-file list (already in hand from step 1, no extra calls needed), determine which lenses are in scope:

- **Completeness & correctness** and **code quality** — always in scope.
- **Database & performance** — in scope if the diff touches migration files (`migrations/`), a DataStore/SQL implementation file, or any file whose hunks contain SQL or query-builder calls (`SELECT`/`INSERT`/`UPDATE`/`DELETE`/`JOIN`, ORM query methods, etc.). This is deliberately broader than "schema changes" — a modified or new query on an existing table still needs this lens.
- **Test quality** — in scope if any changed file matches a test naming convention (`*.test.ts`, `*.spec.ts`, `__tests__/`, etc.). If in scope, this lens always gets the *full* diff, never trimmed — judging untested branches/edge cases requires seeing the non-test source changes too, not just the test file.
- **UX** — in scope if the diff touches user-facing files (components, routes/pages, client-visible UI). Out of scope for pure backend/infra PRs.

Count the in-scope lenses and estimate size (total additions + deletions, file count). **Go direct (skip fan-out entirely) if either holds:**
- 2 or fewer lenses are in scope, or
- the diff is small (roughly under ~150 changed lines across under ~5 files).

Otherwise, fan out (step 3b).

### 3a. Direct review (small PRs / few applicable lenses)

Work through each in-scope lens's checklist yourself (definitions in step 3b below apply equally here — just run them as your own analysis instead of a subagent prompt), invoking the relevant sub-skill inline where it applies (`/review-postgres-schema`, `/enhance-code`, `/test-coverage` in audit-only mode) only for lenses actually in scope. Produce the final markdown directly, grouped Blocking/Non-Blocking per step 3c's format — there's nothing to dedup since it's a single pass. Skip to step 4.

### 3b. Fan out (single message, one Agent tool call per in-scope lens)

Each prompt = shared context block + the lens-specific instructions below + this shared output format:

```
Return findings as a flat list, one per line:
BLOCKING|NON-BLOCKING | SEVERITY | one-line summary | file:line | 1-3 sentence rationale (why this severity, and why blocking or not)
SEVERITY ∈ {critical, high, medium, low}. If you found nothing, say "No findings."
A finding is BLOCKING only if it is: incomplete coverage of the ask without a disclosed and sound
justification, a new bug/unhandled edge case introduced by this diff, or unwarranted
over-engineering/overcomplication relative to this codebase's actual architecture and scale needs.
Everything else is NON-BLOCKING, even if worth reporting (e.g. DB/perf notes, style, minor gaps).
When in doubt about whether disclosed scope-narrowing reasoning is sound, call it BLOCKING.
Do not re-fetch the PR diff, PR description, or linked issue — they're already in your prompt above.
Do not run the test suite, linter, typechecker, or build — assume CI catches those; only read code.
```

**a. Database modeling & performance** (only if in scope per step 2). Review query patterns, indexes, table/schema changes, N+1s, missing indexes on new foreign keys or filter columns, migration safety (locking, backfills on large tables), and any new/modified query on existing tables. Use the `/review-postgres-schema` skill (or repo-specific DB review skill) to augment this. Performance issues here are always worth reporting, even if minor — say so explicitly and let severity reflect actual impact rather than dropping them.

**b. Code quality & conventions** (always in scope). How well do the added/modified lines follow this codebase's existing patterns (see the convention-doc paths in the shared context) and idiomatic practice for its language/framework. Use the `/enhance-code` skill to aid this. Skip anything a linter/typechecker/formatter would catch. This lens is not about making the code perfect — findings here are non-blocking by nature (idiom/style mismatches, not the three blocking criteria from the intro); report them, but they should never end up in the Blocking group.

**c. Completeness, correctness & right-sizing against the linked issue** (always in scope — most important lens, give this the most care; also carries all three blocking criteria, so judge it thoroughly). Read what the issue actually asks for, what the PR description claims, and what the diff actually does. You also get the convention-doc paths from the shared context — use them for the right-sizing judgment below.

- **Coverage**: Does the diff solve the issue? If it intentionally narrows scope or defers part of the ask, check whether the PR description or comments disclose it *and* whether the stated reasoning is actually sound given the ticket's intent and this codebase's constraints — a disclosed-but-weak justification is still a gap. If it's ambiguous whether the reasoning holds up, treat it as a gap rather than giving benefit of the doubt. Undisclosed narrowing is always a gap. Flag genuine gaps as blocking.
- **Correctness**: does the diff introduce new bugs or leave new edge cases unhandled? Pre-existing issues the diff didn't touch are out of scope. If verifying correctness requires reading code the PR didn't touch (e.g. a caller of a changed function, an existing type it must satisfy), read it — don't limit yourself to the diff. Flag genuine new bugs/edge cases as blocking.
- **Right-sizing**: does the diff introduce complexity, abstraction, or indirection beyond what this codebase's architecture and scalability needs actually call for right now? This is nuanced — weigh it against this repo's own conventions and stated principles (from the convention docs), not a generic "simpler is always better" instinct. Only flag as blocking when the added complexity is genuinely unwarranted, not merely more elaborate than you'd have written it.

**d. Test quality** (only if in scope per step 2). Give this lens the full shared diff, unfiltered. Are the added/modified tests' assertions strong (exact values, not vague truthy checks) and do they cover the critical/crucial branches of the new *and existing* code touched by this diff, not just the happy path — this requires seeing the source changes, not just the test file. Use the `/test-coverage` skill to aid this, in audit-only mode ("just audit, don't write anything" — stop after its Step 3, no new tests). Read-only — do not execute the suite.

**e. UX issues** (only if in scope per step 2). Trim this lens's diff to hunks touching user-facing files (components, routes/pages, client-visible UI) — keep the full PR description and linked issue as-is, just filter the diff itself. Look for inconsistencies, unhandled edge cases, or rough interaction flows introduced by this change.

### 3c. Dedup and score

Do this yourself, in-thread, using the shared context block you already have (from step 1) plus the raw finding lists returned by whichever lenses you spawned — don't spawn a fresh agent just to re-ingest context you're already holding. Only spawn a dedicated dedup agent when 4+ lenses were fanned out or the combined raw finding count is large (rough guide: 12+) — at that scale independent cross-checking earns back its cost.

Either way, apply the same process:
- Merge findings from different lenses that point at the same root cause into a single entry (keep the strongest rationale, note which lenses independently flagged it — corroboration from multiple lenses is a signal of real severity, not a reason to list it multiple times).
- Drop: pre-existing issues (not introduced by this diff), issues a linter/typechecker/CI would already catch, pedantic nitpicks a senior engineer wouldn't raise, and anything that reads as a false positive on closer inspection of the diff.
- Never drop a database or general performance finding for being "minor" — keep it non-blocking, just size the severity honestly.
- Re-confirm the BLOCKING/NON-BLOCKING call per finding — a lens can mislabel, so re-check against the three blocking criteria from the intro (unjustified incomplete coverage, new bug/unhandled edge case, unwarranted over-engineering) rather than trusting the lens's tag at face value. When a disclosed scope-cut's soundness is genuinely ambiguous, keep it blocking.
- Re-confirm final severity per finding using this rubric (severity is orthogonal to blocking/non-blocking — a non-blocking finding can still be high-severity-in-its-own-right, e.g. a bad-but-pre-existing-pattern-adjacent perf issue):
  - **Critical** — the PR does not solve (or unsoundly narrows) what the linked issue asked for, or introduces a correctness/security/data-loss bug, or introduces genuinely harmful over-engineering. (Always blocking.)
  - **High** — a real bug or gap that will bite in normal usage, or a significant scope reduction whose justification is shaky. (Usually blocking.)
  - **Medium** — a real issue but low-frequency or limited-impact; meaningful but not urgent perf/convention problems. (Usually non-blocking.)
  - **Low** — minor, nice-to-have, cosmetic. (Always non-blocking.)
- Output final markdown in two top-level groups, **Blocking** first, then **Non-Blocking**, each internally ordered Critical→High→Medium→Low (omit empty groups/subgroups). Each item succinct but complete enough to stand alone in a PR comment: what's wrong, where (file:line), why it's (non-)blocking, why it earned that severity.

### 4. Present, then ask before posting

Show the final markdown to the user directly. **Do not post it to GitHub yourself.** Ask whether to post it as a PR comment (`gh pr comment <n> --body-file <tmpfile>`) — posting is visible to the whole team, so it needs explicit confirmation each time, not just once per session.
