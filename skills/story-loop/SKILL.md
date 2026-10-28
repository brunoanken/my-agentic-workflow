---
name: story-loop
description: Run an autonomous implementation loop over a source of work — a user-stories doc, a PRD, a Linear issue, PR review findings, or a backend PR whose frontend impact must be derived. Resolves a run contract up front (scope, git policy, time tracking, PR self-review, autonomy), normalizes the source into an ordered work list, then drives research → implement → quality gates → verify → commit → PR → self-review-and-fix per item via sub-agents, logging decisions and open questions as it goes. Use when asked to work through stories autonomously, run the story loop, or implement a doc end to end.
metadata:
  author: Bruno Zaninello
  version: "1.5.0"
---

# Story Loop

You are a senior software engineer acting as an **orchestrator of sub-agents**. Coordinate them
sequentially — each phase depends on the previous phase's output — so your own context stays
limited to briefs and summaries rather than full implementation detail. You should rarely read
implementation files yourself; dispatch instead.

**Adapt to the repo you're in.** Read its `CLAUDE.md` / `AGENTS.md` / `README` for stack
conventions, test runners, environment management, and domain posture (e.g. "prefer correctness
over scope"). Those are authoritative and override anything generic in this skill. This skill
supplies the *process*; the repo supplies the *facts*.

## Invocation

`/story-loop <free text>` — everything after the command is parsed for the run contract. Both of
these work:

```
/story-loop docs/2026_07_25_ledger/user-stories.md — Story 3 only, no new PR, push to this branch
```

```
/story-loop
Source: results from /review-pr #485
Pick: everything blocking for this PR
Git: same branch, no new PR
Timing: off
```

```
/story-loop docs/2026_07_25_ledger/user-stories.md — everything remaining, review each PR when it's
up and fix all findings, not just blocking ones
```

Anything not settled by the text gets asked once, in Step 0. Anything still unsettled takes the
documented default.

## Inside bb

Check once, at the start: you're in bb when `BB_THREAD_ID` is set and `bb status` succeeds. If not,
ignore this section and every "inside bb" note below — the skill runs exactly as written. If so:

1. Sub-agents run as bb child threads, nested under this thread in bb's sidebar — see
   [Dispatching a sub-agent](#dispatching-a-sub-agent).
2. If `bb loop --help >/dev/null 2>&1` succeeds (the Story Loop plugin is installed), report the run
   to its board — see [Reporting to the Story Loop board](#reporting-to-the-story-loop-board).

---

## Step 0 — Resolve the run contract

Before reading the source, before `git log`, before anything else: determine these five knobs.

| Knob | Options | Default if unstated and unasked |
|---|---|---|
| **Scope** | one named item / a named subset / everything remaining in the source | ask |
| **Git policy** | `branch-pr` · `same-branch` · `stage-only` · `none` | ask |
| **Timing** | on (path to timing log) / off | ask |
| **PR self-review** | `off` · `blocking-only` · `all-findings` | ask |
| **Autonomy** | full send / check in between items | full send |

- **`branch-pr`** — one branch per item, one draft PR per item. Never fold two items into one branch.
  In a multi-item run this **stacks by default**: item N branches off item N−1's branch, and its PR
  targets that branch (`gh pr create --base <prev-branch>`). Only the first PR targets the trunk.
- **`branch-pr-from-trunk`** — one branch and PR per item, every one cut from the trunk. Use when the
  items are genuinely independent and you want them mergeable in any order.
- **`same-branch`** — commit (and push) to the current branch; update the existing PR, never open a new one.
- **`stage-only`** — `git add` the work, no commits.
- **`none`** — no git operations at all.

**PR self-review** runs `/review-pr` against the PR this loop just created for the item, then fixes
the findings **on that same branch** — no new branch, no second PR, never its own work item. The fix
diff goes back through the same quality gates as the implementation itself. `blocking-only` fixes
the Blocking group; `all-findings` fixes the worthwhile non-blocking ones too. Read the invoking
text accordingly: "review the PR too" → `blocking-only`; "review it and fix everything it finds" →
`all-findings`. It needs a git policy that produces a PR — under `stage-only` / `none` there's
nothing to review, so don't ask about it, and say once that it's off for this run. See `REVIEW.md`.

**Never gate an item on the previous item's PR being merged, reviewed, or acknowledged.** A stack of
open PRs is the expected steady state, and needing to rebase the stack later is a normal cost, not a
signal to stop. If the source document imposes its own merge gate, say so once and carry on stacking
unless the invoking text asked for the gate. The only reason to halt is a *conflicting* in-flight PR
touching the same code from outside this run.

Silent defaults, not asked: decisions doc **on** (see `DECISIONS.md`), quality gates **auto-selected**
by what the staged diff actually touches, tracker ticket **set in progress** when its item starts
(Step 3, step 1) unless the invoking text says to leave the ticket alone.

**Ask about the unresolved knobs in a single `AskUserQuestion` call.** If the invoking text already
settles them, ask nothing and go straight to Step 1. Never ask one at a time, and never ask about a
knob the text already answered.

`AskUserQuestion` takes at most four questions per call, and autonomy already has a silent default —
so the four askable knobs are scope, git policy, timing, and PR self-review. If all five were
somehow open, autonomy takes its default rather than splitting into a second call. Drop the
self-review question too when the git policy is already known to be `stage-only` / `none`.

Then **echo the resolved contract back in one short block** so it's visible and correctable. Don't
write it to disk for a single-item run — for a multi-item run it goes in `progress.md` (Step 3).

---

## Step 1 — Timing gate

If timing is **off**, skip this entirely and go to Step 2.

If timing is **on**, your next action is a **file write** — not a `date` command, not a mental note.
Append a start row to the timing log before you read the source doc, before `git log`, before
anything. Read `TIMING.md` for the format, the definition of "start", and the clock-stop rules.

**GATE:** if you are about to read the source and no start row exists on disk yet, stop and write
it first.

---

## Step 2 — Intake: produce the work list

The loop always needs the same thing: **an ordered list of items, each with acceptance criteria.**
Sometimes the source already is that list; sometimes it must be derived.

Read `INTAKE.md` and follow it. In short:

- **Enumerated source** (user-stories doc, review findings, checklist) → normalize it, check real
  repo state (`git log`, `gh pr list --state open`) rather than trusting a status column, state
  which item you're picking up and why, confirm — unless the text already named the item.
- **Raw source** (PRD, Linear issue, backend PR diff, bare idea) → derive the item list, write it to
  disk as a stories doc, and **get sign-off on the derived list before implementing.** Adopting an
  existing list is low-variance; inventing scope is not.

If the source defines its own execution contract, ordering rules, or status conventions (e.g. an
"Agent Execution Contract" section), **those override everything in this skill.**

If the picked item has open questions or undecided behavior flagged in the source, surface them and
get a decision before implementing — don't silently pick a default on something the author
explicitly left open. (This is narrower than general ambiguity; see `DECISIONS.md`.)

---

## Step 3 — The loop, per item

Every numbered step below that says "sub-agent" means: write the brief to a file in the work folder,
dispatch a sub-agent with the path, and keep only its summary. Briefs-as-files keep your context
small and make a dead session resumable. How to dispatch:
[Dispatching a sub-agent](#dispatching-a-sub-agent). Inside bb, report each step to the board as
you go — see [Reporting to the Story Loop board](#reporting-to-the-story-loop-board).

1. **Branch** — per the git policy. `branch-pr` checks out a fresh branch for this item only, cut
   from the previous item's branch (stacked); `branch-pr-from-trunk` cuts it from the trunk. Do not
   wait for anything upstream to merge first.
   - **Claim the ticket** — if intake mapped the item to a tracker issue (Linear, Jira, GitHub
     issue), move it to the tracker's in-progress state now, under every git policy. Linear:
     `save_issue` with `state: "started"`, or the "In Progress" state from `list_issue_statuses`
     if the team has several started states. Jira: the transition into the "In Progress" status
     category. A tracker with no in-progress state (a plain GitHub issue outside a Project) is
     left alone. **Only ever move it forward** — a ticket already started, in review, done, or
     canceled stays put, which covers resumed runs and items sharing one ticket. Never move it
     past in-progress; later states belong to the PR merge and the people reviewing it.
     Best-effort: if the tracker is unavailable or the call fails, say so in the end-of-session
     summary and carry on.
2. **Research** — read-only `Explore` sub-agent for anything non-trivial. Require concrete code
   excerpts with `file:line` refs so you never re-read the files yourself.
   - **UI items: accessibility is part of research, not a later gate.** If the item already has an
     Accessibility section (stories written by `write-user-stories`), the brief carries its criteria
     forward. If it doesn't (review findings, a Linear ticket, an inline list), the research sub-agent
     invokes `/review-accessibility` in requirements mode to derive them. Either way, the brief names
     the repo's existing accessible primitives (dialog, form field, announcer) with `file:line`, so
     the implementer reuses them instead of rebuilding them. Treat the derived criteria as part of
     the item's acceptance criteria from here on.
3. **Implement** — `general-purpose` sub-agent, handed the research brief directly.
4. **Stage** — `git add`.
5. **Schema gate** *(only if the diff touches migrations, schema, or queries)* — sub-agent invokes
   `/review-postgres-schema`. Apply findings. Stage.
6. **Tests** — sub-agent invokes `/test-coverage` (this covers both writing tests and auditing
   assertion quality). Stage.
7. **Verify** — sub-agent **actually runs** the project's test suite, typecheck, and lint using the
   commands from the repo's CLAUDE.md. It reports pass/fail *with output* — never "looks correct".
   On failure: fix the failures directly and re-verify. After 3 failed
   rounds, stop iterating and log a `DECISIONS.md` entry; carry on only if the failure is
   pre-existing and unrelated. Otherwise it's blocking — park this item per `DECISIONS.md`'s
   "park, don't halt" rule and move to the next independent item rather than stopping the session.
8. **Enhance** — sub-agent invokes `/enhance-code`. Apply findings. Stage.
9. **Acceptance check** — sub-agent compares the staged diff against this item's acceptance criteria
   — including its accessibility criteria for UI items — and reports what's unmet. Fix gaps. Do not
   narrow the item to make it pass.
10. **Re-verify** if steps 8–9 changed code.
11. **Update the source doc's own status convention** (status column, header, checkbox) if it has one.
12. **Commit** — one conventional commit for the item, or a small number of them. Skip if the git
    policy is `stage-only` or `none`.
13. **PR** — always invoke `/create-pr` to author the PR; never hand-roll `gh pr create` with an
    ad-hoc description. Let it pick its own workflow (Linear-backed vs personal) and follow its
    default recommendations; pass it the item's branch, base, and source-doc reference as context.
    - `branch-pr`: a **draft** PR targeting the previous item's branch with `--base` so the stack is
      explicit on GitHub. State the stack position and the base branch in the PR description.
    - `branch-pr-from-trunk`: the same, targeting the trunk.
    - `same-branch`: push; the existing PR updates itself.
    - `stage-only` / `none`: nothing.

    **Do not ask questions during this phase.** Make reasonable calls and proceed — including
    whether to post a PR comment. Blocking on PR cosmetics defeats the entire point of the loop.
14. **Self-review the PR** *(only if the contract set PR self-review on)* — sub-agent invokes
    `/review-pr` against the PR just created, writing the review to the work folder. Fix the
    Blocking group (`blocking-only`) or blocking plus worthwhile non-blocking findings
    (`all-findings`), **on the same branch — never a new branch or a second PR**. Findings you
    disagree with get a `DECISIONS.md` entry rather than a silent drop.

    **The fix diff goes through steps 5–10 again** — schema gate, `/test-coverage`, verify,
    `/enhance-code`, acceptance check, re-verify — auto-selected by what the fixes touch, at the
    same bar as the original implementation. Then commit and push, **update the PR description
    where the fixes made it stale** (especially "What shipped", which stakeholders read), and post
    a short comment on what was fixed and what wasn't. Cap: two rounds per item. Read `REVIEW.md`
    for the fix/defer rules, the round cap, the description-update rules, and the `/review-pr`
    overrides (it must not post to GitHub or ask whether to).

    Running this before the next item's branch is cut is what keeps the stack clean — item N+1
    branches off item N's already-fixed branch, so nothing needs rebasing.
15. **Close the item** — fill in Finish, Duration, and the PR link on the timing row; append any
    decisions to the decisions doc; if this is a multi-item run, update `progress.md` in the work
    folder (resolved run contract, items done, item in flight, items remaining) so a fresh session
    can resume without re-deriving anything.

Then move to the next item, or stop if scope was a single item.

---

## Dispatching a sub-agent

**Outside bb** — the `Agent` tool, with the sub-agent type the step names (`Explore`,
`general-purpose`), handed the brief's path.

**Inside bb** — a bb child thread instead, so each worker can be opened, watched and steered:

```sh
bb thread spawn --project "$BB_PROJECT_ID" --parent-self \
  --environment "$BB_ENVIRONMENT_ID" --lifecycle-owner-thread "$BB_THREAD_ID" \
  --title "S3 · Verify" --prompt-file "<work-folder>/briefs/S3-verify.md" --json   # → {id, ...}
bb thread wait <id> --timeout 9m --json   # → {matched: true} once the child is idle
bb thread output <id> --json              # → {output}: the child's final message
```

- **Title** `<item key> · <Phase>`, adding ` (r2)` for later rounds, so the sidebar reads as the
  pipeline.
- **Same checkout, one at a time.** `--environment` puts the child in this checkout, so git state
  carries between phases. Never run two children that write files at once.
- **Model.** The child inherits this thread's provider, model and permission mode. Research could
  take a cheaper `--model`; don't change it by default.
- **Every brief ends with the same instructions.** Read-only, when the step names `Explore`. Don't
  ask questions or wait for approval — make the call and note it in the summary. Run commands in
  the foreground, and don't end the turn until the work is done. Finish with a short `## Summary`:
  what was done, files touched, commands run with pass/fail, anything unresolved. That message is
  all `bb thread output` returns, and all you keep.
- **Never wait longer than 9m in one call** — the Bash tool kills a foreground command at 10
  minutes. On timeout, read `.thread.status` from `bb thread show <id> --json`: `error` →
  `bb thread retry <id>` once and wait again, then treat a second error as a failed phase under the
  step's own rules; anything else (`active`, `starting`, `pending`) → wait again.
- **Idle isn't always done.** A child also goes idle when it backgrounds a command or blocks on a
  question. If its output has no `## Summary`: a pending interaction in `bb thread interactions
  list <id> --json` is a blocking ambiguity — handle it under `DECISIONS.md`'s "park, don't halt"
  rule. Otherwise wait again; if it's still idle with no summary, `bb thread tell <id> "Finish the
  work, then reply with the ## Summary the brief asks for."` and wait.

## Reporting to the Story Loop board

Inside bb, when `bb loop` is available. The board shows a card per item with each phase's rounds
linked to the thread that did them, plus the PR, open questions and decisions — but only what you
report. `bb loop --help` and `bb loop <command> --help` are the contract. Every call finds the run
from this thread; no run id is passed around.

**Best-effort.** If a call fails, read its hint and fix it once; otherwise move on. Reporting never
stops or slows the loop, and never replaces `progress.md` or the decisions doc.

- **Start** — once, right after the contract is echoed and the work list is final:
  `bb loop start --title … --source … --work-folder … --json`, with one `--contract key=value` per
  knob (scope, git, timing, review, autonomy) and one `--item "KEY=Title"` per item, in order. Keys
  are 1–24 of `[A-Za-z0-9._-]`, e.g. `S3`. Record the run id in `progress.md`. Items added or
  retitled later: `bb loop items add "KEY=Title"`.
- **Phases** — `bb loop phase <key> <phase> running --thread <child id>` right after spawning, then
  `done` or `failed` with a one-line `--note` when it returns. Steps map to phases: 2 `research`,
  3 `implement`, 5 `schema`, 6 `tests`, 7 `verify` (a failed round is `failed`, then `running`
  again), 8 `enhance`, 9 `acceptance`, 10 the next `verify` round, 12 `commit` and 13 `pr` (yours —
  no `--thread`), 14 `review` for `/review-pr` and `fix` for applying findings; the gates it
  re-runs are later rounds of their own phases. Steps that don't run aren't reported.
- **Item** — step 1: `bb loop item <key> --branch <b>`. Step 15: `bb loop item <key> done --pr <url>`.
- **Decisions** — mirror the decisions doc, one line per entry: `bb loop note "<text>" --item <key>`
  for a decision, `--kind question` for an open non-blocking question. A parked item gets
  `bb loop item <key> parked --note "<why>"` plus a `--kind blocker` note. An answered question:
  `bb loop resolve <note-id>`.
- **Resume** — a fresh session resuming the run calls `bb loop adopt <run-id>` (from `progress.md`)
  before anything else.
- **Finish** — the end-of-session summary calls `bb loop finish` when the scope is worked through
  (parked items included), or `bb loop finish --status stopped` when the session ends early.

---

## Autonomy

Once the contract is resolved and the item confirmed, **run the entire loop without further
check-ins** — including PR creation. Hitting an ambiguity is **not** a reason to stop. Write it
down, make the best call you can defend, and keep moving. If you find yourself calling one option
"Recommended," that's proof it's non-blocking — log it and proceed, don't ask. Read `DECISIONS.md`
for the logging format and the — rare — blocking criteria.

**Even a genuinely blocking ambiguity does not halt the session.** Park the affected item (or the
one piece of it that's stuck), keep working everything else still in scope, and batch the question
for the next natural checkpoint or end of session — see `DECISIONS.md`'s "park, don't halt" rule.
The only things that warrant stopping live, mid-session, are: every remaining item also being
blocked with nothing left to do, a conflicting in-flight PR from the same epic, or an environment
hazard affecting unrelated work.

**Environment:** use whatever the repo's CLAUDE.md says manages its dev dependencies. If you hit a
conflict that would require stopping or reconfiguring something **unrelated to this task**, stop and
say so rather than guessing. Once cleared to restart the project's own services, that covers the
rest of the loop without asking again.

## End of session

Always close with a short summary: decisions made autonomously (so they can be overruled), open
non-blocking questions, **any batched blocking questions from parked items**, follow-up work
logged, and a link to the decisions doc. Do this whether or not you were ever blocked, and whether
or not the run completed. Don't ask permission to post updates or comments — just do it. Inside bb,
close the board's run as part of it (`bb loop finish`, see
[Reporting to the Story Loop board](#reporting-to-the-story-loop-board)).

## References

- `INTAKE.md` — turning each source type into an ordered work list
- `DECISIONS.md` — the decisions & open questions doc, and what "blocking" actually means
- `TIMING.md` — timing log format, the gate, clock-stop and resume rules
- `REVIEW.md` — PR self-review: what to fix, what to log, the round cap, the `/review-pr` overrides
