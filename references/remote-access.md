# Remote Warehouse and Catalog Access

Choose the connection mode before requesting credentials. Filesystem catalogs use an `oss://` or `s3://` warehouse URI; REST catalogs use a server URI and a server-side warehouse identifier. REST is a catalog interface, not an object storage scheme. Load `paimon` before the setup SQL below.

## Credential Handling

Reuse an existing S3 credential chain/profile when available. For static credentials or REST tokens, ask the user to provide a local credential file path instead of pasting values into the conversation. Parse the file as data, not as a shell script. Keep credentials out of tool output, command-line arguments, generated artifacts, and version control.

The SQL examples below are templates. Before execution, show the actual setup SQL with sensitive fields replaced by `<redacted>` and identify which local configuration supplies them. Substitute secrets inside the executing process and pass SQL through stdin without echoing it; escape embedded single quotes as `''`. Sanitize error output before returning it because errors may include input SQL. DuckDB Secret Manager redaction does not protect shell logs or an echoed SQL script.

Create temporary Secrets in each new DuckDB session and set `SCOPE` to the warehouse URI used by `ATTACH`. Credential lookup uses that warehouse URI, so a narrower table-path scope will not match. Use one credential provider for each scope; do not create both S3 examples for the same scope.

## Filesystem Catalog: Alibaba Cloud OSS

A local credential file can use:

```text
KEY_ID=your-access-key-id
SECRET=your-access-key-secret
ENDPOINT=oss-cn-hangzhou.aliyuncs.com
```

Read the file without printing its contents and create a scoped Secret:

```sql
CREATE SECRET my_oss (
    TYPE paimon,
    PROVIDER config,
    KEY_ID '<redacted>',
    SECRET '<redacted>',
    ENDPOINT 'oss-cn-hangzhou.aliyuncs.com',
    SCOPE 'oss://your-bucket/warehouse'
);

ATTACH 'oss://your-bucket/warehouse' AS my_catalog (TYPE paimon, READ_ONLY);
```

Choose the endpoint matching the bucket's region. An internal endpoint such as `oss-cn-hangzhou-internal.aliyuncs.com` is appropriate only when reachable from the user's Alibaba Cloud network.

For temporary OSS credentials, include `SESSION_TOKEN` in the local credential file and add `SESSION_TOKEN '<redacted>'` to the Secret. Supply `REGION` when required by the storage configuration.

## Filesystem Catalog: Amazon S3

### Existing Credential Chain

Prefer the user's existing AWS credential configuration, including a selected AWS CLI profile and refreshed SSO credentials when available:

```sql
CREATE SECRET my_s3 (
    TYPE paimon,
    PROVIDER credential_chain,
    PROFILE 'default',
    REGION 'ap-northeast-2',
    SCOPE 's3://your-bucket/warehouse'
);
```

Use the user's profile and region; omit `PROFILE` when relying on the default credential chain rather than an explicitly selected profile. Expired SSO credentials may require the user to refresh their login.

### Static Credentials

When a credential chain is unavailable, use a local file:

```text
KEY_ID=your-access-key-id
SECRET=your-secret-access-key
SESSION_TOKEN=your-session-token
REGION=ap-northeast-2
```

`SESSION_TOKEN` is optional for long-lived credentials and required when the supplied temporary credentials include one. Omit it from both the file and SQL when unused:

```sql
CREATE SECRET my_s3_static (
    TYPE paimon,
    PROVIDER config,
    KEY_ID '<redacted>',
    SECRET '<redacted>',
    SESSION_TOKEN '<redacted>',
    REGION 'ap-northeast-2',
    SCOPE 's3://your-bucket/warehouse'
);
```

After creating either S3 Secret:

```sql
ATTACH 's3://your-bucket/warehouse' AS my_catalog (TYPE paimon, READ_ONLY);
```

For a custom S3 endpoint, configure `ENDPOINT` and, when required by the service, `PATH_STYLE_ACCESS true` in the Secret. Use the service's actual connection settings; an S3-compatible endpoint still needs to be verified with the installed extension.

### Browse and Query Remote Tables

For either attached filesystem catalog:

```sql
SHOW ALL TABLES;
SELECT * FROM my_catalog.db_name.table_name LIMIT 10;
```

Use the same attached-catalog workflow for single-table tasks. For write requests, follow the capability checks in `sql-operations.md` before changing access mode.

## REST Catalog

Obtain the service URI, warehouse identifier, and authentication configuration. For bearer-token access, the local credential file can contain:

```text
TOKEN=your-access-token
```

```sql
ATTACH 'my_warehouse' AS rest_paimon (
    TYPE paimon,
    METASTORE 'rest',
    URI 'https://catalog.example',
    TOKEN_PROVIDER 'bear',
    TOKEN '<redacted>',
    READ_ONLY
);

SHOW ALL TABLES;
SELECT * FROM rest_paimon.your_db.your_table LIMIT 10;
```

`my_warehouse` is sent to the server as its warehouse identifier; `rest_paimon` is only the local DuckDB catalog name. `TOKEN_PROVIDER 'bear'` is the spelling used by the upstream example. Query REST tables through the attached catalog; do not pass the warehouse identifier or REST service URI to path-based `paimon_snapshots`.

REST authentication and access to the underlying data files are distinct. If table metadata is visible but a scan cannot access object storage, check how the REST service and installed extension supply storage credentials for the returned data locations. Do not assume a Secret scoped to a filesystem warehouse also applies to a REST warehouse identifier. For write requests, follow `sql-operations.md`.

## Troubleshooting

| Symptom | Check |
|---------|-------|
| OSS access denied | Credential validity, matching Secret scope, bucket region/endpoint, and object read/list permissions |
| S3 access denied | Profile, refreshed SSO session or session token, region, matching Secret scope, and bucket/object permissions |
| REST authentication failure | Service URI, token validity, token provider, and server access policy |
| REST catalog has no tables | Server-side warehouse identifier and namespace/table visibility |
| Metadata works but data scans fail | Underlying object storage credentials, data location, and network reachability |
| Network timeout | Reachability of both catalog and storage endpoints; internal endpoints require the appropriate network |

Sources: [upstream remote access examples](https://github.com/polardb/duckdb-paimon#query-remote-paimon-tables), [Paimon community extension](https://duckdb.org/community_extensions/extensions/paimon).
