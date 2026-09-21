# Environment Setup Guide

Use DuckDB's community repository for installation. Follow the SQL visibility protocol in `AGENTS.md` when executing the checks below.

## Step 1: Check DuckDB and Platform

```bash
uname -s
uname -m
duckdb -version
```

Community extension availability depends on the DuckDB version and platform. Use the [community extension descriptor](https://github.com/duckdb/community-extensions/blob/main/extensions/paimon/description.yml) and the installation result to determine support rather than maintaining a fixed platform list here.

On macOS with Rosetta 2, `uname -m` may report `x86_64` on Apple Silicon. Verify with:

```bash
sysctl -n hw.optional.arm64
```

Use a native ARM64 DuckDB binary on Apple Silicon. If DuckDB is missing from `PATH`, check existing installations under `~/.duckdb/cli/` before installing another copy. Select a DuckDB version supported by the community build and use the [official installation instructions](https://duckdb.org/install/). For a version-specific CLI installation, replace `{duckdb_version}` with the verified version:

```bash
curl -fsSL https://install.duckdb.org | DUCKDB_VERSION={duckdb_version} sh
export PATH="$HOME/.duckdb/cli/{duckdb_version}:$PATH"
duckdb -version
```

Preserve a working existing installation where possible. GitHub's latest release tag is not a reason to replace the user's DuckDB binary.

## Step 2: Inspect, Install, and Load

Inspect the extension in the selected DuckDB binary:

```sql
SELECT extension_name, installed, loaded, extension_version, installed_from
FROM duckdb_extensions()
WHERE extension_name = 'paimon';
```

If Paimon is not installed, install it from the community repository:

```sql
INSTALL paimon FROM community;
LOAD paimon;
```

If already installed, run `LOAD paimon;` to verify compatibility and repeat the inspection query after loading. Record the DuckDB version, extension version, and installation source. A file's existence is not sufficient verification.

`INSTALL` reuses an existing installation; it does not necessarily upgrade it or change its source. When the task requires replacing an older or differently sourced installation with the community build, use:

```sql
FORCE INSTALL paimon FROM community;
```

Then start a fresh DuckDB process, load the extension, and inspect it again so an already loaded binary is not mistaken for the replacement. Do not replace a user-selected custom build without a task-specific reason.

## Step 3: Handle Compatibility and Installation Failures

- **No build for this DuckDB version/platform** — Check the community descriptor and installation error for supported targets. Select a compatible DuckDB version; do not assume the newest DuckDB release already has a Paimon build.
- **Network or repository failure** — Diagnose connectivity to the community repository. A network failure is not evidence of version incompatibility.
- **Version mismatch on load** — Verify which DuckDB binary is running and the extension's installation source; use the build matching that binary.
- **Missing feature** — Compare the installed extension with the published release and community descriptor before consulting upstream `main`.

Sources: [Paimon community extension](https://duckdb.org/community_extensions/extensions/paimon), [upstream README](https://github.com/polardb/duckdb-paimon#install-and-load).
