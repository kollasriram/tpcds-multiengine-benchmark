# 03 — Environment Setup (all systems)

**Databricks query runtime:** Databricks Runtime 16.4, Scala 2.12
(`16.4.x-scala2.12`), used for both Photon and Standard publication lanes.

Detailed, system-by-system setup. Everything is **West US 2**. Replace all IDs with your own.
Complete the applicable platform section and
[dataset preparation](04-dataset-generation.md) before running your own workload.

This guide records the POC setup, not universal service defaults. The checked-in
notebooks in the private benchmark kit and the hosted runbook use the legacy protocol; see
[latest results](RESULTS.md) for the separate Windows-client measurements.
This documentation-only release does not distribute notebooks, requirements
files or harnesses. Historical file paths below refer to the private kit.

---

## Local desktop notebook environment

The private lab's Fabric Warehouse, Lakehouse SQL endpoint and Snowflake
notebooks executed in a local Python kernel, with SQL drivers and native
connector dependencies. The Fabric Spark and Databricks notebooks used their
VS Code extensions to attach remote platform kernels. They must not run on
local Apache Spark because that would change the engine being measured.
The notebooks and their dependency files are not included in this release.

---

## A. Azure Storage (ADLS Gen2) — the shared dataset

- **Account:** `examplestorage001`, **HNS enabled**, container `stage`.
- **Governed posture (enforced by managed-env policy):** `publicNetworkAccess=Disabled`,
  `allowSharedKeyAccess=false`, default action **Deny**, bypass `AzureServices`.
- **Dataset:** `example-container/tpcds_3tb_opt` — 24 TPC-DS tables as Delta, right-sized to ~1 GB files
  (`store_sales` 358 files @1.37 GB, `catalog_sales` 296 @1.21 GB, `web_sales` 151 @1.15 GB; tiny dims = 1 file).

### RBAC / access to grant
- **Fabric** workspace identity → `Storage Blob Data Reader` (+ `Storage Blob Delegator`) on the account.
- **Databricks** service principal `example-storage-principal` → `Storage Blob Data Contributor`.

