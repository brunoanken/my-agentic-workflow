---
name: review-postgres-schema
description: Analyze git staged code for PostgreSQL migrations, schema changes, and queries. Delegates to design-postgres-tables and supabase-postgres-best-practices for authoritative guidance. Framework-agnostic.
metadata:
  author: brunozaninello
  version: "1.0.0"
---

# Review Postgres Schema

Analyze staged changes affecting PostgreSQL database structure and queries. This skill orchestrates the review process and delegates to external knowledge sources for best practices.

## Knowledge Sources

Before analyzing, load these references:

1. **design-postgres-tables** (PRIMARY for schema/table design): invoke the
   `design-postgres-tables` skill by name via the Skill tool.

   Covers: data types, constraints, indexing strategies, partitioning, JSONB, table design patterns, anti-patterns.

2. **Supabase Postgres Best Practices** (PRIMARY for query/runtime patterns): invoke the
   `supabase-postgres-best-practices` skill by name via the Skill tool.

   Covers: query performance, connection management, security/RLS, concurrency/locking, data access patterns, monitoring.

Both are third-party skills installed separately — see the Dependencies section of the
my-agentic-workflow README. If either is missing, proceed with the other and note the gap in
your findings.

**Priority**: When sources overlap (data types, indexes, schema design), prefer
`design-postgres-tables`. Use the Supabase skill primarily for query patterns, connection
handling, and runtime concerns.

Cite which source informed each finding.

## Runtime Introspection (Optional)

If **Tidewave MCP** is available and connected, use it to augment static analysis with live database inspection. Tidewave provides framework-agnostic runtime introspection regardless of language (Python/Django, Elixir/Phoenix, etc.).

### Capabilities

| Tool | Purpose | Use Case |
|------|---------|----------|
| `execute_sql_query` | Direct database access | Inspect actual schema, indexes, constraints |
| `project_eval` | Run code in app context | Convert ORM queries to generated SQL |
| `get_models` | List all models | Discover model→table mappings |

### When to Use

1. **Validate ORM queries**: When staged code contains ORM calls, use `project_eval` to convert them to SQL, then analyze the generated query
2. **Cross-reference schema changes**: Compare migration DDL against live schema using `execute_sql_query`
3. **Check existing indexes**: Before flagging "missing index", query `pg_indexes` to verify it doesn't already exist
4. **Multi-schema environments**: Query `information_schema` to inspect tables across schemas

### Example Usage

**Convert ORM to SQL** (framework will vary):
```
# Via project_eval - syntax depends on framework
qs = Model.objects.filter(status='active').select_related('user')
print(str(qs.query))  # Outputs raw SQL
```

**Inspect table schema**:
```sql
SELECT column_name, data_type, is_nullable
FROM information_schema.columns
WHERE table_name = 'orders';
```

**Check indexes**:
```sql
SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'orders';
```

### Priority

- Use Tidewave to **validate and enrich** findings, not replace static analysis
- If Tidewave unavailable, proceed with static analysis only
- Cite "Tidewave introspection" when findings come from runtime inspection

## Process

1. **Load knowledge**: Fetch both sources listed above
2. **Check Tidewave availability**: If Tidewave MCP is connected, use it for runtime validation during analysis
3. **Get staged changes**: Run `git diff --staged`
4. **Identify database files**: Migrations, SQL files, schema definitions, files with SQL queries
5. **Analyze against best practices**: Compare changes to loaded knowledge
6. **Report findings**: Use the output format below
7. **Fix issues**: After presenting findings, fix each issue

## What to Analyze

Focus on these categories (details in knowledge sources):

### Schema & Types
- Data type choices (timestamps, strings, numerics, IDs)
- Primary key strategy
- Constraints (NOT NULL, CHECK, UNIQUE, FK)
- Naming conventions

### Indexes
- Missing indexes on FK columns
- Missing indexes on WHERE/JOIN columns
- Index type selection (B-tree vs GIN vs GiST vs BRIN)
- Composite index column ordering
- Partial index opportunities

### Query Patterns
- N+1 queries
- Unbounded SELECTs (missing LIMIT)
- OFFSET pagination on large tables
- SQL injection risks
- **ORM-generated queries** (if Tidewave available, convert ORM calls to SQL for analysis)

### Migration Safety
- Non-concurrent index creation
- Adding NOT NULL without DEFAULT
- Dropping columns still referenced in code

## Output Format

Group findings by **impact**, from highest to lowest. Do NOT group by file.

Impact reflects the combined cost to the **database** (data loss, corruption, constraint violations, migration failures) and the **application** (query performance, scalability, security, correctness).

Use these three impact levels:

**High Impact** — Must fix. Examples: missing indexes on FK columns used in JOINs/CASCADE, data loss risks from unsafe migrations (dropping columns still referenced), SQL injection, missing constraints that allow corrupt data, non-concurrent index creation on large tables causing downtime.

**Medium Impact** — Should fix. Examples: suboptimal data type choices (SERIAL instead of IDENTITY, VARCHAR(n) instead of TEXT), N+1 queries, OFFSET pagination on large tables, missing partial indexes for soft-delete patterns, adding NOT NULL without DEFAULT on populated tables.

**Low Impact** — Nice to have. Examples: naming convention improvements, composite index column reordering for marginal gains, minor schema normalization, idiomatic SQL rewrites.

For each finding:
1. Category tag (e.g., `[Schema]`, `[Index]`, `[Query]`, `[Migration Safety]`, `[Security]`)
2. File and line reference in `path:line` format
3. Brief description of the issue and why it matters
4. Fix (reference which knowledge source informed this)

Within each impact level, order findings from most to least impactful.

## Example Output

### High Impact

- **[Index] `migrations/001_create_orders.sql:5`** — FK `user_id` has no index. JOINs and CASCADE operations cause full table scans.
  - Source: design-postgres-tables ("FK indexes: PostgreSQL does not auto-index FK columns")
  - Fix: `CREATE INDEX ON orders (user_id);`

- **[Migration Safety] `migrations/003_add_status.sql:2`** — Adding `NOT NULL` column without `DEFAULT` on a populated table. Migration will fail or lock the table for a full rewrite.
  - Source: Supabase Postgres Best Practices
  - Fix: Add `DEFAULT 'pending'` or use a multi-step migration (add nullable → backfill → set NOT NULL).

### Medium Impact

- **[Schema] `migrations/001_create_orders.sql:3`** — Using `SERIAL` instead of `IDENTITY`. SERIAL has ownership and permission edge cases.
  - Source: design-postgres-tables ("use BIGINT GENERATED ALWAYS AS IDENTITY")
  - Fix: `id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY`

### Low Impact

- **[Schema] `migrations/001_create_orders.sql:8`** — Column `order_name` uses `VARCHAR(255)` instead of `TEXT`. No performance benefit in PostgreSQL.
  - Source: design-postgres-tables
  - Fix: Use `TEXT` with a `CHECK (length(order_name) <= 255)` if a limit is needed.
