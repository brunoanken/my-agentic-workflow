# Changelog

What changed in this repo and why. Newest first.

This tracks *skills and repo structure*, not code releases — there is no version number.
Entries are grouped by date, and each line names the skill it touches.

**Conventions**

- `Added` — a new skill, reference file, or section of the setup.
- `Changed` — behaviour of an existing skill changed in a way I'd want to remember.
- `Removed` — a skill dropped from the repo. Always say where it went or why it's gone;
  this is the record that keeps me from re-adding it six months later.
- Skip pure typo fixes and wording polish. If the entry doesn't change how a skill behaves
  or what's installed, it doesn't need a line.
- New dependency? It goes in the README's Dependencies table *and* gets a line here.

---

## 2026-10-27 — Comment rules for every session, and tests that don't test libraries

Agents over-comment, explaining what the code does instead of why. That's a rule for every session,
not just when a skill loads, so it lives in a new `global/` directory, imported from
`~/.claude/CLAUDE.md` rather than symlinked.

Separately, a PR review caught a test that only exercised a UI library's keyboard navigation
through our wrapper. `test-coverage` wrote it, so the fix went there, with `enhance-code` as the
backstop.

### Added

- `global/COMMENTS.md` — comment the why, never the what. Prefer small end-of-line or line-above
  comments next to the code over big header blocks. Imported from `~/.claude/CLAUDE.md`; see
  README Install.

### Changed

- `test-coverage` (2.3.0) — The Bar gains an ownership test: *if our code changed and the library
  didn't, could this assertion fail?* If not, don't write the test. A wrapper's promises are
  tested once, in the wrapper's test file, and when ownership is unclear the test isn't written.
  Step 3 flags failing tests as removal candidates and Step 6 applies the same check.
- `enhance-code` — "Redundant tests" uses the same ownership test and removes unclear cases
  rather than raising them as questions. "Noise" now flags what-not-why comments and large
  comment blocks.

## 2026-10-07 — `story-loop` runs its sub-agents as bb threads and reports to a board

I run `story-loop` inside bb. Its sub-agents were `Agent` tool calls inside the orchestrator's own
session, so bb never saw them, and I couldn't open, watch or steer a worker. Inside bb they're now
child threads nested under the orchestrator, and the run's progress goes to my Story Loop bb
plugin's board. Outside bb, nothing changes.

Smoke-testing the dispatch showed that a child going idle doesn't mean it's finished. A child that
backgrounds a command ends its turn with an interim message, then wakes up when the command
finishes. So the brief tells the child to stay in the foreground and end with a `## Summary`
heading, and the orchestrator treats an idle child without that heading as unfinished.

### Changed

- `story-loop` (1.4.0) — new **Dispatching a sub-agent** section with two paths: the `Agent` tool
  outside bb, `bb thread spawn` / `wait` / `output` inside it. Waits are capped at 9 minutes because
  the Bash tool kills foreground commands at 10. An errored child gets one retry. A child that goes
  idle before its summary is either blocked on a question (park it per `DECISIONS.md`) or nudged
  with `bb thread tell`. New **Reporting to the Story Loop board** section maps the loop steps to
  `bb loop` calls. Reporting is best-effort and never stops the loop. The board's run id goes in
  `progress.md` so a resumed session can `adopt` it. `INTAKE.md`, `DECISIONS.md` and `REVIEW.md`
  each gained a short pointer.

### Added

- Dependency (optional): the `bb` CLI and the Story Loop bb plugin, both for `story-loop`.

## 2026-10-06 — `review-accessibility` checks component aria props

Came out of a PR review nit: custom components took `ariaLabel` instead of `aria-label`. React DOM
and component libraries use the native names, and typing them from React's own types lets any aria
attribute through, not just the one someone anticipated. The rule went into `review-accessibility`
rather than `enhance-code`: `enhance-code`'s React guidance is performance-focused, and it already
calls `review-accessibility` for every UI diff, as does `review-pr`.

The rule distinguishes two cases. A prop passed straight through is an aria attribute and should
use the native name. A prop that also builds visible text or other names is a label and should be
called `label`. A blanket "use hyphens" rule gets the second case wrong.

### Changed

- `review-accessibility` — new **Component APIs** check: new or changed components that forward
  accessibility attributes accept them as `aria-*` props. Low impact, only for props the diff adds
  or changes. A repo's own consistent convention wins. Adds the `[A11y: Component API]` tag.

## 2026-10-06 — README install fixes

### Changed

- Third-party install commands now pass `-g -a claude-code`. Without `-g` the `skills` CLI installs
  into the current project, so running the README's commands from this repo's checkout vendored
  third-party skills into it.

