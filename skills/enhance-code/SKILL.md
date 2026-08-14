---
name: enhance-code
description: Scan the code staged in git and search for improvements across correctness, safety, security, performance, maintainability, and over-engineering. Covers bugs, unhandled edge cases, race conditions, security vulnerabilities, N+1 queries, DRY violations, magic strings, speculative abstractions, and scope creep beyond the user story — with removal as a first-class fix. Reports findings grouped by impact.
---

# Enhance Code

Analyze all staged changes and identify opportunities to improve code quality, correctness, and maintainability.

## CRITICAL: Never modify the git staging area

**NEVER run any git command that modifies the staging area (index) or working tree state.** This includes:
- `git stash` / `git stash pop`
- `git reset`
- `git checkout -- .`
- `git restore`
- Any other command that would unstage, restage, or alter staged files

The user's workflow depends on staged changes remaining exactly as they are. To compare against previous versions, use **read-only** commands like `git show HEAD:path/to/file` or `git diff --staged`. Never use destructive commands to check if issues are "pre-existing".

## Scope

Analyze **changed code** (staged or unstaged) and the **full files containing those changes**. If you find issues anywhere in a changed file — even in lines you didn't touch — fix them. Do NOT stash changes, check out old revisions, or compare against previous states to determine if issues are "pre-existing". That distinction is irrelevant. If there's a bug, type error, lint error, or improvement opportunity in the changed files, fix it.

## Process

1. **Get changes**: Run `git diff --staged` (or `git diff` if nothing is staged) to see the changed code
2. **Find the scope anchor**: If this work corresponds to a user story or PRD (look in `docs/YYYY_MM_DD_*/` for `user-stories.md` / `prd.md`, or ask the user which one), read it. The story's acceptance criteria and the PRD's Non-Goals define intended scope. Flag any implemented behavior, endpoint, field, or flexibility the story didn't ask for — especially anything listed in the PRD's Non-Goals — as an over-engineering finding, with removal as the default recommendation. If no anchor document exists, judge over-engineering on general merit.
3. **REQUIRED — Invoke specialized skills**: Before any analysis, check changed file types:
   - If ANY `.tsx`, `.jsx` files or JSX/React patterns are present: you MUST invoke `/vercel-react-best-practices` FIRST. Do NOT proceed to step 4 until this skill has been invoked and its guidelines applied.
   - This covers: re-render optimization, bundle size, server components, data fetching patterns
4. **Analyze each changed file**: Review against every category below — correctness, safety, security, performance, maintainability, and over-engineering. Read the full file if needed for context.
5. **Check for type and lint errors**: Run the project's type checker and linter on the changed files. Fix any errors found.
6. **Report findings**: Present issues grouped by impact (high → low), not by file
7. **Fix issues**: After presenting findings, fix each issue starting from the highest impact

## Analysis Categories

The analysis must cover all of the following areas. Every finding should map to at least one of these categories.

### Correctness
- Logic errors and off-by-one mistakes
- Null/undefined access without guards
- Incorrect error handling or swallowed exceptions
- Type mismatches or unsafe casts
- Unhandled edge cases: empty arrays/objects, boundary conditions (max/min values), missing input validation, network/IO failure scenarios

### Safety
- Race conditions and concurrent modification scenarios
- Thread safety and shared mutable state
- Transaction boundaries and atomicity (e.g., missing `transaction.atomic()` around multi-step writes)
- Data integrity risks (e.g., partial writes, orphaned rows, missing foreign key constraints)
- Resource leaks (unclosed files, connections, subscriptions, timers)
- Idempotency of retryable operations (tasks, webhooks)

### Security
- Injection risks: SQL, command, template, LDAP, NoSQL
- XSS, CSRF, SSRF, open redirects
- Authentication and authorization gaps (missing access checks, privilege escalation paths, IDOR)
- Secrets, credentials, tokens, or PII exposed in code, logs, or error messages
- Insecure cryptography (weak algorithms, hardcoded keys, predictable randomness)
- Unsafe deserialization, path traversal, unvalidated file uploads
- Dependency/supply-chain risks introduced by the change

### Performance
- Inefficient data structures or algorithms
- N+1 queries and missing `select_related`/`prefetch_related` (or equivalent)
- Missing indexes for new query patterns
- Missing memoization for expensive computations
- Redundant computations, network calls, or DB round trips
- Unnecessary re-renders, large bundles, or blocking work on the critical path

### Maintainability
- **DRY violations**: Duplicated logic that should be extracted
- **Magic strings/numbers**: Hardcoded values that should be constants
- **Idiomatic patterns**: Code that could use language/framework conventions
- **Readability**: Overly complex expressions that could be simplified
- **Naming**: Variables or functions with unclear names
- **Cohesion and layering**: Business logic leaking into views/models, violations of the project's architectural conventions
- **Testability**: Code that's hard to test due to tight coupling or hidden dependencies

