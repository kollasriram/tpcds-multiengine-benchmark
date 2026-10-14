# End-to-end publication runbook

**Databricks query runtime:** Databricks Runtime 16.4, Scala 2.12
(`16.4.x-scala2.12`), used for both Photon and Standard publication lanes.

This records the procedure for the **legacy six-lane hosted protocol** in the
private benchmark kit. Its entry point is `harnesses/run_publication.py`;
platform-specific harnesses are implementation modules and recovery tools.
Those files, query bundles and notebooks are not included in this
documentation-only release. The commands below are historical reference,
not runnable instructions for this public repository.

It is not the runner used for the October 11-13 zero-warmup Windows-client
measurements in [latest results](RESULTS.md). This hosted protocol uses repeated
executions, Spark noop materialization and Databricks job clusters. The latest
client runs used full consumption and fresh all-purpose Databricks clusters;
their controller and raw evidence are not distributed in this public export.
Running this runbook unchanged does not reproduce the latest result table.

The public export contains summary results, not historical run evidence. Keep
new campaign plans, qualification receipts and evidence in a private working
directory outside this export. Generated local `results/` artifacts are ignored
by Git; exclude them from public archives as well.

## Fixed contract

- Lane order: Fabric Warehouse, Fabric Lakehouse SQL endpoint, Fabric Spark,
  Databricks Photon, Databricks Standard, Snowflake.
- Exact retained cohort: 86 query IDs from `queries/qids.csv`.
- Each query receives one warmup and two measured executions.
- At most one lane runs at a time. The predecessor's durable evidence must
  validate before the next lane starts.
- SQL lanes use `sql_client_full_fetch`; Spark lanes use
  `spark_sql_plan_noop`. Never rank the two metric groups together.
- A timed-out terminal wait is not a failed submission. Resume with the durable
  platform ID; never duplicate a hosted submission.
- Fabric is paused in a `finally` path. Databricks job clusters are ephemeral.
  Snowflake cleanup suspends the task and warehouse.

## Prerequisites

1. Prepare the same 24-table SF3000 snapshot on every platform as described in
   [dataset preparation](04-dataset-generation.md).
2. Configure the resources and identities in
   [environment setup](03-environment-setup.md).
3. Install Python dependencies, Azure CLI, Databricks CLI, Snowflake Connector,
   `pyodbc`, and Microsoft ODBC Driver 18.
4. Authenticate to Azure and Databricks. Store the Snowflake password in a local
   ignored file; never place it in source or command output.
5. Run the offline checks:

```powershell
$Python = (Get-Command python).Source
$RunRoot = 'C:\benchmark-private'
& $Python -m unittest discover -s tests -v
& $Python harnesses\lab_preflight.py --platform repository --json
```

## Smoke qualification

Before publication, run each lane over query IDs `1,2,3,6,7`, with zero warmups
and one measured execution. Qualify its persisted evidence using
`smoke_gate.py qualify-fabric` or `smoke_gate.py qualify-databricks`; Snowflake
qualification is recorded from its collected SQL evidence. The publication
runner refuses to start unless every lane in the qualification file is
`QUALIFIED`.

Validate the gate:

```powershell
& $Python harnesses\smoke_gate.py validate `
  --qualification "$RunRoot\smoke-qualification.json"
```

## Freeze a campaign

Create an immutable plan only after smoke qualification and source validation:

```powershell
$CampaignId = [guid]::NewGuid().ToString()
$CampaignRoot = "$RunRoot\publication\$CampaignId"
New-Item -ItemType Directory -Path $CampaignRoot -Force | Out-Null
& $Python harnesses\publication_plan.py create `
  --campaign-id $CampaignId `
  --output "$CampaignRoot\plan.json"
& $Python harnesses\publication_plan.py validate `
  --plan "$CampaignRoot\plan.json"
```

Do not recreate a plan after any full lane has been submitted: lane UUIDs are
part of the durable recovery contract. A narrowly scoped recovery-code repair
must use `publication_plan.py amend-source` and cite an existing durable
submission. Query, protocol, and notebook drift are not amendable.

## Run or resume everything

The same command starts a new campaign or resumes an interrupted one:

```powershell
& $Python harnesses\run_publication.py `
  --plan "$CampaignRoot\plan.json" `
  --qualification "$RunRoot\smoke-qualification.json" `
  --snowflake-password-file C:\secure\snowflake-password.txt
```

The runner:

1. validates qualification and frozen source hashes;
2. activates Fabric, runs and validates its three lanes serially, then pauses it;
3. runs Photon and Standard on ephemeral Databricks job clusters and cleans each;
4. runs the resumable Snowflake task and suspends task and warehouse;
5. writes `evidence-lanes.json` and deterministically aggregates final results.

Evidence is written below your private campaign directory:

```text
<private-run-root>/publication/<campaign-id>/
  plan.json
  evidence/<lane>/
  evidence-lanes.json
  final/final_results.csv
  final/final_results.json
  final/per_query_medians.csv
```

## Recovery rules

1. Rerun the same `run_publication.py` command after a local interruption.
2. The runner first validates collected evidence. Complete lanes are skipped.
3. If evidence is absent, it reads the durable submission record below
   `results/batch-submissions/<run-id>/` and waits/collects that platform run.
4. Do not call a platform `submit` command again after a timeout.
5. Snowflake terminal failures retain successful checkpoints. Repair allowed
   recovery code, dispatch the same run UUID, then rerun the publication command.
6. A Fabric failure still executes the capacity-pause path. Verify with:

```powershell
& $Python harnesses\publication_preflight.py ensure-fabric `
  --plan "$CampaignRoot\plan.json" --state Paused --timeout 900
```

For an unlisted state, stop submissions, leave durable evidence untouched, and
diagnose the existing platform run ID before changing code.

## Verify publication

```powershell
& $Python harnesses\publication_plan.py validate `
  --plan "$CampaignRoot\plan.json"
& $Python harnesses\final_results.py `
  --plan "$CampaignRoot\plan.json" `
  --lanes "$CampaignRoot\evidence-lanes.json" `
  --output "$CampaignRoot\final"
```

A valid publication contains exactly 86 successful query summaries per lane,
all expected repetitions, and two separate metric-group rankings. Preserve the
plan, query hashes, raw lane evidence, platform run IDs, and final outputs
together.

## Maintainer interfaces

- `run_publication.py`: sole end-to-end operator command.
- `publication_plan.py`: immutable IDs, protocol, and source hashes.
- `publication_preflight.py`: resource and predecessor-evidence gates.
- `fabric_batch_job.py`: Fabric submit, wait, collect, and same-UUID recovery.
- `databricks_batch_job.py`: Databricks submit, wait, collect, and cleanup.
- `snowflake_batch_job.py`: Snowflake checkpoints, dispatch, wait, collect, cleanup.
- `final_results.py`: strict evidence validation and aggregation.

The notebooks are optional interactive interfaces. The hosted publication uses
the Python orchestration path above for Snowflake and final aggregation.