### Added

- Dependency: `vercel-react-best-practices`, required by `enhance-code` for any JSX/TSX diff. It was
  already required but listed under "no skill here needs them" — an unrecorded dependency.

## 2026-10-06 — Accessibility built in from the start, via `review-accessibility`

Accessibility had no real place in the flow. The UI story template had an *optional*
"Accessibility Notes" section that got skipped, `enhance-code` had no a11y category, and
`test-coverage` defaulted to `data-testid` locators that pass even when a button has no name. So
a11y only came up if a reviewer happened to notice — after the code was written, when a missing
focus-return means rework.

The fix runs at both ends of every UI story. Up front, the story gets testable a11y acceptance
criteria and names the repo's existing accessible primitives to reuse. After implementation, the
diff gets reviewed against those criteria. Both modes live in one hand-written skill. WCAG guidance
comes from a third-party skill rather than being rewritten here.

Choosing the upstream: `addyosmani/web-quality-skills`'s `accessibility` — WCAG 2.2 AA, cites
success criteria, has an evidence-led Lighthouse/axe workflow, MIT, widely installed. Considered
and passed on:
- `jakubkrehel/better-accessibility` — strong review rubric, but hands contrast and typography to
  two more sibling skills.
- `ibelick/fixing-accessibility` — no WCAG references.
- `anthropics/knowledge-work-plugins/accessibility-review` — WCAG 2.1, and lists 2.5.5 (AAA) as AA.
- `web-design-guidelines` (already installed) — overlaps, but has no WCAG references, no license,
  and fetches its rules at runtime.

React Native is deliberately out of scope. No dependable RN a11y skill exists, and the ones that do
have API errors.

### Added

- `review-accessibility` — web a11y in two modes. **Requirements** turns a UI story into testable
  criteria (role + accessible name + WCAG SC) scaled to what the story actually builds. **Review**
  checks the diff, grouped by impact, and fixes it: native elements over ARIA, the repo's
  primitives over hand-rolled ones, no new dependencies. When `enhance-code` or `review-pr` calls
  it, it returns findings and lets the caller own the report.
- Dependency: `accessibility` from addyosmani/web-quality-skills, required by `review-accessibility`.
- Dependency (optional): Chrome DevTools MCP, for a Lighthouse audit and accessibility-tree
  snapshot when a dev server is already running.

### Changed

- `write-user-stories` — **Accessibility** is now a mandatory UI story section (it was optional
  "Accessibility Notes"), filled by `review-accessibility` in requirements mode. Each criterion gets
  a test scenario. A story with no new surface says so in one line.
- `story-loop` — for UI items, research carries the story's a11y criteria forward, or derives them
  when the source has no stories. The research brief names the primitives to reuse. The acceptance
  check now covers a11y criteria. No new gate: the review side runs inside `enhance-code`.
- `enhance-code` — invokes `review-accessibility` when the diff touches web UI and folds its
  findings into the report under a new Accessibility category.
- `review-pr` — the UX lens now includes accessibility. A new a11y defect that blocks a task for
  keyboard or screen-reader users counts as a new bug, so it's blocking. The code-quality lens skips
  `enhance-code`'s a11y pass so it isn't reviewed twice.
- `test-coverage` — role and label locators (`getByRole`, `getByLabel`) are the default for
  anything interactive, with `data-testid` kept for content that has no role or name. The Bar gains
  a row for it. `UI_ASSERTIONS.md` gains an Accessibility Assertions section (focus return,
  keyboard-only flows, live-region announcements, a scoped axe scan if the project already has
  axe), and validation errors are asserted via `toHaveAccessibleDescription`.

## 2026-08-25 — `review-simplifications` added

Wanted a skill that specifically hunts for simplification opportunities — cleaner code for
the current approach, but also structural alternatives (different data structure, different
state-handling strategy, different module boundary) — and presents them rather than quietly
picking one. Neither existing skill fit: `simplify` (built-in) auto-applies mechanical
cleanups with no presentation step, and `enhance-code` bundles over-engineering findings
alongside bugs/security/performance, which is the opposite of a narrowly-scoped pass. Kept
it as its own skill rather than folding into either — mixing "propose options and wait" into
`enhance-code`'s "scan and fix everything" model would have changed its interaction contract
for every other category it covers.

The two-tier split (apply directly vs. present as options) came out of a live example: a
reorder/dedup loop was rewritten to two list comprehensions for readability, then verified
against the existing test suite before calling it done. That verification step is now a hard
requirement in the skill, not a courtesy — a change that reads as behavior-preserving can
still break something a quick read won't surface (object identity, exception type, ordering),
which is exactly what happened on the first pass of that same example.