### Over-engineering (Simplification & Removal)

**The best fix for over-engineered code is deletion. A net-negative diff is a success, not a failure.** Coding agents have a strong bias toward additive fixes — this category exists to push the other way.

- **Scope creep vs. the anchor**: Behavior, endpoints, fields, or flexibility the user story didn't ask for; anything the PRD lists as a Non-Goal
- **Speculative abstractions**: Interfaces or base classes with one implementation, helpers extracted for a single call site, indirection layers that just forward
- **Unused surface**: Parameters, config options, feature flags, enum values, schema fields nothing reads
- **Defensive code for impossible states**: Validating inputs already guaranteed by the caller, gate, or schema layer
- **Noise**: Over-broad try/except, logging nobody will read, comments restating the code
- **Compat shims with no consumers**: Backwards-compatibility paths for code nothing calls
- **Redundant tests**: Tests that test the framework, or duplicate other tests rather than the story's behavior
- **Premature optimization**: Caching, batching, or tuning with no evidence of need

**Removal findings need higher scrutiny than additive ones** — deleting a guard someone needed is worse than leaving a redundant one. Before recommending deletion, grep for call sites and other consumers to confirm nothing depends on it. If the scope anchor is silent on whether something is needed, flag it as a question for the user rather than removing it.

## Output Format

Group findings by **impact**, from highest to lowest. Do NOT group by file.

Impact reflects the combined cost to the **codebase** (risk of regressions, maintenance burden, architectural debt) and the **product** (user-visible bugs, data loss, security breaches, performance degradation, compliance risk).

Use these three impact levels:

**High Impact** — Must fix. Examples: security vulnerabilities, data loss/corruption risks, broken business logic, missing access checks, race conditions on financial data, N+1 queries on hot paths, bugs that will cause runtime errors.

**Medium Impact** — Should fix. Examples: edge cases that will surface under realistic conditions, noticeable performance issues, missing error handling, significant maintainability problems (large DRY violations, leaky abstractions), missing transaction boundaries on non-critical paths, significant over-engineering (unneeded abstraction layers, unrequested features or flexibility, scope beyond the story) — ongoing maintenance burden and review noise on every future change.

**Low Impact** — Nice to have. Examples: minor refactors, naming improvements, small DRY cleanups, magic strings to constants, small over-engineering cleanups (unused parameters, single-use helpers), idiomatic rewrites.

For each finding:
1. Category tag (e.g., `[Security]`, `[Correctness]`, `[Performance]`, `[Safety]`, `[Maintainability]`, `[Over-engineering]`)
2. File and line reference in `path:line` format
3. Brief description of the issue and why it matters (impact on codebase/product)
4. Recommended fix

Within each impact level, order findings from most to least impactful. When a single finding spans multiple categories, tag all that apply.

## Example Output

### High Impact

- **[Security] `apps/accounts_receivable/api.py:142`** — Endpoint accepts `organization_id` but never calls the access gate, allowing cross-org data access within a tenant.
  - Pass `request=request` to `execute_in_tenant_context` so the org access gate runs.

- **[Correctness] [Safety] `apps/payments/services.py:87`** — Payment creation and GL transaction write are not wrapped in `transaction.atomic()`. A failure mid-operation leaves orphaned payments without ledger entries.
  - Wrap both writes in a single `transaction.atomic()` block.

- **[Performance] `apps/reports/services.py:210`** — Balance sheet report iterates invoices in Python and issues one query per customer (N+1). On organizations with 1k+ customers this will time out.
  - Replace the loop with a single aggregated query using `annotate(Sum(...))` and a `select_related('customer')`.

### Medium Impact

- **[Correctness] `apps/invoices/services.py:45`** — `items.length === 0` not handled before division, returns `NaN` for empty invoices.
  - Add guard: `const avg = items.length > 0 ? total / items.length : 0`.

- **[Maintainability] `apps/invoices/services.py:12,34,56`** — Status strings `"pending"`, `"completed"` repeated across the module.
  - Extract to a `PAYMENT_STATUS` constant (see `apps/general_ledger/constants.py` for the project pattern).

- **[Over-engineering] `apps/invoices/export_services.py:1-85`** — CSV export supports configurable delimiters and a pluggable formatter registry; the story asked for a fixed-format CSV download and the PRD lists "export customization" as a Non-Goal.
  - Delete the registry and delimiter option; keep a single function producing the fixed format (~60 lines removed).

### Low Impact

- **[Over-engineering] `apps/utils/helpers.py:23`** — `format_amount` accepts an unused `locale` parameter.
  - Remove the parameter and its call sites (grep confirmed no caller passes it).
