# 04 — Dataset Generation

**Databricks query runtime:** Databricks Runtime 16.4, Scala 2.12
(`16.4.x-scala2.12`), used for both Photon and Standard publication lanes.

You need 24 TPC-DS tables at **Scale Factor 3000 (≈ 3 TB)** as Parquet, then right-sized to ~1 GB
**Delta** files (`tpcds_3tb_opt`). Two generation options; pick one.
These are recorded generation notes, not a bundled generator or notebook lab.
Use the same dataset snapshot and layout label for every engine included in
a comparison. Neither the dataset nor executable notebooks are distributed here.

## Benchmark Table Record Counts

These are the **recorded counts for the nominal SF3000 benchmark dataset**, not
SF1 smoke-test data. The completed campaign recorded
matching source and native counts with `OK` status for all 24 tables.

| Table | Records |
|---|---:|
| `call_center` | 48 |
| `catalog_page` | 36,000 |
| `catalog_returns` | 432,006,840 |
| `catalog_sales` | 4,320,004,419 |
| `customer` | 30,000,000 |
| `customer_address` | 15,000,000 |
| `customer_demographics` | 1,920,800 |
| `date_dim` | 73,049 |
| `household_demographics` | 7,200 |
| `income_band` | 20 |
| `inventory` | 1,033,560,000 |
| `item` | 360,000 |
| `promotion` | 1,800 |
| `reason` | 67 |
| `ship_mode` | 20 |
| `store` | 1,350 |
| `store_returns` | 864,006,076 |
| `store_sales` | 9,177,187,206 |
| `time_dim` | 86,400 |
| `warehouse` | 22 |
| `web_page` | 3,600 |
| `web_returns` | 215,999,442 |
| `web_sales` | 2,159,930,761 |
| `web_site` | 66 |
| **Total (24 tables)** | **18,250,185,186** |

This is saved load evidence, not a fresh recount of every engine or proof of
row-value equivalence. The source log records times of day without a date or
timezone. Counts describe this actual dataset; nominal SF3000 alone is not a
guarantee of identical cardinalities in a newly generated dataset.

---

## Option A — Databricks (BlueGranite) — *what this benchmark used*

[BlueGranite/tpc-ds-dataset-generator](https://github.com/BlueGranite/tpc-ds-dataset-generator) —
two Databricks notebooks wrapping [databricks/spark-sql-perf](https://github.com/databricks/spark-sql-perf)
+ [databricks/tpcds-kit](https://github.com/databricks/tpcds-kit) `dsdgen`. Generates **parquet** at any
scale 1 GB → 100 TB.

1. `TPC-DS-Configure` — cluster init script builds `dsdgen` on every node + installs the `spark-sql-perf`
   JAR (vendored in the repo's `lib/`), mounts ADLS.
2. `TPC-DS-GenerateData` — set `scaleFactor = "3000"`, `fileFormat = "parquet"`, run
   `TPCDSTables(...).genData(location=…, format="parquet", numPartitions=…)`.

The upstream `databricks/spark-sql-perf` URL returned 404 during the documentation
check. The recorded BlueGranite route used vendored JARs targeting older Spark
and Scala versions. This does not establish compatibility with today's runtimes;
validate the generator and dependencies on a small dataset before SF3000 use.

## Option B — Standalone Spark (maropu) — *unqualified alternative*

[maropu/spark-tpcds-datagen](https://github.com/maropu/spark-tpcds-datagen) documents
Spark 3.1.1 / Scala 2.12 and bundled tpcds-kit. The latest repository commit shown
during review was from 2021; active maintenance and compatibility with the POC's
current runtimes were not established. This is an alternative, not the recorded
generation path:
```bash
./bin/dsdgen --output-location abfss://example-container@<acct>.dfs.core.windows.net/tpcds_3tb \
             --scale-factor 3000 --format parquet --num-partitions <N>
```

---

## Right-size parquet to Delta

The POC's raw `dsdgen` Parquet included oversized files (for example, a 96 GB
`store_sales` file); the recorded fix was a repartitioned rewrite, not a claim
about every runtime's `OPTIMIZE` capabilities.
The rewrite reads each table, repartitions to `ceil(size / 1 GB)`, and writes **~1 GB Delta** files
(optimizeWrite off; `repartitionByRange(date_sk)` + `sortWithinPartitions(date_sk, item_sk)`), to a **new**
folder `tpcds_3tb_opt` — source untouched, engine-agnostic output. Process smallest→largest (self-validates
early), resumable.

Result: `store_sales` 358 files @1.37 GB, `catalog_sales` 296 @1.21 GB, `web_sales` 151 @1.15 GB,
returns ~0.9 GB, inventory 369 MB, tiny dims 1 file each.
