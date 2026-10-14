# 07 — Methodology

**Databricks query runtime:** Databricks Runtime 16.4, Scala 2.12
(`16.4.x-scala2.12`), used for both Photon and Standard publication lanes.

## Latest Full-Result Measurement

The October 11-13, 2026 [latest results](RESULTS.md) use zero warmups, one
measured execution, one pass and no automatic benchmark-query retries. Results
are consumed fully without output caps. Fabric SQL and Databricks use ODBC on
the same Windows host; Snowflake uses its native Python connector there.
Fabric Spark consumes results on its notebook driver, not on Windows.

The primary metric is batch elapsed time from first SQL submission through
final full consumption, including inter-query overhead. Total query time is
the sum of individual execution/consumption timers. Both are displayed in
minutes and exclude setup, authentication, metadata checks and cleanup.
Geometric means summarize query timings; they are not repeated-query medians
or billed durations. Neither zero warmups nor cache bypass guarantees cold data.

Latest Databricks client runs used fresh all-purpose VNet-injected clusters,
not the legacy runner's job clusters. Spark's Windows ODBC path failed full-result
qualification; no completed client timing is reported for it. The historical
measurement rules below do not describe these latest results, and the public
lab does not reproduce that private client campaign unchanged.

## Query set

Query paths below describe the private benchmark kit, not files distributed in
this results/documentation-only release. No query files or notebook bundles
are included here.

