# Fabric, Databricks and Snowflake: Exploratory Analytics POC

Results and documentation for a TPC-DS-inspired analytical workload at nominal
SF3000, approximately 3 TB, including multi-billion-row fact tables in Azure
West US 2. This is not an official TPC-DS benchmark or a definitive platform
ranking.

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

## Consolidated Full-Result Measurements

Windows-client lanes are ordered by ascending geometric mean per query. The Spark
notebook row is listed last because its execution/consumption boundary differs;
it must not be ranked as a Windows-client measurement.

| Lane / runtime and client | Statement outputs | Rows consumed on Windows (approx.) | Batch execution time (min) | Total measured query time (min) | Geometric mean/query (s) | Cost / Power | Compute configuration | Status |
|---|---:|---:|---:|---:|---:|---|---|---|
| Fabric Warehouse<br>Managed T-SQL; build not captured<br>SQL ODBC 18.06.0002 | 86 | ~61.0 million | **23.3462** | 23.3371 | 5.0223 | $11.52/h model<br>64 CU | F64 shared capacity<br>Warehouse native tables | Completed |
| Fabric Lakehouse SQL endpoint<br>Managed T-SQL; build not captured<br>SQL ODBC 18.06.0002 | 86 | ~61.0 million | **30.9591** | 30.9419 | 6.8782 | $11.52/h model<br>64 CU | F64 shared capacity<br>Lakehouse SQL analytics endpoint | Completed |
| Snowflake optimized<br>Managed SQL; build not captured<br>Python connector 4.7.5 | 86 | ~61.0 million | **23.9822** | 23.9743 | 7.1424 | $15/h base model + QAS<br>5 credits/h base | Medium STANDARD Gen2<br>1 cluster; QAS max 8<br>Experimental optimized layout | Completed |
| Snowflake baseline<br>Managed SQL; build not captured<br>Python connector 4.7.5 | 86 | ~61.0 million | **29.3691** | 29.3611 | 11.3612 | $15/h base model + QAS<br>5 credits/h base | Medium STANDARD Gen2<br>1 cluster; QAS max 8<br>Baseline native layout | Completed |
| Azure Databricks Photon<br>DBR 16.4; Scala 2.12<br>ODBC 2.12.1.1001 | 86 | ~61.0 million | **47.3428** | 47.3256 | 12.6435 | $13.06/h model<br>40 vCPU; 160 GiB RAM | Standard_D8ads_v5<br>4 workers + 1 driver<br>On-demand all-purpose; Photon<br>Classic VNet-injected compute | Completed |
| Azure Databricks non-Photon<br>DBR 16.4; Scala 2.12<br>ODBC 2.12.1.1001 | 86 | ~61.0 million | **84.8936** | 84.8760 | 22.5370 | $12.096/h model<br>64 vCPU; 256 GiB RAM | Standard_D8ads_v5<br>7 workers + 1 driver<br>On-demand all-purpose; Standard<br>Classic VNet-injected compute | Completed |
| Fabric Spark SQL (notebook driver)<br>Fabric Runtime 2.0<br>Spark 4.1.1; Python 3.13.11 | 86 | N/A (notebook driver) | **35.2560** | 34.9754 | 15.6807 | $11.52/h model<br>64 CU; executor allocation not captured | F64 shared capacity<br>Fabric notebook; full driver consumption | Notebook completed; Windows ODBC blocked |

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
outputs, with zero warmups and one measurement, fully consumed on the notebook
driver. No completed Spark-to-Windows timing exists; the ODBC full-result gate
remains blocked. These notebook values are not Windows full-fetch or Spark noop results.

All six completed Windows lanes consumed 85 queries and 86 outputs, totaling
approximately 61.0 million rows per lane on the Windows client. These rounded
totals conceal small cross-engine count differences, which remain unresolved
and do not establish equal result values.

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

## Configuration and Limits

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

Warehouse had the lowest observed batch time and geometric mean among the
latest Windows-client lanes. A single pass does not establish a repeatable
winner. Earlier repeated runs used different protocols and must not be treated
as repetitions of this experiment. Full cross-platform result-value equivalence
and actual billed workload cost remain unverified.

F64's nominal hourly price was a configuration-selection reference, not an
equal-cost or identical-hardware constraint. The configurations have different
price models, acceleration, caching, transports and consumption boundaries.
Startup, authentication, metadata checks and cleanup are outside the displayed
timers. Baseline and experimental optimized Snowflake layouts are reported
separately; this is not a uniformly untuned comparison.

## Documentation

- [Architecture and data access](https://github.com/kollasriram/tpcds-multiengine-benchmark/blob/main/docs/02-architecture.md)
- [Recorded environment setup](https://github.com/kollasriram/tpcds-multiengine-benchmark/blob/main/docs/03-environment-setup.md)
- [Dataset generation and recorded table counts](https://github.com/kollasriram/tpcds-multiengine-benchmark/blob/main/docs/04-dataset-generation.md)
- [Methodology and historical protocols](https://github.com/kollasriram/tpcds-multiengine-benchmark/blob/main/docs/07-methodology.md)
- [Observed troubleshooting](https://github.com/kollasriram/tpcds-multiengine-benchmark/blob/main/docs/09-troubleshooting.md)
- [Legacy hosted runbook](https://github.com/kollasriram/tpcds-multiengine-benchmark/blob/main/docs/RUNBOOK.md)
- [Latest results](https://github.com/kollasriram/tpcds-multiengine-benchmark/blob/main/docs/RESULTS.md)

## Release Scope

This is a results-and-documentation-only release. It does not include SQL query
files, notebooks, harnesses, tests, raw run evidence, query manifests, logs,
resource/run IDs, real deployment endpoints, credentials or the dataset.
File paths and commands in the historical setup/runbook describe the private
benchmark kit; that runnable kit is not distributed here. This repository
cannot independently reproduce or reaggregate the private measurements.
The private hosted/notebook lab uses a separate legacy one-warmup/two-measurement
protocol; running it unchanged does not reproduce the latest zero-warmup results.

## Licensing and Attribution

The workload used TPC-DS-derived queries adapted for Microsoft Fabric,
Azure Databricks and Snowflake. [Agirish/tpcds](https://github.com/Agirish/tpcds)
is credited as the Spark SQL source reference; adaptations included
dialect-specific syntax and numeric-semantics fixes. No upstream affiliation
or endorsement is claimed. The source reference is attribution, not a grant
of redistribution rights.

When checked on October 14, 2026, GitHub reported no license for Agirish/tpcds,
and its complete file tree contained no license, notice or README file. This
does not determine all rights in the underlying TPC-DS material. The
[TPC toolkit EULA](https://github.com/databricks/tpcds-kit/blob/master/EULA.txt)
also has redistribution conditions. Query files and notebooks containing query
text are deliberately excluded from this release. Modifications and attribution
alone do not resolve redistribution permissions.

No repository-wide open-source license is granted for this documentation.
TPC-DS is a trademark of the Transaction Processing Performance Council. The
benchmark data is synthetic.