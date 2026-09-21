# duckdb-paimon-analyze

Make duckdb-paimon ready for agentic data analysis. (🤖 + 🦆)

This repository contains the `duckdb-paimon-analyze` skill, a reusable skill for analyzing Apache Paimon warehouses with DuckDB and the duckdb-paimon extension.

## What It Does

- Checks the local DuckDB and duckdb-paimon environment.
- Installs and loads the Paimon extension from DuckDB's community repository.
- Attaches local, OSS, or S3 filesystem warehouses and REST catalogs.
- Queries tables through attached catalogs, including for single-table analysis.
- Explores tables, schemas, snapshots, and sample rows.
- Helps produce analysis SQL with read-only access by default.

## When To Use It

Use this skill when an agent needs to inspect, query, or analyze data stored in an Apache Paimon warehouse through DuckDB.

It is intended for workflows such as schema discovery, snapshot inspection, time travel queries, warehouse troubleshooting, and joining Paimon tables with other formats supported by DuckDB.

## Documentation

- [SKILL.md](SKILL.md) — Skill entry point and reference routing.
- [AGENTS.md](AGENTS.md) — Agent workflow and SQL visibility protocol.
- [Environment setup](references/setup-guide.md) — Community installation and version verification.
- [Remote access](references/remote-access.md) — OSS, S3, and REST catalog configuration.
- [SQL operations](references/sql-operations.md) — Queries, snapshots, time travel, and write operations.

## Safety

The workflow attaches warehouses as read-only by default. Write access should only be enabled when the user explicitly intends to modify data.

Reuse an existing S3 credential chain/profile where available. For static object storage credentials or REST tokens, provide a local credential file path instead of pasting secrets into conversation history. Display credential-bearing SQL with sensitive values redacted.
