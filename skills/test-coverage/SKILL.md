---
name: test-coverage
description: Write tests for staged changes and audit test assertion quality. Analyzes the staged diff, finds existing coverage to extend, writes e2e/integration tests that mimic real user workflows, and holds every assertion to an exact-value bar (identity over count, state over status). Its defaults yield to prompt-level instructions and project conventions in CLAUDE.md/AGENTS.md. Use when adding test coverage, reviewing existing tests, or strengthening weak assertions.
metadata:
  author: Bruno Zaninello
  version: "2.1.0"
---

# Test Coverage

Cover staged changes with tests, and hold every assertion — existing or new — to a bar where it fails only for real bugs.

Writing and auditing are one job, not two. The audit standards below are the acceptance criteria for the tests you write.

## Precedence

Everything in this skill is a **default**, not a rule. It loses to more specific direction, in this order:

1. **The user's prompt in this conversation** — highest authority
2. **Project instructions** — `CLAUDE.md`, `AGENTS.md`, `.cursorrules`, or a `TESTING.md` in the repo
3. **Existing conventions in the test files you're editing** — match the surrounding code
4. **This skill** — the fallback when nothing above speaks to the question

When an override applies, follow it without arguing for the default, and without asking
permission to comply. Note in one line what you followed and why, then move on. Do not
re-raise it later in the same task.

**Read the project's instruction files before Step 4** (test strategy). If they specify a
framework, fixture pattern, file layout, naming convention, or coverage policy, those win
over anything here.

Examples of overrides and the correct response:

| The user or project says | Do this |
|---|---|
| "no e2e tests" / "unit tests only" | Write unit tests. Drop the workflow-test shape and the e2e/integration preference entirely — keep the assertion bar, which is stack-independent. |
| "one assertion per test" / project uses focused tests | Abandon "assert many things per test" and the consolidation preference. Write small tests that each clear the bar. |
| "new file per feature" | Skip the enhance-existing-test priority in Step 4. |
| "just audit, don't write anything" | Stop after Step 3 and report. Change nothing. |
| "skip the audit, just add coverage" | Skip Step 3. Still self-audit your own tests at Step 6 — that's how you know what you wrote is worth keeping. |
| "don't touch the existing tests" | Write new tests only. Report weak assertions you noticed without editing them. |
| "just get it passing, don't over-engineer" | Cover the main path. Say which edge cases you skipped so the decision is visible. |
| Project mandates a framework/fixture/factory pattern | Use theirs, even where it conflicts with the examples in the reference files. |

If an override reduces what the tests actually prove — dropping state verification,
accepting count-only assertions — comply, and say plainly in one sentence what the tests
no longer catch. State it once as information, not as a case to reconsider.

The reference files ([API_ASSERTIONS.md](./API_ASSERTIONS.md),
[UI_ASSERTIONS.md](./UI_ASSERTIONS.md)) are subject to this same precedence. Their code
samples illustrate patterns; they are not a required style.

## Philosophy

These are the defaults. **On test shape** is the more situational half — project
conventions and explicit instructions override it routinely. **On assertion strength**
holds across stacks and test sizes, so keep it unless told otherwise directly.

**On test shape** — e2e and integration tests are expensive to run and maintain. Aim for minimal test count with maximum coverage:

1. **Mimic real user behavior** — test complete workflows, not isolated units
2. **Assert many things per test** — validate the full journey, not a single outcome
3. **Consolidate** — enhance an existing test before creating a new one

**On assertion strength** — a test that passes without proving anything is worse than no test, because it buys false confidence:

4. **Verify exact expected state**, not that "something happened"
5. **Every mutation gets a state verification** — if you changed data, prove the change is correct
6. **Assertions must fail for the right reasons** — specificity means a failure points at a bug, not a coincidence

## The Bar

Every assertion should clear this. Use it both to audit existing tests and to check your
own. This bar is stack- and size-independent — it applies to a unit test as much as an
e2e test, so it survives most overrides to test *shape*. Drop rows from it only when the
user or the project asks you to.

| Weak pattern | The bar |
|---|---|
| Count/length only — `assert len(results) == 3` | Assert the identity set too: which specific IDs came back |
| Truthy/existence — `assert result`, `assert x is not None` | Assert the value: `assert result.amount == 100.00` |
| Status only — `assert status == 200` | Assert status **and** the body fields that matter |
| Mutation with no state check | Re-read the persisted record; assert exact field values |
| No side-effect check | Assert every entity the action touches, not just the primary one |
| Unchanged fields unverified on update | Assert the changed field **and** that the others held |
| Error case by status alone — `assert status == 400` | Assert which field failed and the message |
| Visibility only (UI) — `expect(el).toBeVisible()` | Assert exact text content |
| Arbitrary wait (UI) — `waitForTimeout(2000)` | Wait on a specific condition, then assert it |
| Filter tested only positively | Assert the expected items **and** the exclusions |

Two rules that follow from the table and are worth stating outright:

- **Identity beats count.** Count passes on stale data, coincidental matches from unrelated tests, and silently-failed setup. Store IDs at creation and verify them in the response. Count is a secondary assertion.
- **Don't trust the response alone.** An API can echo back a value it never persisted. Re-query the database for anything that mutated.

## Workflow

### Step 1 — Read the change and the ground rules

`git diff --cached` to understand:
- What models, services, endpoints, or UI were added
- Which existing features the change affects
- The user-facing behavior and business rules it supports

At the same time, collect the constraints that override this skill's defaults:
- Re-read the user's prompt for scope limits ("unit tests only", "just audit", "don't touch existing tests")
- Check `CLAUDE.md` / `AGENTS.md` and any `TESTING.md` for test conventions, required frameworks, or coverage policy
- Note how tests are already run in this project (test command, fixture helpers, factories)

