# Setup

This guide sets up the sample in a Zoho CRM org. The API names below match the constants in the code. If you use different names, update the constants in every function that uses them (see [Configuration constants](#configuration-constants)).

Do this in a sandbox or a test org first.

## 1. Modules and fields

Create four custom modules. Once each field exists, check its API name: Zoho may assign one that differs from the label.

### SFR_Projects (parent)

| Field (API name) | Type | Notes |
|---|---|---|
| `Name` | Single line (standard) | Record name |
| `Work_Lines` | Subform | Rows to mirror (fields below). In the layout's **Subform Properties**, set the maximum row count to **100** (the default is 25) |
| `Rollup_Total_Amount` | Currency, 2 decimal places | SUM of `Amount`. Written by the function only |
| `Rollup_Line_Count` | Number (integer) | COUNT of rows |
| `Rollup_Latest_Work_Date` | Date | MAX of `Work_Date`; empty when no row has a date |
| `Mirror_Sync_Status` | Picklist | Values: `synced`, `needs_resync`, `failed`, `out_of_spec` |
| `Mirror_Last_Synced_At` | Date/Time | Time of the last sync that changed the parent's values |
| `Mirror_Last_Error` | Single line | Summary of the latest failure, cut to 255 characters |
| `Mirror_Fail_Count` | Number (integer) | Consecutive failures |

`Work_Lines` subform fields:

| Field (API name) | Type |
|---|---|
| `Line_Item` | Single line (text) |
| `Amount` | Currency, 2 decimal places |
| `Work_Date` | Date |

The `Rollup_*` and `Mirror_*` fields should be read-only for regular users, so that only the functions change them.

### SFR_Line_Mirrors (mirror)

One record per subform row. Copied fields should match the subform exactly: same type and same decimal places. If decimal places differ, writes fail with `INVALID_DATA`.

| Field (API name) | Type | Notes |
|---|---|---|
| `Name` | Single line (standard) | Generated as `<parent Name>#<Mirror_Key>`, cut to 255 characters |
| `Parent_Project` | Lookup → SFR_Projects | Lets reports pull parent fields |
| `Parent_Record_Id` | Single line | Parent record ID as text; used to find orphans |
| `Subform_Row_Id` | Single line | Subform row ID as text |
| `Mirror_Key` | Single line | `<Parent_Record_Id>:<Subform_Row_Id>`; used to match mirrors to rows |
| `Line_Item` | Single line | Copy |
| `Amount` | Currency, 2 decimal places | Copy |
| `Work_Date` | Date | Copy |
| `Row_Index` | Number (integer) | Row position in the subform (1-based) |

Leave the copied fields optional so that a row with empty values can still be mirrored. Regular users should not be able to create, edit, or delete mirrors. In testing, the Standard profile had no access to any of the four SFR modules by default. If users need the mirrors for reports, give them View only.

### SFR_Sync_Logs

One record per sync run, plus delete-snapshot records.

| Field (API name) | Type | Notes |
|---|---|---|
| `Name` | Single line (standard) | Run ID: `SYNC-<timestamp>-<parentId>`, or `RECON-<timestamp>` for a reconcile run |
| `Parent_Record_Id` | Single line | Empty on reconcile summary logs |
| `Sync_Source` | Picklist | `workflow`, `button`, `reconcile`, `delete` (`delete` is accepted by the core but not used by the included functions) |
| `Sync_Result` | Picklist | `delete_planned`, `success`, `partial`, `failed`, `skipped`, `out_of_spec`, `timeout` |
| `Added_Count` | Number (integer) | |
| `Updated_Count` | Number (integer) | |
| `Deleted_Count` | Number (integer) | |
| `Error_Detail` | Multi-line (small) | Codes and notes, cut to 2,000 characters |
| `Deleted_Snapshot` | Multi-line (large) | Key and values of mirrors before they are deleted. Split across records every 32,000 characters |
| `Started_At` | Date/Time | |
| `Ended_At` | Date/Time | |

### SFR_Sync_State

Holds one record, named `reconcile`. The reconcile function creates it on its first run if it doesn't exist.

| Field (API name) | Type | Notes |
|---|---|---|
| `Name` | Single line (standard) | `reconcile` |
| `Last_Run_At` | Date/Time | End time of the latest reconcile |
| `Last_Run_Result` | Single line | `success`, `timeout`, or `failed` |

## 2. Connection

Create a Zoho CRM connection with the link name **`crm_module`**. Every `invokeurl` call in the code refers to this exact name.

Scopes needed:
- Create, read, update, and delete on the four SFR modules (for example, `ZohoCRM.modules.ALL` or the narrower custom-module scope)
- COQL read (for example, `ZohoCRM.coql.READ`)

Check the exact scope names in Zoho's current OAuth scope documentation when you create the connection.

## 3. Functions

Register the functions in **Setup → Developer Hub → Functions**. Create `sfr_sync_core` first, because the other functions call it.

| Function | Category | Argument | Notes |
|---|---|---|---|
| `sfr_sync_core` | Standalone | `parentId` (String), `source` (String) | Sync engine. The other functions call it as `standalone.sfr_sync_core(...)` |
| `sfr_sync_mirror` | Automation | `parentId` (String) | Called by the workflow rule; passes the record ID to the core |
| `sfr_reconcile` | Schedule | none | Hourly reconcile |
| `sfr_resync_button` | Button | `parentId` (String) | Put it on the SFR_Projects detail page |
| `sfr_recon_run` | Standalone | none | Optional. Runs the reconcile on demand for testing |

**Button setup:** add a custom button on the SFR_Projects **detail page** that runs `sfr_resync_button`. Map `parentId` to the record ID merge field (in the test org it was labeled *SFR Projects - SFR Project Id*). Don't type the literal text `Id`. During testing that mistake passed the string `"Id"` as the parent ID.

## 4. Workflow rule

Create one rule on **SFR_Projects**:

- **When:** on record action **Create or Edit**
- **Repeat:** run each time the record is edited (not only the first time)
- **Condition:** none (all records). A field-based condition may miss edits that change only the subform.
- **Instant action:** call the function `sfr_sync_mirror`, with `parentId` mapped to the record ID

Write-backs from the functions send an empty `trigger` list, so they don't start this rule again.

## 5. Schedule

Create a schedule that runs **`sfr_reconcile` every hour**.

Each run writes one `RECON-...` summary log and updates `SFR_Sync_State`. During testing, some hourly runs left no trace (see the README's Limitations section). Monitor `Last_Run_At` if you depend on the reconcile.

## Configuration constants

These constants are at the top of `sfr_sync_core`:

| Constant | Default | Meaning |
|---|---|---|
| `API_BASE` | `https://www.zohoapis.jp/crm/v8` | API base URL. Change the domain for your data center (e.g. `www.zohoapis.com`, `www.zohoapis.eu`) |
| `CONN` | `crm_module` | Connection name, for reference. The `invokeurl` blocks hard-code `"crm_module"`, so if you rename the connection, update those too |
| `PARENT_MODULE` | `SFR_Projects` | Parent module API name |
| `SUBFORM_FIELD_API` | `Work_Lines` | Subform key in the parent record |
| `SUBFORM_COQL_API` | `Work_Lines` | Subform name used in COQL `FROM`. Usually the same as above, but it can differ |
| `MIRROR_MODULE` | `SFR_Line_Mirrors` | Mirror module API name |
| `LOG_MODULE` | `SFR_Sync_Logs` | Log module API name |
| `MAX_ROWS` | `100` | Parents with more rows are marked `out_of_spec` and skipped. Keep it equal to the subform's row limit |
| `FAIL_LIMIT` | `3` | Consecutive failures before a parent becomes `failed` (provisional value) |
| `TEST_FAIL_MIRROR_KEY` | `""` | Test hook only. Keep empty |
| `TEST_FAIL_PARENT_UPDATE` | `false` | Test hook only. Keep `false` |

`sfr_reconcile` has its own copies of `API_BASE`, `CONN`, and the module names. Keep them consistent with `sfr_sync_core`. It also defines run limits: `MAX_PARENTS_PER_RUN = 100`, `DEADLINE_SEC = 780`, and `ROWS_BUDGET_PER_RUN = 1500`. When a run hits any of these limits, it stops with result `timeout`, and the next run starts again from the first parent.

The subform's row fields (`Line_Item`, `Amount`, `Work_Date`) are referenced directly in the code, not through constants. To mirror different fields, edit the row-reading and record-building sections of `sfr_sync_core`.

## Quick check

1. Create an SFR_Projects record with a few subform rows. A `SYNC-...` log with result `success` should appear, along with one mirror per row. The roll-ups should be filled in, and the status should be `synced`.
2. Edit one row's `Amount`. The log should show `Updated_Count` 1, and `Rollup_Total_Amount` should change.
3. Click the re-sync button on a record with no changes. The message should report no changes, and the result should be `skipped`.
4. Run `sfr_recon_run` (or wait for the hourly schedule). A `RECON-...` log should appear, and `SFR_Sync_State.Last_Run_At` should update.
