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
