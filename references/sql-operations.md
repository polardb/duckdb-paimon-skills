# duckdb-paimon SQL Operations Reference

## Read Operations

Attach the warehouse or REST catalog before querying, even when only one table is needed. Use catalog-qualified table names. For OSS, S3, or REST connection setup, follow `remote-access.md`.

```sql
ATTACH '/path/to/warehouse' AS my_catalog (TYPE paimon, READ_ONLY);
SELECT * FROM my_catalog.db_name.table_name WHERE col1 > 100;
```

## Cross-Format Joins

```sql
SELECT o.order_id, o.amount, c.customer_name
FROM my_catalog.db.orders o
JOIN read_csv('customers.csv') c ON o.customer_id = c.id;

SELECT * FROM my_catalog.db.events e
JOIN read_parquet('dim_users.parquet') u ON e.user_id = u.id;
```

## Snapshot Inspection

```sql
SELECT snapshot_id, commit_kind, commit_time, total_record_count
FROM paimon_snapshots('/path/to/warehouse/db_name.db/table_name')
ORDER BY snapshot_id;

-- Also works with three-argument form and OSS/S3 filesystem paths
SELECT * FROM paimon_snapshots('/path/to/warehouse', 'db_name', 'table_name');
```

`paimon_snapshots` inspects filesystem table metadata by path. Do not substitute a REST service URI or warehouse identifier for the path. Query table data through the attached catalog.

## Time Travel

Use `paimon_snapshots` to identify a filesystem table's snapshot before selecting it. `VERSION` in these examples is a numeric snapshot ID. For tag-based requests or other historical selectors, check the current upstream syntax and installed extension support rather than treating a tag name as a snapshot ID.

```sql
-- By snapshot version
SELECT * FROM my_catalog.db_name.table_name AT (VERSION => 2);

-- By timestamp
SELECT * FROM my_catalog.db_name.table_name
    AT (TIMESTAMP => TIMESTAMP '2026-01-15 10:48:23.5');
```

## Write Operations

Write only when the user explicitly intends to modify data, after confirming the installed version and catalog support the operation. Reattach a supported writable catalog without `READ_ONLY`; do not remove the flag merely to retry a failed analysis query.

The DDL/DML examples below target a writable local filesystem catalog and an append-only table. For other catalog types, primary-key tables, or bucket modes, check the installed extension's support and required options against current upstream documentation. Removing `READ_ONLY` does not enable an operation the extension or catalog cannot perform.

### DDL

```sql
CREATE SCHEMA my_catalog.new_db;

CREATE TABLE my_catalog.new_db.orders AS
    SELECT 1 AS order_id, 99.9::DECIMAL(18,2) AS amount, 'Alice' AS customer;
```

### DML

```sql
INSERT INTO my_catalog.new_db.orders
    SELECT 2, 49.5, 'Bob'
    UNION ALL
    SELECT 3, 150.0, 'Charlie';
```

### Drop Objects

Run only when the user intends to delete these objects, after any queries or inserts that need them:

```sql
DROP TABLE my_catalog.new_db.orders;
DROP SCHEMA my_catalog.new_db;
```

Sources: [upstream usage](https://github.com/polardb/duckdb-paimon#usage), [community release descriptor](https://github.com/duckdb/community-extensions/blob/main/extensions/paimon/description.yml).