- **Source:** TPC-DS-derived queries. Repaired Spark lanes use a pinned local snapshot of the [Agirish/tpcds](https://github.com/Agirish/tpcds)
  Spark-SQL port (`queries/spark_sql/`); the T-SQL (`queries/tsql_warehouse/`) and Snowflake
  (`queries/snowflake/`) sets are dialect translations of the retained query IDs.
  Shared IDs and inspected constants do not prove all-query semantic equivalence.
  The Lakehouse SQL endpoint runs `queries/tsql_sqlendpoint/`: the Warehouse T-SQL with
  `OPTION (USE HINT ('DISABLE_RESULT_SET_CACHE'))` added before each statement terminator, because that
  endpoint keeps result-set caching on at item level.
- **Same substitution parameters.** Verified: e.g. `query1.sql` is byte-identical across Spark-SQL and T-SQL
  except the final clause (`LIMIT 100` vs `OFFSET 0 ROWS FETCH NEXT 100 ROWS ONLY`) — same `d_year=2001`,
  `s_state='TN'`, `AVG(...)*1.2`, top-100.
- **86 kept / 13 excluded.** Excluded (blank/failed on ≥1 engine in a prior run): `4, 5, 11, 14, 23, 24, 50,
  64, 67, 72, 75, 78, 93`. See `queries/qids.csv`.

### Source Attribution and Redistribution

The workload uses TPC-DS-derived queries adapted for Microsoft Fabric,
Azure Databricks, and Snowflake. [Agirish/tpcds](https://github.com/Agirish/tpcds)
is credited as the Spark SQL source reference. Local adaptations include
dialect-specific syntax and numeric-semantics fixes; they do not establish
full cross-platform result-value equivalence or imply upstream endorsement.

No explicit license was found in the Agirish repository when checked on
October 14, 2026; GitHub also reported no license. Attribution and modifications
are not redistribution permission, and underlying TPC-DS terms may also apply.
The results/documentation-only release excludes bundled query text, notebooks,
and runner code. See [licensing and attribution](../README.md#licensing-and-attribution)
for the release scope and outstanding permissions.

## Dialect Translation and Validation Limits

| From Spark-SQL / T-SQL | To | Lanes |
|---|---|---|
| `OFFSET n ROWS FETCH NEXT m` ↔ `LIMIT m` | both directions | WH uses OFFSET/FETCH; Spark & Snowflake use LIMIT |
| `[bracket]` identifiers ↔ `"double quote"` | quoting | T-SQL brackets; Snowflake double-quotes |
| `COUNT(*)` → `COUNT_BIG(*)` | overflow fix | Warehouse only (>2.1 B rows) |
| `STDEV`/`STDEVP` → `STDDEV`/`STDDEV_POP` | function name | Snowflake |
| **T-SQL `+` string concat → `||`** | concat operator | Snowflake (critical: `+` is numeric → runtime error on q66/q80/q84) |
| `GROUPING()`/alias-in-ORDER-BY fixes | T-SQL grammar | Warehouse q2/q22/q36/q70/q86 |

Matching query text and constants does not establish matching semantics. Integer
division, AVG return types, decimal precision, null behavior and nondeterministic
ties need explicit validation. Warehouse q34/q73 now cast the ratio numerator to
DECIMAL(18,6). Integer quantity/count AVG expressions in q7/q13/q17/q22/q26/q27/
q29/q35/q39/q85 now have fractional inputs. Remote schemas and results must still
be compared. q39 has two statements, and STDDEV is sample standard deviation.

## Historical and Legacy Lab Measurement

- **Historical protocol:** SQL clients and recorded Databricks runs used one warmup
  and two measured executions; Fabric Spark used one measured execution. The
  Databricks/shared and UC code selects the upper middle value for even samples,
  which is not a true median. SQL clients used statistics.median correctly.
- **Repaired defaults:** SQL clients default to one warmup plus five measured
  executions. The published October 7-8 reruns explicitly used one plus two;
  three independent corrected cycles have not been completed.
- **Historical aggregation:** true two-sample medians, then geomean over 86 queries.
  Snowflake instead averages its two October 6 cycle medians per query before
  the geomean. Warehouse combines 70 original successes and 16 later retries;
  it is not one uninterrupted clean cycle.
- **Per-query median**, then **geomean** across the 86 queries = the single comparison number
  (geomean resists outlier domination).
- **Historical different metrics:** Spark noop materialization excludes client result transfer.
  SQL-client timing includes submission and full fetch; shared Spark timing excluded
  initial SQL/planning while UC included it. Do not combine these as one metric.
- **Cache policy:** Warehouse `RESULT_SET_CACHING OFF`; lakehouse SQL uses the
  per-SELECT bypass hint; Snowflake `USE_CACHED_RESULT=FALSE`; Spark clears its
  catalog cache after attempts. None of these establishes equal data-cache state.
- Clearing Spark's catalog cache does not clear disk/OS caches. Capacity recycling
  does not prove equal cold-cache state. State the warm-cache experiment explicitly.
- **Timeout:** repaired SQL clients configure a 900 s driver/server statement
  timeout. Warehouse sends a batch; Snowflake sends separate statements, so this
  is not an identical query-file deadline. Historical UC TIMEOUT was unused.
- Unique run directories contain a query-hash manifest, an execution ledger and
  per-query summary. Each execution records result-set cardinalities; values must
  be checked independently. Legacy rows now means last result set on both clients.
- Repaired Spark runners pin queries and retain UUID-scoped event evidence.
  Legacy notebook launchers are not the current publication path.

## Budget Reference and Cost Measurement

The experiment uses Fabric F64's nominal hourly price as a configuration-selection
reference, not an equal-cost constraint. Actual selected list-price models differ.
This is appropriate for exploratory platform selection, but does not identify the
lowest cost to complete a workload.

The historical latency-derived index was:

```
latency_cost_index = (geomean_seconds / 3600) × nominal_hourly_rate
```
This index is not mean cost per query or billed experiment cost. Retain the
original compute sizes and disclose price differences; the hourly budget does
not establish equivalent cores, RAM or allocation. Historical configuration,
QAS use and actual billing require run-level evidence.

A cost-performance follow-up must compare equivalent successful outputs with a
consistent workload-completion boundary and actual charges. It must also measure
sustained repetitions or concurrency before drawing capacity-sizing conclusions.
Do not merge SQL-client full-fetch and Spark noop timings into one cost ranking.

Use a common successful cohort and state the cross-cycle aggregation. For complete
SQL-client cycles, arithmetic-average per-query medians across cycles, then report
their geometric mean. Also report arithmetic mean and distribution. Sum of medians
is not total measured runtime; include warmups, repeats, retries, idle and startup
for experiment costs. Do not count borrowed Fabric capacity as sustainable throughput.

## Native-execution note (Fabric Spark)

NEE was configured enabled, but native markers were absent in the inspected
historical plans. This does not establish the execution mode of every query in
the latest notebook run.
This is insufficient to prove fallback or its performance impact. File-layout,
statistics and engine effects need controlled experiments. No causal claim is made.

## Legacy Lab Reproduction Checklist

1. Provision Fabric, Databricks, Snowflake, and governed storage in one region;
  install Python 3.12 x64, Azure CLI, ODBC Driver 18, and the dependencies listed
  in the [environment guide](03-environment-setup.md).
2. Generate SF3000 data through a validated distributed path, rewrite large
  Delta files to approximately 1 GB, and verify the documented table counts.
3. Configure storage identities and private access, then create the Fabric
  lakehouse shortcuts, native Warehouse tables, and Databricks external tables.
4. Load Snowflake native tables from the same snapshot and validate all 24 counts.
5. Follow the [legacy hosted runbook](RUNBOOK.md), explicitly selecting its
  one-warmup/two-measurement protocol and new evidence directories. This does
  not reproduce the latest zero-warmup client campaign unchanged.
6. Validate query cohort, sample counts, retries, hashes, schemas, and values
  before publishing. Never overwrite historical evidence.

This remains a guided reproduction, not a one-command deployment. Confirm
authorization, budget, idle shared capacity, identities, and current service
settings before starting billable work. See [troubleshooting](09-troubleshooting.md)
for observed access, SQL translation, and capacity failures.

## Validation Status

Local static guards cover retained-query coverage and known integer arithmetic
patterns, not compilation or cross-engine value equivalence. Offline unit tests
exercise timing, true medians, retries, SQL parsing and evidence collection using
fake cursors. Corrected timing reruns completed, but full cross-engine value
correctness and actual allocation and billing remain outside this publication.
