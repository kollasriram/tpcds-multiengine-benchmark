# Latest benchmark results

**Databricks query runtime:** Databricks Runtime 16.4, Scala 2.12
(`16.4.x-scala2.12`), used for both Photon and Standard publication lanes.

The completed full-result measurements below were recorded October 11-13,
2026 UTC at nominal SF3000. They use the same frozen **85-query cohort**, excluding
Q57, with **zero warmups and one measured execution**. Query 39 has two statements,
so a complete pass consumes **86 statement outputs**. Six lanes consume results
on Windows; Fabric Spark SQL consumes results on the Fabric notebook driver,
not the Windows client. Results are fully consumed without output caps or
automatic benchmark-query retries. Run IDs and
infrastructure identities are omitted from this public summary.

## Consolidated full-result measurements

Windows-client lanes are ordered by ascending geometric mean per query. The Spark
notebook row is listed last because its execution/consumption boundary differs;
it must not be ranked as a Windows-client measurement.

| Lane / runtime and client | Statement outputs | Batch execution time (min) | Total measured query time (min) | Geometric mean/query (s) | Cost / Power | Compute configuration | Status |
|---|---:|---:|---:|---:|---|---|---|
| Fabric Warehouse<br>Managed T-SQL; build not captured<br>SQL ODBC 18.06.0002 | 86 | **23.3462** | 23.3371 | 5.0223 | $11.52/h model<br>64 CU | F64 shared capacity<br>Warehouse native tables | Completed |
| Fabric Lakehouse SQL endpoint<br>Managed T-SQL; build not captured<br>SQL ODBC 18.06.0002 | 86 | **30.9591** | 30.9419 | 6.8782 | $11.52/h model<br>64 CU | F64 shared capacity<br>Lakehouse SQL analytics endpoint | Completed |
| Snowflake optimized<br>Managed SQL; build not captured<br>Python connector 4.7.5 | 86 | **23.9822** | 23.9743 | 7.1424 | $15/h base model + QAS<br>5 credits/h base | Medium STANDARD Gen2<br>1 cluster; QAS max 8<br>Experimental optimized layout | Completed |
| Snowflake baseline<br>Managed SQL; build not captured<br>Python connector 4.7.5 | 86 | **29.3691** | 29.3611 | 11.3612 | $15/h base model + QAS<br>5 credits/h base | Medium STANDARD Gen2<br>1 cluster; QAS max 8<br>Baseline native layout | Completed |
| Azure Databricks Photon<br>DBR 16.4; Scala 2.12<br>ODBC 2.12.1.1001 | 86 | **47.3428** | 47.3256 | 12.6435 | $13.06/h model<br>40 vCPU; 160 GiB RAM | Standard_D8ads_v5<br>4 workers + 1 driver<br>On-demand all-purpose; Photon<br>Classic VNet-injected compute | Completed |
| Azure Databricks non-Photon<br>DBR 16.4; Scala 2.12<br>ODBC 2.12.1.1001 | 86 | **84.8936** | 84.8760 | 22.5370 | $12.096/h model<br>64 vCPU; 256 GiB RAM | Standard_D8ads_v5<br>7 workers + 1 driver<br>On-demand all-purpose; Standard<br>Classic VNet-injected compute | Completed |
| Fabric Spark SQL (notebook driver)<br>Fabric Runtime 2.0<br>Spark 4.1.1; Python 3.13.11 | 86 | **35.2560** | 34.9754 | 15.6807 | $11.52/h model<br>64 CU; executor allocation not captured | F64 shared capacity<br>Fabric notebook; full driver consumption | Notebook completed; Windows ODBC blocked |

Runtime/client versions are recorded values, not assumed managed-engine builds.
Spark's recorded build is `4.1.1.5.5.20260930.238388267`; Databricks' configured
runtime family is `16.4.x-scala2.12`, not an immutable patch ID.

**Cost / Power:** dated October 7, 2026 USD/hour list-price models for West US 2,
not actual run bills or current quotes. Fabric is PAYG; Databricks models include
VMs and Premium all-purpose DBUs, including the driver; Snowflake assumes Enterprise Gen2 pricing and
excludes variable QAS charges. Storage, disks, networking, discounts and other
charges are excluded. Fabric smoothing/settlement and complete lifecycle billing
were not measured. Do not multiply benchmark timers by these rates to report
actual workload cost. Power means compute capacity here, not watts or energy;
CU, credits and vCPU are not interchangeable measures. Snowflake physical cores
and Fabric Spark executor allocation were not captured.

Spark's available timing statistics cover 85 measured queries and 86 statement
outputs, with zero
warmups and one measurement, fully consumed on the notebook driver. No
completed Spark-to-Windows timing exists; the ODBC full-result gate remains
blocked. These notebook values are not Windows full-fetch or Spark noop results.

All six completed Windows lanes consumed 85 queries and 86 outputs. Fabric Warehouse
and Lakehouse SQL each consumed 60,959,601 rows; Databricks Photon and non-Photon
each consumed 61,035,697; both Snowflake layouts consumed 61,035,727. These
cross-engine count differences remain unresolved and are not evidence of equal
result values.

**Batch execution time is the primary metric, displayed in minutes:** first SQL submission through
final full consumption at the recorded client/driver location, including
inter-query evidence overhead. Spark excludes transfer to Windows.
**Total measured query time** is the sum of the 85 individual SQL execution and
full-fetch timers, displayed in minutes. Startup, authentication, metadata checks and cleanup are outside
both intervals. Neither interval is total billed compute time.

Fabric Warehouse, Lakehouse SQL endpoint and Databricks use ODBC; Snowflake uses its native Python
connector. All result rows are fetched, counted and discarded. Client placement
and consumption are aligned, not transports, hardware or guaranteed cache state.
Photon and non-Photon refer to Databricks, not Snowflake.

## Configuration and limits

- Fabric Warehouse: F64, native tables, result caching disabled.
- Fabric Lakehouse SQL endpoint: F64, per-query result-cache bypass, SQL Server
	ODBC full fetch. Completed October 13; capacity pause confirmed afterward.
- Fabric Spark SQL: F64, Fabric notebook execution with full driver consumption.
  Its available 85-query timing is separate from the blocked Windows ODBC path.
- Databricks Photon client: DBR 16.4, four Standard_D8ads_v5 workers plus a
	matching driver, fresh all-purpose compute, Unity Catalog Delta tables.
- Databricks non-Photon: DBR 16.4, seven Standard_D8ads_v5 workers plus a matching
	driver, fresh all-purpose compute; this differs from Photon's worker count.
	Completed October 13; owned-cluster termination confirmed afterward.
- Snowflake: Medium STANDARD Gen2, one cluster, QAS enabled with maximum scale
	factor 8, session result caching disabled. Baseline and experimental optimized
	layouts are reported separately.

Single passes do not establish repeatability, confidence intervals or an
equivalent-compute ranking. Cross-engine result-value equivalence and historical
row-count differences remain unresolved. Earlier capped publication statistics
are not included.

## Scope

Raw run evidence, campaign plans and qualification receipts are kept outside the
public export. This page contains reported summary numbers, not a public archive
for independently reaggregating the private client runs. The private hosted
and notebook lab uses a separate legacy 86-query protocol; running it unchanged
does not reproduce the 85-query unlimited-output Windows experiment.
That lab, its query files and its runner code are not distributed in this
results/documentation-only repository.
