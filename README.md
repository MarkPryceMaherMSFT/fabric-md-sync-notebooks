# Fabric metadata sync notebooks

Two Microsoft Fabric notebooks for assessing SQL analytics endpoint metadata sync
and upgrading eligible tables from the legacy **V1** architecture to **V2**.
Use the workspace notebook for a repeatable, audited assessment across a workspace,
or the smaller notebook for an individual lakehouse.

The background and upgrade workflow below are based on Microsoft's public
documentation:

- [SQL analytics endpoint metadata sync and the upgrade process](https://learn.microsoft.com/en-us/fabric/data-engineering/sql-analytics-endpoint-metadata-sync#upgrade-existing-sql-analytics-endpoints-to-new-metadata-sync)
- [What is the SQL analytics endpoint for a lakehouse?](https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-sql-analytics-endpoint)

## What MD Sync does

A lakehouse stores its tables in Delta format in OneLake. Fabric automatically
provisions a **SQL analytics endpoint**, which provides a read-only T-SQL query
surface over those Delta tables, using the same engine as Fabric Data Warehouse.
Data does not need to be copied into a separate SQL database to query it.

**Metadata sync (MD Sync)** connects the lakehouse's Delta tables to the endpoint's
SQL catalog. It reads the Delta transaction logs under the OneLake `/Tables`
folder and keeps the SQL view of the lakehouse up to date:

- **Table discovery:** detects newly created or dropped Delta tables and creates
  or removes their corresponding SQL tables.
- **Schema changes:** detects changes such as added or removed columns and data
  type changes, then refreshes the SQL table definition.
- **Data freshness:** detects inserts, updates, and deletes in existing Delta
  tables so SQL queries can see the underlying data changes.

This synchronization is normally managed by Fabric. The notebooks do not replace
that background service: they inspect its architecture version, assess table
compatibility, and invoke the supported upgrade APIs when requested.

Upgrading MD Sync does **not** make the SQL analytics endpoint writable. Inserts,
updates, and deletes against the underlying lakehouse data still need to be
performed through the lakehouse, rather than through its read-only SQL endpoint.

## Why use MD Sync V2?

Microsoft describes the new metadata sync as making lakehouse data available to
query **within seconds** of it arriving. Its main architectural improvements are:

| Improvement | Benefit |
| --- | --- |
| External-table-based Delta log parsing and catalog creation | Replaces the legacy regular-table synchronization architecture. |
| Separate detection of schema and data changes | Decouples these refresh paths rather than treating every change as the same catalog operation. |
| Periodic background data refresh | Keeps checking for new data independently of an explicit manual refresh. |
| On-demand refresh when a read query detects stale data | Provides another freshness path at query time. |
| Automatic freshness for V2 tables | Removes the need to keep scheduling the older metadata refresh REST API solely to maintain data freshness for those tables. |

The SQL analytics endpoint documentation also states that **`varchar(max)` is
supported only with new metadata sync**; legacy endpoints can still truncate
data to 8 KB, including shortcuts to mirrored items. This is a documented
endpoint capability, not a promise that these notebooks automatically change
column definitions.

"Within seconds" describes the new sync design, not a latency guarantee made by
the notebooks. Table compatibility and the latest documented limitations still
apply.

Enabling **New metadata sync** in workspace settings affects newly provisioned
endpoints. It does **not** upgrade existing V1 endpoints; those need the assessment
and upgrade process.

## Notebook features

| Notebook | Purpose |
| --- | --- |
| [Workspace Validation and Upgrade](notebooks/MD_Sync_Workspace_Validation_Upgrade.ipynb) | Discover all lakehouses/SQL endpoints in a workspace, assess upgrade readiness, optionally upgrade, and append responses and outcomes to a Delta audit table. |
| [Single Lakehouse](notebooks/MD_Sync_Single_Lakehouse.ipynb) | Take one `lakehouse_id`, select an action enum, and display endpoint and per-table assessment/upgrade results. |

### Workspace Validation and Upgrade

This notebook is designed for workspace-wide assessment and later review:

- Discovers lakehouses and SQL analytics endpoints, including nested folders and
  paginated results, and avoids processing the same endpoint twice.
- Checks the current architecture and migration state. Healthy V2 endpoints are
  skipped; V2 endpoints with known remaining V1 failures are still assessed.
- Runs non-mutating preflight on endpoints that still need conversion and records
  eligibility counts and named table blockers/corrective actions.
- Supports two explicit upgrade switches:

| Setting | Behavior |
| --- | --- |
| Both switches `False` | Validate and log only. |
| `UPGRADE_IF_ALL_ELIGIBLE = True`, `FORCE_UPGRADE = False` | Upgrade only when every evaluated table is eligible and no blockers are reported. |
| `FORCE_UPGRADE = True` | Attempt the remaining V1 tables despite known eligibility blockers, after a valid assessment. |

It rechecks endpoint state before an upgrade, polls asynchronous REST operations,
and queries state again after an upgrade attempt. It distinguishes successful,
partial, and failed outcomes rather than treating HTTP success or an endpoint
version of V2 as proof that every table converted.

Responses, operation IDs/states, table blockers/results, decisions, and endpoint
summaries are appended to the configured **Delta audit table**. Each run has a
`run_id`, so results can be compared later without overwriting history. An audit
write failure stops the run rather than allowing an unrecorded upgrade.

The parsing supports both documented migration-attempt field names and the
shorter names observed in MSIT, while retaining the original responses for review.

### Single Lakehouse

This notebook provides a smaller, interactive workflow:

- Takes **one notebook input parameter**, `lakehouse_id`.
- Derives the workspace from the notebook runtime and resolves the associated
  SQL analytics endpoint. The lakehouse must be in that workspace.
- Shows endpoint status and discovers Delta table names through OneLake metadata,
  including schema-enabled lakehouses.
- Displays per-table readiness and corrective actions. If the discovered table
  inventory and preflight counts disagree, it reports the mismatch instead of
  inventing eligibility for individual tables.
- Uses an editable `Action` enum rather than additional upgrade input flags.

| Action | Behavior |
| --- | --- |
| `Action.DUMMY_RUN` | Validate only and display endpoint/table details. |
| `Action.UPGRADE` | Invoke the endpoint upgrade when eligible V1 tables exist; display the results and known blocked tables that remain unconverted. |
| `Action.FORCE_UPGRADE` | Attempt remaining V1 tables even when preflight reports none eligible, then raise an error naming tables that did not upgrade. |

The single-lakehouse notebook displays its results and does **not** create an audit
table. Force mode checks the returned per-table results and post-upgrade failures;
it cannot make an incompatible Delta table succeed.

## Assessment and upgrade workflow

The notebooks follow Microsoft's documented sequence:

1. **Check the current version/state.** `V1` means legacy metadata sync; `V2`
   means the new architecture. Also inspect active migration and table failures.
2. **Run preflight.** The assessment does not convert tables. It returns evaluated
   and eligible table counts, plus named blockers and corrective actions.
3. **Review compatibility issues.** Resolve blockers where possible before
   selecting an upgrade action. A preflight result is point-in-time, not a
   guarantee against later changes.
4. **Upgrade during a maintenance window.** The upgrade changes tables in the SQL
   analytics endpoint from the regular-table to external-table architecture.
5. **Inspect every result.** If any tables fail, address their errors and retry.
   Repeated upgrade requests reattempt only tables that have not reached V2.

The management flow uses Fabric REST APIs; it does not need SQL/TDS or DAX
queries to trigger an assessment or upgrade. The single-lakehouse table inventory
uses NotebookUtils' OneLake file-system access.

## Blockers and partial upgrades

Microsoft's guidance highlights these compatibility issues:

- **Multi-part Delta checkpoints:** these are not supported by the new metadata
  sync. Use the managing runtime's supported migration procedure to single-part
  classic or V2 checkpoints.
- **More than 1,000 commits since the last checkpoint:** consider creating a
  checkpoint before upgrading.
- **Column mapping by ID:** review the managing runtime's supported migration
  options. Do not assume that changing `delta.columnMapping.mode` directly is
  supported; removing mapping can rewrite files and require control of writers.

Delta checkpoint policies named `classic` or `v2` are separate from MD Sync
architecture versions **V1/V2**. Neither notebook automatically rewrites data,
changes Delta properties, or repairs these blockers.

**After a partial upgrade, failed V1 tables do not sync automatically.** If at
least one table converts, the endpoint can report V2 even while other tables
remain V1. If no tables convert, the endpoint remains V1. Review the per-table
results rather than relying only on the endpoint's version.

The REST API upgrades an **endpoint**, not a caller-selected table list. Filtering
or displaying table results does not change the scope of the upgrade request.
There is no server-side `force` parameter: force is the notebook's decision to
attempt conversion despite known blockers.

## Before running

Import the `.ipynb` file into Fabric and use **PySpark (Python)**. Authentication
uses the signed-in notebook user's permissions; credentials are acquired at
runtime and are not embedded in the source.

These snapshots preserve the saved notebook configuration:

- The workspace notebook currently has `UPGRADE_IF_ALL_ELIGIBLE = True`.
- The single-lakehouse notebook currently has `action = Action.FORCE_UPGRADE`.

**Review the configuration before running.** For validation only, set both
`UPGRADE_IF_ALL_ELIGIBLE` and `FORCE_UPGRADE` to `False` in the workspace
notebook, or set `action = Action.DUMMY_RUN` in the single-lakehouse notebook.
Workspace/lakehouse IDs are retained as configuration; replace them for another
environment.

For the workspace notebook, attach the configured audit lakehouse as the default
lakehouse. The single-lakehouse notebook derives its workspace from the notebook
runtime and discovers tables through OneLake metadata without creating an audit
table.

## Permissions

The user needs appropriate workspace/lakehouse read access, SQL analytics
endpoint write permission for preflight and upgrade, and audit-lakehouse write
access when using the workspace notebook. OneLake metadata discovery also uses
the user's data access permissions.

The metadata sync migration REST APIs are currently **Preview**. Check the latest
API documentation, permissions, and limitations before production use.

## Source hygiene

Execution outputs, execution counts, and user/session metadata are removed from
these exports. The executable notebook code and parameter settings are preserved.
Fabric sensitivity labels do not travel into Git, so this repository is private.
Do not commit credentials, runtime outputs, or generated audit data.

## Documentation

- [SQL analytics endpoint metadata sync: upgrade existing endpoints to V2](https://learn.microsoft.com/en-us/fabric/data-engineering/sql-analytics-endpoint-metadata-sync#upgrade-existing-sql-analytics-endpoints-to-new-metadata-sync)
- [Lakehouse SQL analytics endpoint: purpose, capabilities, and limitations](https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-sql-analytics-endpoint)
- [Get metadata sync migration status REST API](https://learn.microsoft.com/en-us/rest/api/fabric/sqlendpoint/items/get-sql-endpoint-metadata-sync-migration-status)
- [Preflight metadata sync upgrade REST API](https://learn.microsoft.com/en-us/rest/api/fabric/sqlendpoint/items/preflight-sql-endpoint-metadata-sync-upgrade)
- [Upgrade metadata sync REST API](https://learn.microsoft.com/en-us/rest/api/fabric/sqlendpoint/items/upgrade-sql-endpoint-metadata-sync)