If an instruction conflicts with something below, follow the instruction. See **Precedence**.

### Step 2 — Locate existing coverage

Search for tests covering the same feature area, related functionality the new code extends, and parent features it integrates with. Note the conventions in use — assertion style, setup helpers, file naming — and match them rather than the reference files' examples.

### Step 3 — Audit what already exists

Run the existing tests in that area against **The Bar**. For each test: identify the mutation under test, list its assertions, and flag what's missing.

Categorize by severity:

- **Critical** — passes without verifying the behavior at all: no body assertions after a mutation, missing state verification on money or other irreversible data, count-only checks on the thing the test exists to prove
- **Warning** — incomplete: status without body, existence without value, related entities unverified
- **Suggestion** — could be sharper: add before/after comparison, verify more fields, cover an edge case

Report findings before changing anything (format below).

**If the user only asked to audit, stop here.** Present the report and ask which fixes to apply. Likewise, if they asked you to skip the audit, skip this step.

### Step 4 — Decide strategy

Project conventions win here. If the repo's instructions or existing layout dictate file organization, follow that and skip this table.

| Situation | Approach |
|---|---|
| Change extends a feature already under test | Add assertions to the existing test |
| New independent feature in a tested area | Add a test case to the existing file |
| Entirely new area | New test file with a comprehensive workflow test |

Don't create a new file for a minor extension. A discounts feature belongs in the invoice test file, not `test_invoice_discounts`.

### Step 5 — Write and strengthen, one at a time

Write ONE test (or fix ONE weak assertion) → run it → iterate until it passes → move on. Never write the batch and run at the end. This step holds regardless of test shape or size — it's how you avoid a pile of tests that have never run.

Structure tests as workflows — unless the project or the user prefers focused tests, in which case write those and keep the assertions specific:

```
test "complete feature workflow" {
    // Setup: realistic preconditions, isolated to this test
    parent = POST /api/parents { name: "Parent {timestamp}" }

    // Action 1
    response = POST /api/resources { parent_id: parent.id, ...payload }
    assert response.status == 201
    assert response.body.parent_id == parent.id
    assert response.body.status == "draft"        // default applied
    resource_id = response.body.id

    // Action 2 — uses the result of action 1
    response = GET /api/resources/{resource_id}
    assert response.status == 200
    assert response.body.name == "expected name"

    // Action 3 — complete the workflow
    response = PATCH /api/resources/{resource_id} { status: "completed" }
    assert response.status == 200

    // Verify final persisted state, not just the response
    final = db.resources.find(resource_id)
    assert final.status == "completed"
    assert final.completed_at is not null
    assert final.related_items.count == 3

    // Verify side effects
    log_entries = db.activity_log.where(resource_id: resource_id)
    assert log_entries.count == 1
    assert log_entries[0].type == "completed"
}
```

Anti-pattern — one workflow split into four shallow tests:

```
// Bad
test "create invoice" { ... }
test "get invoice" { ... }
test "update invoice" { ... }
test "invoice creates ledger entry" { ... }

// Good: one test, the full lifecycle, assertions at every step
test "invoice lifecycle" { ... }
```

### Step 6 — Self-audit

Before reporting done, re-read the tests you just wrote against **The Bar**. Writing tests and grading them are the same standard — apply it to your own work.

### Step 7 — Report

State which functionality is now covered, which existing tests you strengthened, and anything you deliberately left uncovered and why. If you departed from this skill's defaults because of an instruction or project convention, say so in one line — so the choice is visible, not so it can be relitigated.

## Scope Guard

If a test fails because of a real product bug:

- **Bug is in the staged change** — fix it, and leave the test as the regression test.
- **Bug is in pre-existing code** — report it and ask before touching product code. Don't expand the change to fix things the user didn't ask about.

Never weaken an assertion just to make a test pass. A failing assertion is either a bug or a wrong expectation — determine which. (If the user explicitly tells you to loosen a specific assertion, that's their call: do it, and note what it stops catching.)

## Test Data Isolation

Integration and e2e tests run against shared databases. Isolate to avoid flakes — unless the project already has an isolation strategy (transactional rollback, per-test schema, seeded fixtures), in which case use theirs:

1. **Scope to a unique parent entity** — create your own vendor/customer/tenant/role per test
2. **Use unique names** — append a timestamp: `"Test Vendor {timestamp}"`
3. **Filter queries by that parent** — `?vendor_id={vendor.id}`
4. **Assert IDs, not counts** — count is secondary

## Context-Specific Guidance

- **API and database tests** — see [API_ASSERTIONS.md](./API_ASSERTIONS.md): CRUD assertions, derived-state and multi-entity verification, list/filter/ordering/pagination, database state, audit fields
- **UI and Playwright tests** — see [UI_ASSERTIONS.md](./UI_ASSERTIONS.md): data display, visual state, user flows, forms and validation, navigation

## Audit Report Format

Group by file, then by test, then by severity. Every finding names the line, quotes
the weak assertion, and says **what bug would pass undetected** — not just that it's
weak. Follow each with a concrete before/after code block.

~~~
## Test File: `path/to/test_file.ext`

### test_function_name

**Critical:**
- [ ] Line 42: `assert len(results) == 3` — count-only; a stale record from a prior
      run satisfies this. Should verify the specific IDs created.

**Warnings:**
- [ ] Line 58: `assert response.status == 201` — no body verification; would pass
      if the endpoint returned the wrong resource.

**Suggestions:**
- [ ] Consider capturing the balance before the payment and asserting the delta.

**Proposed fix (line 42):**

```
// Current
assert len(results) == 3

// Improved
assert len(results) == 3
result_ids = {r.id for r in results}
assert result_ids == {item1.id, item2.id, item3.id}
```
~~~
