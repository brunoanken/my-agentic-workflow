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
