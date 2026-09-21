# duckdb-paimon-analyze — AI Agent Workflow Guide

> This document instructs AI agents on how to set up and use DuckDB + duckdb-paimon to analyze Apache Paimon data lake tables. Follow the phases in order; skip completed phases on subsequent activations within the same conversation.

## Before Use: Check Skill Updates

When this skill is activated **for the first time in a conversation**, check for newer commits on the remote:

```bash
git -C <skill_directory> fetch origin main
git -C <skill_directory> log --oneline HEAD..origin/main
```

If newer commits exist, summarize them and ask the user whether to update. If the check fails (network, credentials), continue with the local version.

## Phase 1: Environment Check

Follow `references/setup-guide.md` to detect the platform, locate or install DuckDB, and inspect the installed Paimon extension version and source.

Outcomes:
- **Extension already installed** — Verify that it loads with this DuckDB binary in Phase 2.
- **Extension not installed** — Proceed to Phase 2.

## Phase 2: Extension Installation & Loading

Follow `references/setup-guide.md` to install from the community repository when needed and run `LOAD paimon;`. Record the verified DuckDB version, extension version, and installation source. A successful load is required even if an extension file already exists.

## Phase 3: Connection Selection & DuckDB Invocation

Use connection details already provided by the user and ask only for missing information:

- **Filesystem warehouse** — A local path, `oss://` URI, or `s3://` URI; attach it to browse databases and tables.
- **Single table** — Identify its warehouse, database, and table, then attach the warehouse and query the catalog-qualified table name. If only a table path is provided, resolve its warehouse root from the layout or ask for the missing connection details.
- **REST catalog** — The service URI, server-side warehouse identifier, and authentication source. The warehouse identifier is not a filesystem path.

For OSS, S3, or REST, follow `references/remote-access.md`. Reuse an existing S3 credential chain/profile when available; request a local credential file only when needed. Load the extension before creating Paimon Secrets, then create Secrets before attaching the remote catalog.

Run DuckDB non-interactively. After installation, use this stdin-script pattern for a local filesystem warehouse:

```bash
duckdb <<'SQL'
LOAD paimon;
ATTACH '/path/to/warehouse' AS paimon_cat (TYPE paimon, READ_ONLY);
.timer on
-- analysis SQL statements go here
SQL
```

Use `READ_ONLY` by default to prevent accidental writes. Drop `READ_ONLY` only when the user explicitly intends to write data and the installed version and catalog support the operation; see `references/sql-operations.md` for write limits.

Each new DuckDB process needs `LOAD` and the relevant Secrets and attachments again. Retain the verified installation details across the conversation, but do not treat temporary Secrets or attached catalogs as persistent across processes.

### Verify

```sql
SHOW ALL TABLES;
```

For an attached catalog, this should list its databases and tables. If empty, verify the filesystem warehouse root and metadata beneath its table directories, or the REST warehouse identifier and catalog permissions.

## Phase 4: Schema Exploration

Help the user understand what data is available:

```sql
SHOW ALL TABLES;
DESCRIBE paimon_cat.db_name.table_name;
SELECT * FROM paimon_cat.db_name.table_name LIMIT 5;
```

Present the schema information clearly before generating analysis queries. Understanding column names, types, and sample values is essential for producing correct SQL.

## Phase 5: Query & Analysis

Generate SQL queries based on the user's analysis requirements. See `references/sql-operations.md` for examples of catalog queries, time travel, snapshot inspection, write operations, and cross-format joins. These examples are not an exhaustive capability list; for a requested feature not covered here, consult the current upstream documentation and verify support in the installed extension.

### SQL Visibility Protocol

For every SQL statement that is generated and executed on the user's behalf:

1. Show the exact SQL to the user in a fenced `sql` code block before executing it. For credential-bearing setup SQL only, replace sensitive values with explicit redaction placeholders; keep the remaining SQL visible.
2. Execute only SQL that has already been shown, except for trivial session setup commands already documented in earlier phases. Credential substitution from the user's local configuration is the only permitted difference from displayed setup SQL; never print the substituted SQL or raw credentials.
3. Report the `Run Time (s)` line printed by DuckDB for each analysis query. Do not use agent-side wall-clock timing as the query time.
4. When presenting results, include an "Executed SQL" section that lists the statements used to produce those results. If several exploratory statements were run, include all of them in execution order. Preserve credential redaction in any setup SQL included there.
5. If a statement must be changed after an error, show the revised SQL before running it.

Do not summarize results from hidden ad hoc SQL. The user must be able to see which SQL produced each answer.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Community installation unavailable | Unsupported DuckDB version/platform, or repository/network failure | Check `references/setup-guide.md`; distinguish missing builds from network errors before choosing a compatible version or resolving connectivity |
| `Extension ... version mismatch` | DuckDB version != extension build version | Re-run Phase 1 to verify version alignment |
| `SHOW ALL TABLES` returns empty | Wrong warehouse, missing table metadata, or catalog permissions | Check filesystem table directories or the REST warehouse identifier and permissions |
| Remote access denied | Wrong credential scope, expired credentials, endpoint, or permissions | See `references/remote-access.md` troubleshooting section |
| `Table scan returns 0 rows` | Querying wrong snapshot or empty table | Check the selected snapshot and filters; use `paimon_snapshots()` for filesystem tables |
