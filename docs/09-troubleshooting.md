# 09 — Troubleshooting (every blocker we hit + the fix)

**Databricks query runtime:** Databricks Runtime 16.4, Scala 2.12
(`16.4.x-scala2.12`), used for both Photon and Standard publication lanes.

These are observed POC incidents, not universal service limitations or proof
that a workaround is production-ready. See [latest results](RESULTS.md) for
the current measurement scope and unresolved correctness limits.

## Storage / governance

**Managed-env forces `pna=Disabled` + `allowSharedKeyAccess=false` on all storage.**
Sub-owner can't flip it (`az storage account update --public-network-access Enabled` silently stays Disabled).
→ Use **trusted-workspace access** (Fabric resource rule) for Fabric; **SP OAuth** for Databricks;
**OneLake source access followed by a native load** for Snowflake.

**`COPY INTO` (Warehouse) fails from ADLS and from OneLake shortcuts.**
The tested private-source path failed; the OneLake shortcut path returned
`Access token couldn't be fetched … unsupported URL`. These observations do
not establish that every private-ADLS configuration is unsupported.
The recorded Warehouse load used **CTAS/`INSERT…SELECT`**. A pipeline Copy
activity was an alternative, not the executed loader.

## Databricks

**UC external location → `PERMISSION_DENIED: Request for user delegation key is not authorized`.**
Early UC setup failed; this is not a general limitation of UC with private storage.
Two paths were tested:
- **Historical direct-read path:** direct `abfss` with **SP OAuth** (`ClientCredsTokenProvider`)
  on the cluster (`SINGLE_USER`). The read goes over the VNet private endpoint.
- **Proper UC path (verified):** UC external tables *do* work against the firewalled account if you:
  1. grant the **Databricks Access Connector** managed identity `Storage Blob Data Reader` on the account;
  2. add that Access Connector as a **trusted resource instance** on the storage firewall
     (`az storage account network-rule add --resource-id <accessConnector> --tenant-id <tid>` — allowed even
     with `pna=Disabled`, same mechanism as the Fabric workspace rule);
  3. create a **dedicated storage credential bound to that Access Connector** — do **not** reuse the
     workspace-default credential (it's path-restricted to the workspace's own storage →
     `PERMISSION_DENIED: ... only allowed to access ... unity-catalog-storage`);
  4. create the **external location** (disable managed file events to avoid a stale-queue update block), then
     `CREATE TABLE … USING DELTA LOCATION` per table.
  **Run it on the VNet-injected cluster, not serverless** — serverless egress isn't allowlisted and fails with
   a storage-firewall error in this environment, whereas the classic cluster worked.
   Current reruns use UC. The historical timing comparison does not isolate UC
   overhead because the timers differ; see [results](RESULTS.md).

**`COUNT(*)` overflow on huge facts.** Use `COUNT_BIG(*)` (warehouse) / ensure 64-bit counts.

## Fabric Spark

**Windows ODBC full-result retrieval remains blocked.** The installed Fabric
Data Engineering ODBC 2.0.0.0 path introduced a negative `GlobalLimit` with large
table support enabled. Disabling that mode returned only 1,000 rows for a
10,001-row synthetic test. It is not a full-result workaround. No completed
Spark-to-Windows benchmark timing exists; notebook-driver measurements are
separate. Cleanup receipts recorded F64/Paused. The precise driver-versus-service
root cause remains unresolved.

**NEE configured on, but native plan markers absent.** String-marker checks are
heuristics, not proof of JVM fallback. The historical Warehouse-Delta lane was
slower, but layout alone was not isolated as the cause. Use execution diagnostics
and controlled layout tests before making causal claims.

**Headless `RunNotebook` job: no cell output.** → Write results to a **lakehouse Delta table**; read back via
the **OneLake DFS API** (the SQL endpoint lags with `PathNotFound`). Writing 86 rows → many tiny parquet files.

**`RunNotebook` fails `Not supported language`.** → Notebook metadata must include
`language_info.name = python` (+ kernelspec `synapse_pyspark`, `dependencies.lakehouse`).

**Capacity scale (F→F) kills a running job** (`System_Cancelled_Session_Statements_Failed`). A scale =
suspend+resume in the observed incident. **Never scale while a job runs.** Preserve
the interrupted run; do not resume merely because a query ID already has a median.
Repaired runs use unique evidence directories and pinned query/protocol identity.

**Livy/token flakiness on long runs.** Historical harnesses used token refresh
and polling recovery. Do not generalize those retries to benchmark SQL or
ambiguous resource mutations; the latest client campaign did not automatically
retry benchmark queries.

## Dataset generator (Fabric DuckDB)

**`%pip install` unreliable in headless jobs.** → Install via
`subprocess.check_call([sys.executable,"-m","pip","install","duckdb"])`.

**`/lakehouse/default/Files` FUSE path doesn't mount in headless jobs** → `os.makedirs`/`COPY` there fail.
→ Generate to local `/tmp`, then `notebookutils.fs.cp("file:"+path, "abfss://…/<lh>/Files/…")`.

**Schema parity:** DuckDB renames `s_tax_precentage`→`s_tax_percentage` (harmless) and
`c_last_review_date`→`c_last_review_date_sk` (**breaks query 30**). → Post-generation `ALTER…RENAME COLUMN`
back to the classic names (see [04-dataset-generation.md](04-dataset-generation.md)).

## Snowflake

**All queries fail `invalid identifier`.** `INFER_SCHEMA` created lowercase, case-sensitive columns.
→ Rename all columns to **UPPERCASE** (metadata-only).

**`Numeric value 'ZOUROS' not recognized` (q66/q80/q84).** T-SQL `+` string concat is **numeric** in Snowflake.
→ Convert `+` → `||` in the dialect converter. **EXPLAIN validates compile but not runtime coercion — execute-test.**

**Storage-integration consent blocked.** Needs a directory-admin role to admin-consent the Snowflake app.
→ Ask an authorized directory administrator to approve the required consent, then
grant the service principal the required Fabric workspace access.

**`COUNT` ≠ ground truth after load.** A wildcard Parquet COPY can include files
outside the active Delta snapshot. Refreshing the external table alone does not
repair an already-loaded native table. Compare the active source snapshot and
loaded data, then plan a corrected load; row-count agreement alone is insufficient.

## Local tooling (Windows)

- Inline `python -c` with regex/commas breaks in PowerShell → use **script files**.
- `Out-File` defaults to UTF-16 → prefer explicit UTF-8 when writing files other tools read.
- On Windows-on-ARM, build venvs from **x64 Python** (ARM64 wheels for pyarrow/cryptography often fail).
