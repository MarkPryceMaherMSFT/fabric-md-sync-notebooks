# Fabric metadata sync notebooks

Source snapshots of two Microsoft Fabric notebooks for assessing and upgrading
SQL analytics endpoint metadata sync from V1 to V2.

| Notebook | Purpose |
| --- | --- |
| [Workspace Validation and Upgrade](notebooks/MD_Sync_Workspace_Validation_Upgrade.ipynb) | Discover all lakehouses/SQL endpoints in a workspace, assess upgrade readiness, optionally upgrade, and append responses and outcomes to a Delta audit table. |
| [Single Lakehouse](notebooks/MD_Sync_Single_Lakehouse.ipynb) | Take one `lakehouse_id`, select an action enum, and display endpoint and per-table assessment/upgrade results. |

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

## Permissions and behavior

The user needs appropriate workspace/lakehouse read access, SQL analytics
endpoint write permission for preflight and upgrade, and audit-lakehouse write
access when using the workspace notebook.

Preflight is non-mutating and point-in-time. Upgrades operate at endpoint level
and attempt the remaining V1 tables; the REST API does not accept a caller-selected
table list or a server-side force flag. Force mode cannot make unsupported Delta
features compatible.

**After a partial upgrade, failed V1 tables do not sync automatically.** Review
the per-table results, resolve the reported blockers, and retry the remaining
tables in a maintenance window.

## Source hygiene

Execution outputs, execution counts, and user/session metadata are removed from
these exports. The executable notebook code and parameter settings are preserved.
Fabric sensitivity labels do not travel into Git, so this repository is private.
Do not commit credentials, runtime outputs, or generated audit data.

## Reference

[Microsoft's metadata sync upgrade process](https://learn.microsoft.com/en-us/fabric/data-engineering/sql-analytics-endpoint-metadata-sync#upgrade-existing-sql-analytics-endpoints-to-new-metadata-sync)
