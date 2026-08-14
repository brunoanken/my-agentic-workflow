# Backend Story Template

Use this template for stories where the primary deliverable is system internals: an endpoint, migration, service logic, business rules, configuration, or data processing.

## Story Format

```markdown
### Story [N]: [Title — imperative verb + noun]
**Type:** Backend | **Status:** Not Started
```

Valid statuses: `Not Started`, `In Progress`, `Completed`, `Dropped`, `Blocked`

The story body is expressed as BDD scenarios:

```markdown
**Scenario: [descriptive name]**
**Given** [precondition / initial state]
**When** [action / trigger]
**Then** [expected outcome]
```

Include multiple scenarios per story: at minimum the happy path and relevant error paths.

## Mandatory Sections

Every backend story MUST include all of the following sections:

### 1. Objective
2-3 sentences describing what this story delivers at the system level. Focus on the capability being added or changed, not the implementation details.

### 2. Scenarios
Given-When-Then scenarios describing all expected behaviors. Group them logically:

```markdown
**Scenario: Successfully create a credit memo**
**Given** a user with AP write permissions and an existing vendor
**When** they POST to `/api/credit-memos` with valid data
**Then** the system creates a credit memo with status `OPEN` and returns 201

**Scenario: Reject credit memo with invalid vendor**
**Given** a user with AP write permissions
**When** they POST to `/api/credit-memos` with a non-existent vendor ID
**Then** the system returns 404 with error detail "Vendor not found"
```

### 3. Acceptance Criteria
Checklist of conditions for "done" beyond the scenarios. These capture cross-cutting concerns, non-functional requirements, or constraints that don't fit neatly into Given-When-Then.

```markdown
- [ ] [Criterion 1]
- [ ] [Criterion 2]
```

### 4. Dependencies
List of story numbers this story depends on, and what each dependency provides.

```markdown
- **Story 1** — Provides the credit_memos table and migration
```

If there are no dependencies, state "None."

### 5. Scope Boundaries
What is explicitly NOT part of this story. Be specific about adjacent functionality that might be assumed to be in scope but isn't.

### 6. Test Plan
Follow the test plan structure defined in the main SKILL.md:
- **Required setup** — State/data/fixtures that must exist
- **Test scenarios** — Name, preconditions, actions, expected assertions (exact status codes, response bodies, state changes)
- **Edge cases** — Boundary values, null handling, concurrent requests
- **What NOT to test** — Explicitly scoped out

## Optional Sections

Include these only when they apply to this specific story. Omit entirely if not applicable.

### 7. API Contract
Endpoints, HTTP methods, request/response schemas, status codes, and error formats. Use tables for schemas:

```markdown
**POST /api/credit-memos**

Request:
| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| vendor_id | UUID | Yes | Must reference existing vendor |

Response (201):
| Field | Type | Notes |
|-------|------|-------|
| id | UUID | Generated |
| status | string | Always "OPEN" on creation |

Errors:
| Status | Detail | When |
|--------|--------|------|
| 404 | "Vendor not found" | vendor_id doesn't exist |
```

### 8. Data Model Changes
New or modified tables, columns, constraints, indexes, and migrations. Specify column types, nullability, defaults, and foreign key behavior.

### 9. Performance Criteria
Expected throughput, latency bounds, resource limits, or query performance targets.

### 10. Idempotency & Concurrency
How concurrent requests are handled, retry safety, optimistic locking, or race condition prevention.

### 11. Edge Cases
Boundary values, null handling, race conditions, or unusual input combinations that the implementation must handle correctly.

### 12. Migration & Rollback
How to apply and revert data or schema changes safely. Include ordering constraints if multiple migrations are involved.

### 13. Requirement Traceability
Which specific requirements from the source requirements document this story fulfills. Use the same labels/identifiers used in the requirements doc.
