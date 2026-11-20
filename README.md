# zoho-crm-subform-rollup-sample

A Deluge sample for Zoho CRM. It copies subform rows into a regular child module so you can report on them, then writes roll-ups (SUM / COUNT / MAX) back to the parent record.

## The problem

Zoho CRM subforms are handy for line items, but they have two gaps:

- **Subform fields aren't available in Zoho CRM reports.** You can't group, filter, or total subform rows the way you can with a normal module.
- **Standard roll-up summary fields can't aggregate subform rows.** They work on related-list (lookup) records, not on rows inside a subform.

This sample works around both. Each subform row is copied to a record in a child module (the "mirror"). Reports run on the mirror module, and the function writes roll-up values back to the parent.

## How it works

**One-way diff sync.** The subform is the source of truth. A row's identity is **parent record ID + subform row ID** (stored as `Mirror_Key` = `<Parent_Record_Id>:<Subform_Row_Id>`). Each run reads the parent's subform, loads the existing mirrors with COQL, and inserts, updates, or deletes only the mirrors that differ. Mirrors are read-only copies; any direct edits are overwritten on the next sync.

**Three entry points, one core.** All sync logic lives in `sfr_sync_core`. Three thin wrappers call it:

| Entry point | Function | When it runs |
|---|---|---|
| Workflow rule | `sfr_sync_mirror` | Each time a parent record is created or edited (editing only the subform also counts) |
| Re-sync button | `sfr_resync_button` | Manual run from the parent's detail page. Runs even for parents in `failed` status and resets the failure count when it succeeds |
| Hourly reconcile | `sfr_reconcile` | Re-syncs every parent in ID order and removes orphan mirrors. Catches changes that never fire a workflow (imports, restores, partial failures, hand edits to mirrors) |

**Roll-ups are fully recomputed.** Each run recalculates all three values from the current subform rows instead of adjusting the previous totals:

- `Rollup_Total_Amount`: SUM of `Amount`. Empty amounts count as 0.
- `Rollup_Line_Count`: COUNT of rows.
- `Rollup_Latest_Work_Date`: MAX of `Work_Date`. Empty dates are ignored. The field is empty when there are no dated rows.

**The parent is written only when something changed.** The function compares five fields: the three roll-ups, `Mirror_Sync_Status`, and `Mirror_Last_Error`. If none changed, it doesn't write to the parent. When it does write, it sends the `If-Unmodified-Since` header with the `Modified_Time` it read. If the parent was edited in the meantime (HTTP 412), it re-reads the parent and retries once. Write-backs send an empty `trigger` list, so they don't start the workflow again.

**The subform key is never sent to the parent.** The update payload is explicitly stripped of the subform field. In Zoho CRM, sending a subform key with an empty array deletes every row, so this guard prevents that kind of accident.

**Deleted mirrors are snapshotted first.** Before deleting mirrors, the function writes their key and values to `SFR_Sync_Logs.Deleted_Snapshot` with result `delete_planned`. If that log insert fails, nothing is deleted. After the deletes, the same log record is updated with the final result.

**Delete safety guards.**
- *Delete all:* if the subform now has no rows but mirrors still exist, a COQL `COUNT(id)` on the subform must return 0 before any mirror is deleted.
- *Bulk replace:* if a save would delete more than half of the existing mirrors while also adding new ones, the run stops with `BULK_REPLACE_GUARD` and deletes nothing.
- *Missing subform:* if the parent can be read but the subform key is missing, the run stops. It never treats a missing subform as "zero rows".
- *Parent gone:* a parent is treated as deleted only when the single-record GET returns no record **and** a COQL `COUNT(id)` returns 0. Only then are its mirrors removed.

**Failure limit.** A run that fails or partly fails sets the parent to `needs_resync` and increments `Mirror_Fail_Count`. When the count reaches `FAIL_LIMIT`, the parent becomes `failed`. The workflow and the reconcile job then skip it until someone fixes the cause and runs the re-sync button. A parent with more rows than `MAX_ROWS` is marked `out_of_spec`. Nothing is written for it, and its failure count doesn't go up.

## Repository contents

