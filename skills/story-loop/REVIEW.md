# PR self-review

Only applies when the run contract set **PR self-review** to `blocking-only` or `all-findings`. If
it's off, none of this happens.

This is the loop reviewing **its own output**: the PR it just created for the item it just finished.
It is not the same thing as consuming someone else's `/review-pr` findings as a source of work —
that's intake (`INTAKE.md`, Branch A).

## Preconditions

Needs a PR to exist, so it needs a git policy that produces one: `branch-pr`,
`branch-pr-from-trunk`, or `same-branch`. Under `stage-only` / `none` there is nothing to review —
say so once when the contract resolves and treat self-review as off for the run.

Runs **after** the item's PR exists and **before** the next item's branch is cut. Keeping that order
is what makes the "no new branch" rule free: item N+1 branches off item N's already-fixed branch, so
nothing needs rebasing.

## 1. Review — one sub-agent, one file

Dispatch a sub-agent that invokes `/review-pr <pr-number>` and writes the full review markdown to
`<work-folder>/reviews/<item-id>-pr<N>.md`. It returns only the finding list (the two grouped
blocks), not the reasoning behind them.

Two overrides on `/review-pr` that the sub-agent must be told explicitly, because they contradict
that skill's own defaults:

- **Do not post anything to GitHub, and do not ask whether to post.** `/review-pr` ends by asking;
  here it stops at the markdown. Posting, if any, is step 4 below and the orchestrator owns it.
- **Do not fix anything.** Review and fix are separate agents so the reviewer never grades work it
  just wrote.

## 2. Decide what to fix

| Contract | Fix | Log instead of fixing |
|---|---|---|
| `blocking-only` (default) | every finding in the **Blocking** group | the whole Non-Blocking group → decisions doc, one entry covering them, with the review file linked |
| `all-findings` | Blocking group, plus every Non-Blocking finding that is genuinely worth fixing | Non-Blocking findings you judge wrong, cosmetic-to-the-point-of-noise, or belonging to another item |

Three rules that hold under both settings:

- **A finding you disagree with is never silently dropped.** Log a `DECISIONS.md` entry naming the
  finding, why you think it's wrong or out of scope, and what you did instead. The reviewer being
  overruled is fine; it being overruled invisibly is not.
- **A blocking "incomplete coverage" finding gets fixed, not deferred.** Step 9 of the loop already
  asserted the item met its acceptance criteria — a coverage gap surviving that is a miss in the
  work, not a matter of taste. The only exception is a gap that genuinely belongs to a *different*
  item in the work list; say which one, in the decisions doc.
- **Findings that are real but belong to another story go to the decisions doc as follow-up**, not
  into this item. Self-review is not a licence to grow the item.

## 3. Fix, then put the fixes through the full gate set

Review fixes are code like any other code, and they're written under more time pressure and less
context than the original implementation — which is exactly when quality slips. **The fix diff goes
through the same gates as the item's own implementation, not a lighter version of them.**

1. **Fix** — sub-agent, handed the review file path and the explicit list of findings to fix.
2. **Stage** — `git add`.
3. **Run loop steps 5–10 over the fix diff**, in the same order and with the same standards:
   - **Schema gate** (`/review-postgres-schema`) — if the fix diff touches migrations, schema, or
     queries. Apply findings. Stage.
   - **Tests** (`/test-coverage`) — new behaviour or changed behaviour in the fixes needs coverage
     and assertion-quality auditing, same bar as the original. Stage.
   - **Verify** — an actual run of suite + typecheck + lint, with output. Same 3-round failure rule:
     fix and re-verify, and after 3 failed rounds log a `DECISIONS.md` entry and park rather than
     grinding.
   - **Enhance** (`/enhance-code`) — apply findings. Stage.
   - **Acceptance check** — the fix diff must not have broken or narrowed the item's acceptance
     criteria while satisfying the reviewer. Re-check against the criteria, not just against the
     findings.
   - **Re-verify** if the enhance or acceptance steps changed code.

   The gates are auto-selected by what the fix diff actually touches, exactly as in the main loop —
   a one-line null-check fix doesn't need the schema gate. Auto-selected means *judged*, not
   *skipped by default*: if the fix diff touches it, the gate runs.
4. **Confirm the fixes land** — a short sub-agent pass over the fix diff against the round-1
   findings: does each fixed finding actually resolve, and did the fix introduce anything new? This
   is deliberately cheaper than a second `/review-pr`. Escalate to a full second `/review-pr` only
   if the fixes went well beyond the files the findings named, or added substantive new logic.
5. **Commit and push to the same branch.** One conventional commit (or a small number), e.g.
   `fix(ledger): address PR review — <what>`. **Never** cut a branch, never open a second PR, never
   stack the review fixes as their own item. The existing PR updates itself.

**Cap: two rounds per item.** If a second round still returns blocking findings, fix the ones that
are clearly correct, then stop — log whatever remains as decision entries and, if any of it is
genuinely blocking, park it per `DECISIONS.md`'s park-don't-halt rule. Ping-ponging with your own
reviewer is worse than shipping a PR with a documented open finding on it.

## 4. Record it

- **PR description** — the description was written against the pre-review diff, so anything the
  fixes changed has made it stale. Update it in place (`gh pr edit <n> --body-file <tmpfile>`),
  keeping `/create-pr`'s section structure and voice rather than bolting a changelog onto the end:
  - **What shipped** — the one that matters most. It's written for non-technical stakeholders and
    gets copy-pasted into Slack, so a fix that changed user-visible behaviour, wording, or an
    outcome claim must be reflected here. If the fixes were purely internal, leave it alone —
    don't churn it just to show activity.
  - **What's included** — update where the fixes changed the technical shape of the change.
  - **Deferred** — add findings you consciously deferred to another item or follow-up.
  - **Decisions & tradeoffs** — add decisions the review forced, including reviewer findings you
    overruled and why.

  Rewrite the affected sections as if they'd always read that way. The PR description describes the
  PR's final state, not its history — the comment below is where the review narrative goes.
- **PR comment** — post one short comment on the PR: findings fixed, findings deliberately not
  fixed and why, link to the decisions doc, and a note if the description was updated. Don't paste
  the full review markdown, and don't ask first (per the skill's autonomy rule). This is the record
  the human reviewer needs to know what the self-review already caught.
- **Decisions doc** — the entries from step 2.
- **Timing** — review and its fixes belong to the item's existing row. Don't close the row at PR
  creation when self-review is on; close it once the fixes are pushed. Don't open a second row.
- **`progress.md`** (multi-item runs) — note the round count and outcome per item, e.g.
  `Story 3 — PR #512 — self-review: 2 blocking fixed, 1 non-blocking deferred (D7)`.

## Autonomy

This whole step is silent. It never asks — not about which findings to fix, not about posting the
comment, not about whether a disagreement with the reviewer is worth raising. Write it down and keep
moving.