### Fabric trusted-workspace access (works with pna=Disabled)
Add a **resource-instance rule** so the Fabric workspace can read the firewalled account **without**
flipping `pna`:
```powershell
$resourceId = "/subscriptions/00000000-0000-0000-0000-000000000000/resourcegroups/Fabric/providers/Microsoft.Fabric/workspaces/<WORKSPACE_ID>"
$tenantId = "<TENANT_ID>"
$resourceGroupName = "<STORAGE_ACCOUNT_RESOURCE_GROUP>"
$accountName = "<STORAGE_ACCOUNT_NAME>"
Add-AzStorageAccountNetworkRule -ResourceGroupName $resourceGroupName -Name $accountName -TenantId $tenantId -ResourceId $resourceId
```
Keep the zero subscription GUID and literal `Fabric` resource group in the workspace
resource ID. Replace the other placeholders and run in an authenticated Azure
PowerShell session with permission to update the storage account's network rules.
See [trusted workspace access](https://learn.microsoft.com/en-us/fabric/security/security-trusted-workspace-access).
This is the shortcut-access path used by this campaign; managed private endpoints
for direct Spark access are a separate configuration.

---

## B. Microsoft Fabric

### Capacity & workspace
- Capacity `example_capacity`; **F64** was used for measured runs. The $11.52/hr
  PAYG figure is the October 7, 2026 nominal model, not an invoice or current quote.
  Pause after authorized use and verify cleanup; the October 13 pass recorded
  F64/Paused. A scale operation interrupted a POC Spark session, so do not resize
  during execution or assume that resizing is the same as cleanup.
- Workspace `example_workspace` on `example_capacity`. Provision a **workspace identity** (gets the RBAC above).

### Runtime 2.0 + NEE
The POC selected Runtime 2.0 explicitly. Inspect your workspace's current default
and supported runtime before reuse; do not assume a new workspace uses RT 1.3:
```http
PATCH https://api.fabric.microsoft.com/v1/workspaces/<ws>/spark/settings
{ "environment": { "runtimeVersion": "2.0" } }
```
The POC requested Native Execution Engine per-session with the settings below.
Configured enablement does not prove native execution for every query:
```python
spark.conf.set("spark.native.enabled", "true")
spark.conf.set("spark.sql.ansi.enabled", "false")   # Spark 4.x defaults ANSI=TRUE; off for parity
```

### Lakehouses (shortcuts, zero-copy)
- `example_lakehouse` — 24 OneLake **AdlsGen2 shortcuts** → `tpcds_3tb_opt/<table>` (this is the Spark-ADLS lane
  source **and** the writable results lakehouse).
- `example_warehouse_shortcuts` — 24 shortcuts → `example_warehouse` Tables/dbo (warehouse V-Ordered Delta) for the WH-shortcut lane.

### Warehouse (native load)
- The executed load used **CTAS / chunked `INSERT…SELECT`** from `example_lakehouse.dbo.<t>`
  into `example_warehouse`. Pipeline Copy was an alternative, not this run's loader.
  See the [load architecture and timings](02-architecture.md#fabric-access-and-warehouse-loading).
- Big facts: `store_sales` 9,177,187,206 · `catalog_sales` 4,320,004,419 · `web_sales` 2,159,930,761.
- `COUNT(*)` overflows int on >2.1 B-row tables → use `COUNT_BIG(*)`.
- **Disable result-set caching for fairness:** `ALTER DATABASE example_warehouse SET RESULT_SET_CACHING OFF`.

### Connecting with pyodbc (for the T-SQL harness)
ODBC Driver 18 + Entra token (resource `https://database.windows.net/`, attr `SQL_COPT_SS_ACCESS_TOKEN=1256`).
Endpoint host: `<id>.datawarehouse.fabric.microsoft.com` (lakehouse + warehouse share it). See `harnesses/_fabsql.py`.

---

## C. Azure Databricks

- Workspace `example-databricks` (VNet-injected; private endpoints to the ADLS dfs + blob).
- The original direct-read lane used SP OAuth. **Unity Catalog also worked** on
  classic VNet compute with the Access Connector and dedicated storage credential.
  Earlier failures were setup/network-path issues, not a general UC limitation.
  Do not drop working external locations to reproduce the direct-read experiment.
- **Service-principal OAuth** on the cluster (direct `abfss`, auth over the private endpoint):
  ```
  fs.azure.account.auth.type.<acct>.dfs.core.windows.net OAuth
  fs.azure.account.oauth.provider.type.<acct>...          org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider
  fs.azure.account.oauth2.client.id.<acct>...             <appId of example-storage-principal>
  fs.azure.account.oauth2.client.secret.<acct>...         {{secrets/tpcds/sp_secret}}
  fs.azure.account.oauth2.client.endpoint.<acct>...       https://login.microsoftonline.com/<tenant>/oauth2/token
  ```
  Store the SP secret in a Databricks secret scope (`tpcds/sp_secret`); use `SINGLE_USER` access mode.
- **Clusters:** historical reruns reused existing all-purpose clusters; the latest
  Windows-client runs created fresh on-demand all-purpose clusters in the
  VNet-injected workspace and terminated them afterward. `Standard_D8ads_v5` for driver
  and workers, DBR `16.4.x-scala2.12`; each VM has 8 vCPUs and 32 GiB RAM.
  - Photon: 1 driver + 4 workers (5-node); 40 total vCPUs, 160 GiB RAM;
    4 DBU/node-hour, nominal Premium all-purpose model $13.06/hr.
  - non-Photon: 1 driver + 7 workers (8-node); nominal Premium all-purpose model $12.096/hr.
    Total 64 vCPUs and 256 GiB RAM, at 2 DBU/node-hour.
  - Existing-cluster runs are all-purpose billing, not discounted job-cluster pricing.

---

## D. Snowflake

- Account in **Azure West US 2**; confirm region before reuse. Shared regional
  placement does not prove zero network or storage-request charges. The POC used
  broad `ACCOUNTADMIN` access, not a production least-privilege recommendation.
- DB/schema `TPCDS.PUBLIC`. Benchmark warehouse `WH_TPCDS_BENCH` = **MEDIUM**,
  `AUTO_SUSPEND=60`, single-cluster, `MAX_CLUSTER_COUNT=1`.
- October 7 metadata recorded Enterprise, `STANDARD_GEN_2`, and QAS enabled with
  maximum scale factor 8. Historical settings and actual QAS usage require separate
  evidence; MEDIUM alone does not establish cost.
- Latest runs measured both baseline native tables and a separate experimental
  optimized layout on Medium STANDARD Gen2, with one cluster and QAS max 8.
- **Fairness:** `ALTER SESSION SET USE_CACHED_RESULT=FALSE`; recycle (suspend+resume) between cycles.

### Native load from OneLake (the path that worked under governance)

COPY INTO creates Snowflake-native storage. This is not zero-copy or a Delta-native
execution lane. Verify the current platform's external-Delta support before reuse.
1. Create a **storage integration** to OneLake. The recorded allowlist was
  workspace-wide: `azure://onelake.blob.fabric.microsoft.com/<WORKSPACE_ID>/`.
  Review a narrower scope before reuse; the stage itself targeted the lakehouse.
2. `DESCRIBE INTEGRATION` → grab the `AZURE_CONSENT_URL` + app name; **admin-consent** the Snowflake
   enterprise app in Entra (needs Global Admin / Cloud App Admin). Grant that SP **Contributor** on the
   Fabric workspace.
3. External stage `onelake_lh` (STORAGE_INTEGRATION + the OneLake URL), file format `pq = (TYPE=PARQUET)`.
4. Per table: `CREATE EXTERNAL TABLE … TABLE_FORMAT=DELTA` → `REFRESH` → verify `COUNT(*)` = ground truth;
   then `INFER_SCHEMA` + `CREATE TABLE … USING TEMPLATE` + `COPY INTO … MATCH_BY_COLUMN_NAME=CASE_INSENSITIVE`.
5. **Fix column case:** `INFER_SCHEMA` makes lowercase, case-sensitive columns → all queries fail
   `invalid identifier`. Rename all columns to UPPERCASE (metadata-only).

Benchmark execution is described in the [end-to-end runbook](RUNBOOK.md).
Use separate Snowflake load and benchmark warehouses, validate native row counts,
and repair inferred column case before running the cohort.

---

## Pricing and Capacity

Configuration prices are nominal models, not observed bills or
identical-compute comparisons. The documented models were checked October 7,
2026. Record current regional prices with each rerun.
