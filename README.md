# Fabric, Databricks and Snowflake: Exploratory Analytics POC

Results and documentation for a TPC-DS-inspired analytical workload at nominal
SF3000, approximately 3 TB, including multi-billion-row fact tables in Azure
West US 2. This is not an official TPC-DS benchmark or a definitive platform
ranking.

## Latest Results

Recorded October 11-13, 2026. The latest experiment used zero warmups, one
measured execution per query, and full result consumption without output caps.
Each completed lane measured 85 distinct queries and consumed 86 statement
outputs. Query 39 contains two statements.

| Lane | Batch execution time (min) | Total measured query time (min) | Geometric mean/query (s) |
|---|---:|---:|---:|
| Fabric Warehouse | **23.3462** | 23.3371 | 5.0223 |
| Fabric Lakehouse SQL endpoint | **30.9591** | 30.9419 | 6.8782 |
| Snowflake optimized | **23.9822** | 23.9743 | 7.1424 |
| Snowflake baseline | **29.3691** | 29.3611 | 11.3612 |
| Azure Databricks Photon | **47.3428** | 47.3256 | 12.6435 |
| Azure Databricks non-Photon | **84.8936** | 84.8760 | 22.5370 |
| Fabric Spark SQL (notebook driver) | **35.2560** | 34.9754 | 15.6807 |

The first six lanes consume every result row on the same Windows host. Spark's
completed measurement consumes results on the Fabric notebook driver, not on
Windows; its Windows ODBC full-result path remains blocked. Spark is listed
separately, not ranked as a Windows-client result. Other rows are ordered by
ascending geometric mean per query, not batch time.

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

See [detailed results](docs/RESULTS.md) for runtime/client versions, compute
configuration, dated hourly price models, row-count differences and limitations.

## Documentation

- [Architecture and data access](docs/02-architecture.md)
- [Recorded environment setup](docs/03-environment-setup.md)
- [Dataset generation and recorded table counts](docs/04-dataset-generation.md)
- [Methodology and historical protocols](docs/07-methodology.md)
- [Observed troubleshooting](docs/09-troubleshooting.md)
- [Legacy hosted runbook](docs/RUNBOOK.md)
- [Latest results](docs/RESULTS.md)

## Release Scope

This is a results-and-documentation-only release. It does not include SQL query
files, notebooks, harnesses, tests, raw run evidence, query manifests, logs,
resource/run IDs, real deployment endpoints, credentials or the dataset.
File paths and commands in the historical setup/runbook describe the private
benchmark kit; that runnable kit is not distributed here. This repository
cannot independently reproduce or reaggregate the private measurements.

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