### Added

- `review-simplifications` — simplification-only review of changed code. Applies
  no-tradeoff wins directly (re-verified against tests/lint/types after each one); writes up
  anything with a real tradeoff — architecture, data structure, state handling — as 2-3
  labeled options instead of choosing. Explicitly excludes database schema, migrations, and
  query design, deferring those to `database-change-modeler` / `review-postgres-schema`.

## 2026-08-19 — Postgres guidance comes from skills; Tiger MCP and TimescaleDB dropped

`database-change-modeler` and `review-postgres-schema` read their schema-design guidance through
Tiger MCP's `view_skill` tool, which meant standing up an MCP server and a Tiger Cloud account just
to read reference markdown. That content is public, Apache-2.0, and installable on its own from
[timescale/pg-aiguide](https://github.com/timescale/pg-aiguide) — so both skills now invoke it by
name via the Skill tool instead. Same upstream, one less moving part.

TimescaleDB went out with it. Hypertables are a Tiger product feature rather than stock Postgres,
and nothing in this setup runs them.

### Added

- Dependency: the pg-aiguide Postgres reference skills, required by `database-change-modeler` and
  `review-postgres-schema` — `npx skills add timescale/pg-aiguide`. Five of them only:
  `design-postgres-tables`, `postgres-database-migration`, `design-postgis-tables`,
  `pgvector-semantic-search`, `postgres-hybrid-text-search`. All five are generic Postgres or
  open-source-extension knowledge.
- `database-change-modeler` now consults `postgres-database-migration` for DDL lock levels, timeout
  and retry patterns, batched backfills, and rollback planning, feeding its `Data Migration /
  Backfill` and `Recommended Migration Order` sections. The skill was always producing a migration
  order without a migration-specific source behind it. Its Fork-Based Migration Testing section
  names two providers as the only ones supporting fast forking; the modeler is told to disregard
  that list and recommend whatever the project actually has.

### Changed

- `review-postgres-schema` — invokes `design-postgres-tables` by name instead of calling
  `mcp__tiger__view_skill`.
- `database-change-modeler` — same swap for every design skill it references. Its Modeling Rules
  line on time-series and append-heavy tables now points at native declarative partitioning, BRIN
  indexes on the time column, and a retention strategy. The design concern was worth keeping; the
  hypertable answer to it was not.

### Removed

- **Tiger MCP**, as a dependency of anything here — not required, not optional. The pg-aiguide
  skills replaced `view_skill`, and `database-change-modeler` no longer calls `search_docs`: live
  doc search is the one thing that genuinely needs the server, and it doesn't justify an MCP server
  plus a cloud account on its own. Reach for `WebFetch` against the Postgres docs instead. Note the
  README row had pointed at `timescale/tiger-mcp`, which 404s — the server actually lives at
  [timescale/tiger-cli](https://github.com/timescale/tiger-cli).
- Hypertable guidance from `database-change-modeler` — the `setup-timescaledb-hypertables` and
  `find-hypertable-candidates` rows. TimescaleDB-specific.
- The `postgres` index skill reference from `database-change-modeler`. It's a router, and its
  "Database Management" section sends the reader to a third-party managed-database product;
  `design-postgres-tables` already covers what the row was there for.

## 2026-08-14 — Initial import

`skills/` becomes the source of truth. Previously these lived unversioned in
`~/.agents/skills/` and were symlinked into `~/.claude/skills/`; that directory is now
replaced by this repo, with `install.sh` recreating the same symlinks.

### Added

- `write-prd` — feature idea → PRD.
- `write-user-stories` — PRD → sequenced stories, with UI and backend templates.
- `database-change-modeler` — Postgres schema changes as a design exercise.
- `story-loop` — the autonomous implementation loop, plus its `INTAKE`, `DECISIONS`,
  `REVIEW`, and `TIMING` reference files.
- `test-coverage` — writes tests for staged changes and audits assertion quality;
  carries `API_ASSERTIONS` and `UI_ASSERTIONS`.
- `enhance-code` — staged-diff review across correctness, safety, security, performance,
  and over-engineering.
- `review-postgres-schema` — staged migration and query review.
- `review-pr` — multi-lens PR review against the linked ticket.
- `create-pr` — draft PRs, Linear-backed or personal.
- `record-demo-video` — stakeholder video walkthrough of a branch.

### Removed

Deliberately left out of the repo:

- `audit-tests`, `write-tests` — merged and deduped into `test-coverage`. The two
  overlapped heavily and drifted apart; one skill that writes *and* audits is the version
  I actually use.
- `commit-session`, `fix-types-and-lint`, `frontend-summary` — no longer part of my flow.