| File | Type | Purpose |
|---|---|---|
| `sfr_sync_core.deluge` | standalone | Sync engine: reads the parent, diffs and writes mirrors, verifies by reading back, writes roll-ups, logs the run |
| `sfr_sync_mirror.deluge` | automation | Called by the workflow rule; runs the core with source `workflow` |
| `sfr_resync_button.deluge` | button | Detail-page button; runs the core with source `button` and shows a summary message |
| `sfr_reconcile.deluge` | schedule | Hourly reconcile across all parents, orphan cleanup, one summary log per run, updates `SFR_Sync_State` |
| `sfr_recon_run.deluge` | standalone (helper) | Runs `schedule.sfr_reconcile()` on demand, so you can test the reconcile without waiting for the schedule. Not needed in production |

User-facing strings (button messages, code comments) are in Japanese. Constants, API names, and log codes are in English.

## Setup

See [SETUP.md](SETUP.md) for module and field definitions, function registration, the workflow rule, the schedule, the connection, and the configuration constants.

## Testing

See [TEST_RESULTS.md](TEST_RESULTS.md) for the full table.

Summary: the test plan has 30 cases (T01–T31; T21 was dropped during planning). **28 were run and all 28 passed.** Two of them, T01 and T07, failed on the first attempt. The cause was found, fixed in the code, and both cases passed on re-test. **2 cases were not run**:

- **T06 (subform field missing):** reproducing it would mean deleting the subform field, which would destroy other test data. The guard was checked in code review instead.
- **T26 (concurrent edit → HTTP 412):** I couldn't line up an overlapping edit in time to trigger a 412 on the write-back. The If-Unmodified-Since + retry-once logic was checked in code review only.

## Measured notes

These were observed while testing against a live Zoho CRM org (API v8). They may help with your own subform work:

- **A GET for a deleted record returns HTTP 204 with no body.** No error code comes back. To detect a deleted parent reliably, the code combines the empty response with a COQL `COUNT(id)` that returns 0.
- **An empty subform reads differently depending on the method.** A raw `invokeurl` GET returns the subform key with a `null` value. `zoho.crm.getRecordById` returns an empty List. The core uses `getRecordById` so an empty subform is never mistaken for a missing one.
- **Subform row IDs are stable.** They didn't change when row values were edited, when rows were reordered, or when the parent was restored from the recycle bin. That's why row ID works as the identity key.
- **Imports and recycle-bin restores don't fire the workflow.** The hourly reconcile picks up those records. (The import wizard's "trigger automation" option is off by default; it was left off.)
- **Function time was UTC-8 in this test org.** `zoho.currenttime` ran 17 hours behind the tester's local time (JST), whatever the org or account settings suggested. Log timestamps and run IDs use the function clock, so read them on that basis.

## Limitations

- **One parent module and one subform per deployment.** They are set by constants at the top of each function. Covering more subforms means copying and adjusting the functions.
- **The mirror is one-way and read-only.** Edits made directly to mirror records are overwritten. Write access to the mirror module should be restricted.
- **`FAIL_LIMIT = 3` is a provisional value.** It hasn't been tuned against real workloads.
- **Two hourly reconcile runs left no trace during testing.** The 00:30 and 02:30 runs on 2026-11-11 wrote no reconcile log and no state update. Runs before and after them did. The cause wasn't identified. If you depend on the reconcile, monitor `SFR_Sync_State.Last_Run_At`.
- When a parent write-back fails, the failure shows up only in `SFR_Sync_Logs`. The parent record itself keeps its previous status. The next reconcile compares values and writes them again.
- When `BULK_REPLACE_GUARD` stops a run, no mirrors change, but the roll-ups are still written from the current subform.
- `API_BASE` points to the `zohoapis.jp` data center. Change it for other data centers (see SETUP.md).

## About

This is a prototype I built and tested in my own Zoho CRM org using synthetic data only. It isn't client work, and no client data was involved.

`sfr_sync_core` has failure-injection hooks for testing: `TEST_FAIL_MIRROR_KEY` and `TEST_FAIL_PARENT_UPDATE`. They are off by default (`""` and `false`) and must stay that way in normal use.
