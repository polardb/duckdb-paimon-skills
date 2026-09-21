---
name: duckdb-paimon-analyze
description: Query, browse, and analyze Apache Paimon tables using DuckDB and the paimon community extension, through local, OSS, or S3 filesystem warehouses or a REST catalog.
metadata:
  short-description: Analyze Paimon data lake tables with DuckDB
---

# duckdb-paimon-analyze

Use this skill when the user asks to query, browse, or analyze Apache Paimon tables using DuckDB, including schema discovery, snapshots, time travel, and cross-format joins.

## Installation

Install with `INSTALL paimon FROM community;` and load with `LOAD paimon;`. Follow the setup guide to verify availability for the user's DuckDB version and platform.

## Resources

Read [AGENTS.md](AGENTS.md) for the agent workflow covering environment check, installation and loading, connection selection, schema exploration, and query generation. Use `ATTACH` and query tables by their catalog-qualified names, including for single-table tasks. Attach catalogs with `READ_ONLY` by default.

When running analysis queries, follow the SQL visibility protocol in `AGENTS.md`: show each generated SQL statement before execution and include the executed SQL with the results. Redact credential values in connection SQL and never expose them in conversation or tool output.

Reference docs (load on demand):

- [references/setup-guide.md](references/setup-guide.md) -- DuckDB environment checks, community installation, and version verification.
- [references/sql-operations.md](references/sql-operations.md) -- SQL reference: catalog queries, paimon_snapshots, time travel, DDL/DML limits, cross-format joins.
- [references/remote-access.md](references/remote-access.md) -- OSS and S3 credentials, remote filesystem warehouses, and REST catalog connections.
