---
type: decision
question: "What database should we use?"
chosen: "PostgreSQL"
rejected: ["SQLite (doesn't handle concurrent writes)", "MongoDB (no strong consistency guarantees)"]
rationale: "Need concurrent writes, transactions, and dataset will exceed 10GB. Postgres is the safe default."
scope: "Backend services"
status: accepted
created: 2026-04-11
last_verified: 2026-04-11
triggers: [postgres, database, db, sql, data store, sqlite, mongo, rds]
sources: ["Discussion: 2025-11-03 architecture session"]
---

# Why We Chose Postgres

**PostgreSQL** is our default database for all backend services.

## Rationale
- Concurrent write support (SQLite can't handle our write volume)
- ACID transactions for financial/critical data
- Dataset projected to exceed 10GB within 6 months
- Strong ecosystem (extensions, tooling, managed hosting options)

## Rejected Alternatives
- **SQLite**: Great for embedded/edge use cases, but can't handle concurrent writes from multiple service instances.
- **MongoDB**: Evaluated for flexibility, but we need strong consistency guarantees that require extra work in Mongo.

## When to Override
If a specific module needs an embedded database (CLI tool, edge device), SQLite is acceptable. Document the override in the module's wiki article.

<!-- 
TEMPLATE NOTE: Delete this article and create your own decisions.
-->
