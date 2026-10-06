# UI Story Template

Use this template for stories where the primary deliverable is something a user sees or interacts with: a page, form, drawer, component, modal, or visual change.

## Story Format

```markdown
### Story [N]: [Title]
**Type:** UI | **Status:** Not Started
**As a** [role], **I want** [action], **so that** [benefit].
```

Valid statuses: `Not Started`, `In Progress`, `Completed`, `Dropped`, `Blocked`

## Mandatory Sections

Every UI story MUST include all of the following sections:

### 1. Objective
2-3 sentences describing what this story delivers and what the user can do after it's implemented. Focus on the outcome, not the implementation.

### 2. Acceptance Criteria
Checklist of conditions that must be true for this story to be considered "done." Each criterion is a concrete, verifiable statement.

```markdown
- [ ] [Criterion 1]
- [ ] [Criterion 2]
```

### 3. Interaction Flow
Step-by-step description of how the user interacts with the feature. Describe what the user does (clicks, types, navigates) and what they see in response (transitions, feedback, state changes).

### 4. Visual States
What the UI looks like in each possible state:
- **Loading** — Skeleton, spinner, or placeholder
- **Empty** — No data available, zero-state message or CTA
- **Populated** — Normal state with data
- **Error** — Error messages, retry options
- **Disabled** — When actions are unavailable and why

Only include states that apply to this story.

### 5. Dependencies
List of story numbers this story depends on, and what each dependency provides. Example:

```markdown
- **Story 2** — Provides the API endpoint for fetching invoice data
```

If there are no dependencies, state "None."

### 6. Scope Boundaries
What is explicitly NOT part of this story. Be specific about what might be assumed to be in scope but isn't.

### 7. Test Plan
Follow the test plan structure defined in the main SKILL.md:
- **Required setup** — State/data that must exist
- **Test scenarios** — Name, preconditions, actions, expected assertions
- **Edge cases** — Unusual inputs, boundary conditions
- **What NOT to test** — Explicitly scoped out

Each accessibility criterion in section 8 gets a scenario here, written with role and label locators.

### 8. Accessibility
Produced by the `review-accessibility` skill in requirements mode — invoke it via the Skill tool
rather than writing this section from memory. These are acceptance criteria: the story isn't done
until they hold.

- **Criteria** — Testable statements naming the role and accessible name a test locates the element
  by. Example: "After activating *Delete invoice*, focus is inside `dialog` named *Delete INV-001?*;
  Escape closes it and focus returns to the *Delete invoice* button." Cite the WCAG 2.2 success
  criterion each one comes from.
- **Primitives to reuse** — The repo's existing accessible components and helpers (dialog, form
  field, announcer) that satisfy the criteria, with file paths.

Scale it to the story: one new button gets one or two criteria. If the story adds no interactive or
visual surface, write "No new interactive or visual surface."

## Optional Sections

Include these only when they apply to this specific story. Omit entirely if not applicable.

### 9. Component Reuse
- **Existing components to use** — File paths to components that should be reused
- **New components to create** — Components that need to be built for this story

### 10. Edge Cases
Unusual user behaviors, boundary inputs, concurrent actions, or race conditions that the implementation should handle.

### 11. Requirement Traceability
Which specific requirements from the source requirements document this story fulfills. Use the same labels/identifiers used in the requirements doc.
