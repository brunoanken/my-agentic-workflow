---
name: write-prd
description: Transform a feature idea into a Product Requirements Document. Use this skill when you need to define the why, what, and scope of a feature before implementation. Produces a PRD that feeds into write-user-stories for actionable breakdown.
metadata:
  author: Bruno Zaninello
  version: "2.1.0"
---

# Write PRD

Transform a raw feature idea into a Product Requirements Document (PRD) that defines the problem, scope, and shape of a feature. The output is a markdown file at `docs/YYYY_MM_DD_feature_name/prd.md`.

## Philosophy

A PRD answers "why are we building this?" and "what does it look like when it's done?" — before anyone writes code.

1. **Problem first** — Start with the pain point or opportunity, not the solution
2. **Scope explicitly** — Goals and non-goals prevent scope creep and misaligned effort
3. **Simplest solution that achieves the outcome** — Aim for 80% of the value at 20% of the effort. The PRD's job is to reach the desired outcome, not to anticipate every future need. No speculative flexibility, no premature scale, no niceties that aren't immediately useful
4. **Describe from the user's perspective** — What the user experiences, not how the code works
5. **Technical context supports, doesn't lead** — Architecture and constraints inform decisions but don't dominate the document
6. **Omit what doesn't apply** — Empty sections are noise. If there are no breaking changes, don't include the section
7. **Point to real code, not snippets** — File paths with line numbers stay accurate; pasted code rots

## Workflow

### Step 1: Understand the Problem

Parse the user's raw input. Identify:
- The problem being solved or the opportunity being captured
- Who is affected (which users, roles, or personas)
- Why this matters now

**Ask clarifying questions before exploring.** Don't assume — confirm intent, scope boundaries, and any constraints the user has in mind.

### Step 2: Research the Codebase

Understand the landscape this feature lives in:
- Look for similar features already implemented (e.g., AR invoices when building AP bills)
- Identify files that will need modification
- Note reusable components, shared utilities, or patterns to follow
- Read the project's `CLAUDE.md` for conventions and patterns
- Identify constraints that will shape the solution

Use the Explore agent or Glob/Grep tools to search thoroughly. Record file paths for everything you reference.

### Step 3: Research External Interfaces (if applicable)

When the feature involves external interfaces, research what already exists:

- **Consuming an API:** Fetch specs, map schemas, identify new vs. modified endpoints
- **Exposing an API:** Design endpoints, define schemas, document errors
- **Other interfaces:** CLI commands, message queues, webhooks, file formats

### Step 4: Define Scope with User

Present your findings and surface decisions that need to be made:
- Propose goals and non-goals based on what you've learned
- List each decision point with options and trade-offs
- Propose a recommended option where you have enough context — **default to the simplest option that achieves the outcome.** The burden of proof is on the more complex option, not the simpler one
- Flag things you couldn't resolve

**Run a simplicity pass before locking scope.** For each goal, ask: "does removing or shrinking this still achieve the outcome?" Anything that survives only because it's "nice to have," "might be needed later," or "would handle scale we don't have" moves to Non-Goals with a one-line reason.

Iterate until the scope is locked — all decisions are either resolved or explicitly deferred to Open Questions.

### Step 5: Draft the PRD

Create the file at `docs/YYYY_MM_DD_feature_name/prd.md` using today's date and a snake_case feature name.

Follow the Document Structure below. **Only include sections that apply** — omit any conditional section that has no content.

### Step 6: Review with User

Present the draft. Iterate on corrections, missing details, or structural changes until the user approves.

## Document Structure

### Required Sections (always present)

#### 1. Problem Statement
Why does this feature need to exist? What pain point are we addressing or what opportunity are we capturing? 2-4 sentences that ground everything that follows.

#### 2. Goals & Non-Goals

**Goals** — What this feature will achieve. Be specific and measurable where possible.

