---
name: database-change-modeler
description: Use only for planning and modeling PostgreSQL database changes, schema changes, migrations, indexes, constraints, data migrations, or database design tradeoffs. This skill is for structured thinking and design output, not implementation.
metadata:
  author: Bruno Zaninello
  version: "1.0.0"
---

# Database Change Modeler

Model database changes before implementation. Produce a concise design artifact that explains what should change, why it should change, and how to apply it safely.

This skill is planning-only. The user is interested in the thought process, tradeoffs, and structured output, not code changes.

Do not implement anything while using this skill. Do not write migration files, execute migrations, create schemas/models, edit application code, run database mutations, or apply patches unless the user explicitly makes a separate implementation request after the modeling output.

## When To Use

Use this skill when the user asks to model, design, plan, review, or reason about:

- PostgreSQL schema changes
- New tables or columns
- Migrations
- Indexes
- Constraints
- Relationships and foreign keys
- Data migrations or backfills
- RLS or tenant ownership models
- Query access patterns that should influence schema design
- Database tradeoffs before implementation

## Knowledge Sources

Use external database guidance to augment local codebase analysis.

### Tiger MCP

Use the Tiger MCP server for database modeling guidance. Call the available Tiger MCP tools in the current agent environment, such as `tiger_view_skill`, `tiger_search_docs`, or their MCP-equivalent names.

| Need | Tiger MCP Guidance |
| --- | --- |
| General PostgreSQL modeling | View `postgres` or `design-postgres-tables` |
| Table design, data types, constraints, indexes | View `design-postgres-tables` |
| Time-series, events, metrics, logs, append-heavy data | View `setup-timescaledb-hypertables` |
| Existing schema analysis for TimescaleDB candidates | View `find-hypertable-candidates` when relevant |
| PostGIS or location/spatial data | View `design-postgis-tables` |
| Vector embeddings, semantic search, RAG | View `pgvector-semantic-search` |
| Hybrid keyword and semantic search | View `postgres-hybrid-text-search` |
| Authoritative docs lookup | Use `tiger_search_docs` |

Do not use live database execution tools, such as `tiger_db_execute_query`, unless the user explicitly authorizes live database inspection or execution.

### Supabase Postgres Best Practices

Use `supabase-postgres-best-practices` to augment Postgres runtime and performance judgment, especially for:

- Query performance
- Indexing strategy
- RLS and security
- Migration safety
- Concurrency and locking
- Connection and data access patterns

When Tiger MCP and Supabase guidance overlap, prefer Tiger MCP for schema/table design and Supabase guidance for runtime, operational, and performance concerns.

## Workflow

1. Restate the requested database change and product/business reason.
2. Inspect the existing schema, migrations, ORM schemas/models, contexts/services, and relevant queries when available.
3. Identify affected tables, relationships, constraints, indexes, write paths, read paths, and data migration needs.
4. Use Tiger MCP and Supabase Postgres guidance as relevant.
5. Prefer the smallest correct schema change.
6. Separate schema migration, data backfill, and application rollout steps when safety requires it.
7. Call out assumptions, risks, tradeoffs, and open questions.
8. Return the table-first output format below.

## Planning-Only Boundaries

- Produce a design proposal, not implementation artifacts.
- Do not create, modify, delete, or run migration files.
- Do not create, modify, or delete ORM schemas/models, contexts/services, tests, routes, or UI code.
- Do not execute live database mutation tools or SQL that changes state.
- Read-only inspection is allowed when useful for modeling, but keep the final answer focused on the proposed database model and reasoning.
- If the user asks to implement during the same conversation, treat that as a new explicit implementation request and stop using this skill as the active instruction set.

## Modeling Rules

- Prefer explicit database constraints over application-only validation when the invariant belongs in the database.
- Prefer indexes that support known query patterns. Do not add speculative indexes.
- Include foreign keys unless there is a clear operational reason not to.
- Include `NOT NULL` only when existing data and write paths can satisfy it safely.
- For large existing tables, recommend phased migrations when needed.
- For unique constraints on nullable or conditional data, consider partial unique indexes.
- For tenant-scoped data, include tenant/user/account ownership in table design and indexes.
- For time-series, events, metrics, logs, or append-heavy tables, evaluate whether TimescaleDB hypertables are appropriate.
- For search, geospatial, or vector use cases, call out specialized index types and extensions.
- For destructive changes, explicitly identify data loss risk and safer alternatives.

## Output Format

Use a table-first structure. Group all changes by affected database table.

Keep the output concise. Omit subsections that are not relevant to a table, but always include `Summary`, `Table Changes`, `Risks And Tradeoffs`, and `Recommended Migration Order`.

### Summary

| Item | Value |
| --- | --- |
| Goal |  |
| Reason |  |
| Scope |  |
| Assumptions |  |
| Out of Scope |  |

### Table Changes

#### `table_name`

| Item | Details |
| --- | --- |
| Change | create / alter / drop |
| Purpose |  |
| Ownership / Tenant Scope |  |
| Expected Cardinality |  |
| Write Pattern |  |
| Read Pattern |  |
| Why |  |

##### Columns

| Column | Change | Type | Nullable | Default | Constraints | Why |
| --- | --- | --- | --- | --- | --- | --- |

##### Relationships

| Relationship | Type | Constraint | Delete Behavior | Why |
| --- | --- | --- | --- | --- |

##### Indexes

| Index | Columns / Expression | Type | Predicate | Why |
| --- | --- | --- | --- | --- |

##### Constraints

| Constraint | Type | Definition | Why |
| --- | --- | --- | --- |

##### Data Migration / Backfill

| Step | Action | Safety Notes |
| --- | --- | --- |

##### Query Impact

| Query / Use Case | Expected Access Pattern | Supporting Index / Design |
| --- | --- | --- |

### Cross-Table Changes

Include only when relevant.

| Change | Tables Affected | Why |
| --- | --- | --- |

### Application Impact

| Area | Required Change |
| --- | --- |
| Schemas / Models |  |
| Contexts / Services |  |
| API / LiveView / UI |  |
| Tests |  |

### Risks And Tradeoffs

| Risk | Impact | Mitigation |
| --- | --- | --- |

### Recommended Migration Order

| Step | Migration | Notes |
| --- | --- | --- |

## Output Rules

- Use markdown tables for the model.
- Keep explanations brief and decision-oriented.
- Mark assumptions clearly instead of inventing missing requirements.
- Include open questions only when they block a safe recommendation.
- Never include code diffs, generated migration files, or application code unless the user explicitly requests implementation after the planning output.
- If implementation is requested after modeling, use the migration order as the implementation plan, but do not implement as part of the modeling response.
