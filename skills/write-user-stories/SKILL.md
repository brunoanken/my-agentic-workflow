---
name: write-user-stories
description: Transform a PRD into sequenced, implementation-ready user stories with test specifications. Use this skill after write-prd has produced a PRD and you need actionable stories before implementation.
metadata:
  author: Bruno Zaninello
  version: "1.1.0"
---

# Write User Stories

Transform a PRD (Product Requirements Document) into a sequenced set of implementation-ready user stories. Each story is a concrete unit of work with acceptance criteria and a test plan. The output is a single markdown file at `docs/YYYY_MM_DD_feature_name/user-stories.md`, alongside the source PRD.

## Philosophy

1. **Stories are work units, not documentation** - Each one has a clear deliverable and a definition of done
2. **Stories implement the PRD, nothing more** - A story adds no scope beyond the PRD. If genuinely missing work surfaces while writing stories, flag it to the user as a PRD gap — don't silently absorb it into a story
3. **Sequence matters** - Stories build on each other in implementation order. Dependencies flow forward, never backward
4. **Tests are first-class** - Every story specifies what to test and how, with specific assertions, not vague "verify it works"
5. **Templates are non-negotiable** - You MUST read the template file with the Read tool before writing each story type. No improvising sections from memory

## Workflow

### Step 1: Read the PRD

Accept a path to a PRD, or find the most recent one in `docs/`.

Parse it fully — understand:
- Goals, non-goals, and their rationale
- API surface (endpoints, schemas, error formats)
- Data model changes (tables, columns, migrations)
- Key files (existing files to modify, new files to create)
- Open questions (anything unresolved)

**Non-Goals are hard boundaries, not context.** The PRD records deliberately-descoped complexity there, with reasons. A story that implements or partially implements a non-goal is a defect. When a story's natural shape brushes against one, scope the story to stop at the boundary and note it.

Identify all distinct pieces of work that need stories.

### Step 2: Classify Each Piece of Work

Every piece of work gets classified as **UI** or **Backend**:

- **UI story** — The primary deliverable is something a user sees or interacts with: a page, form, drawer, component, modal, CLI output, or visual change
- **Backend story** — The primary deliverable is system internals: an endpoint, migration, service logic, business rules, configuration, or data processing

**Mixed work:** When a piece of work has both UI and backend deliverables, split it into two stories. The backend story comes first in sequence, and the UI story declares a dependency on it.

### Step 3: Read the Appropriate Template

**Before writing any story, you MUST use the Read tool to read the template file for that story type.** This is mandatory — do not write stories from memory.

- For UI stories: read `ui-story-template.md`
- For Backend stories: read `backend-story-template.md`

Both templates sit in this skill's own directory, alongside this file.

You only need to read each template once per session (not once per story), but you must read it before writing the first story of that type.

Follow the template's mandatory sections exactly. Include optional sections only when they apply.

### Step 4: Sequence the Stories

Order stories so each one's dependencies are fulfilled by earlier stories:

1. **Data model / migration stories first** — Schema changes that everything else depends on
2. **API / backend logic stories next** — Endpoints, services, business rules
3. **UI stories last** — They depend on backend being ready

Within each tier, order by dependency chain. Number stories sequentially: Story 1, Story 2, etc.

### Step 5: Write Test Specifications Per Story

Each story gets a **Test Plan** section containing:

- **Required setup** — What state, data, or fixtures must exist before tests run
- **Test scenarios** — Each scenario has:
  - Scenario name
  - Preconditions
  - Actions / inputs
  - Expected assertions (be specific: exact values, status codes, state changes, UI text)
- **Edge cases** — Unusual inputs, boundary conditions, error paths
- **What NOT to test** — Explicitly scope the tests to avoid redundant coverage (e.g., "Don't test auth middleware — covered by shared test suite")

**Scale the test plan to the story's risk.** Cover the acceptance criteria and realistic failure paths; don't enumerate every input permutation for low-risk work.

### Step 6: Cross-Reference & Validate

Every requirement in the source PRD must be covered by at least one story.

Add a **Traceability Matrix** at the end of the document mapping PRD requirements to stories. Flag any requirements that couldn't be mapped — these become open questions.

### Step 7: Review with User

Present the draft. Iterate on feedback until the user approves.

## Output Structure

Create the file at `docs/YYYY_MM_DD_feature_name/user-stories.md` using the same date and feature name as the source PRD.

```markdown
# User Stories: [Feature Name]

> Source: `docs/YYYY_MM_DD_feature_name/prd.md`
> Generated: YYYY-MM-DD

## Story Map

| # | Title | Type | Status | Depends On |
|---|-------|------|--------|------------|
| 1 | [title] | Backend | Not Started | — |
| 2 | [title] | Backend | Not Started | Story 1 |
| 3 | [title] | UI | Not Started | Story 2 |

## Stories

[Each story follows its type-specific template]

## Traceability Matrix

| Requirement | Covered By |
|-------------|------------|
| [requirement from PRD] | Story 1, Story 3 |
| [requirement from PRD] | Story 2 |

## Open Questions

[Any requirements that couldn't be mapped to stories, or new questions that emerged during story writing]
```

## Anti-Patterns to Avoid

- **Improvising story sections** — Read the template file. Every time
- **Writing tests as "verify it works"** — Specify exact assertions: status codes, field values, UI text, state changes
- **Backward dependencies** — Story 5 should never depend on Story 7. Re-sequence if this happens
- **Mixing UI and backend in one story** — Split them. Backend first, UI depends on it
- **Skipping the traceability matrix** — Every requirement must be accounted for
- **Pasting code blocks** — Point to files with line numbers, same as the PRD
- **Scope creep via stories** — Adding endpoints, fields, options, or abstractions the PRD didn't ask for
- **Over-splitting** — Ten two-line stories for a one-day feature; a story is a meaningful unit of work, not a task-tracker line item

## Output

The final deliverable is a single markdown file:
```
docs/YYYY_MM_DD_feature_name/user-stories.md
```

This document is the input for implementation — each story can be worked on sequentially.