**Non-Goals** — What is explicitly out of scope. This prevents scope creep and misaligned assumptions. Include complexity that was *deliberately simplified away* during the simplicity pass — caching, batching, configurability, edge-case polish, scale concerns — each with a one-line reason (e.g. "Non-goal: bulk import — current volume is ~10/day, manual entry suffices"). Cut scope written down is a decision; cut scope omitted creeps back in during user stories.

#### 3. Feature Description
What the feature does, described from the user's perspective. Focus on outcomes and behaviors, not implementation. This is the "what it looks like when it's done" section.

### Conditional Sections (include only when applicable — omit entirely otherwise)

#### 4. User Flows
Step-by-step user interactions when there's a UI component or multi-step process. Describe the happy path first, then variations and error states.

#### 5. Technical Context
Relevant architecture, existing patterns, and constraints that shape the solution. This is where you reference existing code patterns, explain why certain approaches are preferred, and note technical limitations.

When existing code demonstrates the pattern to follow, include a reference table:

| File | What to reference |
|------|-------------------|

#### 6. API Surface
When the feature exposes or consumes API endpoints. Organize into sub-sections as needed:
- **New Endpoints** — Method, path, description
- **Modified Endpoints** — Method, path, what changed
- **Request Schemas** — Field tables with type, required, constraints
- **Response Schemas** — Field tables with type and notes
- **List Endpoint Filters** — Query parameters
- **Error Formats** — Error response shapes
- **Enums** — Value tables with descriptions

Only include the sub-sections that have content.

#### 7. Data Model Changes
When the feature involves database or storage changes:
- **New tables/collections** — Columns, types, constraints, indexes
- **Modified tables** — New columns, changed types, new indexes
- **Migrations** — Ordering, reversibility, data backfills
- **Key relationships** — Foreign keys, cascading behavior

#### 8. Breaking Changes
When existing behavior changes in backward-incompatible ways. Each change gets:
- A numbered heading describing the change
- **Impact:** What breaks and where
- **Action:** What must be done to handle it

#### 9. Success Criteria
How we know this feature is working. Measurable outcomes, acceptance criteria, or observable behaviors that indicate the feature is complete and correct.

#### 10. Configuration & Environment
When the feature introduces new configuration:
- New environment variables
- Feature flags
- Config file changes
- New dependencies or service connections

### Required Sections (always last)

#### 11. Key Files
Two tables:

**Existing Files to Modify:**
| File | Changes |
|------|---------|

**New Files to Create:**
| File | Purpose |
|------|---------|

This section serves as a navigational summary of everything discussed above.

#### 12. Open Questions
Unresolved unknowns, things deferred to future investigation, dependencies on decisions from others. Each item explains what's unknown and why it couldn't be resolved now.

If there are no open questions, state "None — all decisions resolved during requirements gathering."

## Anti-Patterns to Avoid

- **Pasting code blocks** — They go stale. Point to files with line numbers instead
- **Including empty sections** — If "Breaking Changes" doesn't apply, omit it entirely
- **Leading with technical details** — Problem and goals come first. Technical context supports
- **Writing implementation instructions** — This is a PRD, not a how-to guide. Describe *what* needs to happen, not *how* to code it. Implementation breakdown belongs in user stories
- **Skipping the research step** — Don't write a PRD from imagination. Read the actual code first
- **Being vague about scope** — "Support advanced filtering" is not a goal. "Users can filter invoices by status, date range, and customer" is
- **Designing for imagined scale** — Don't plan for load, volume, or concurrency you're nowhere near having
- **Speculative generality** — No config options, abstractions, extension points, or "phase 2 hooks" without a concrete, current need
- **Gold-plating the goal list** — Every goal must trace back to the problem statement; if cutting it still achieves the outcome, it's a non-goal

## Output

The final deliverable is a single markdown file:
```
docs/YYYY_MM_DD_feature_name/prd.md
```

This document is the input for the next step: writing user stories (handled by the `/write-user-stories` skill).
