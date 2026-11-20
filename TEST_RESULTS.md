# Test results

## Environment

- **Platform:** Zoho CRM (the author's own test org), REST API v8, Deluge functions
- **Data:** synthetic test records only (names prefixed `SFRTEST-`); no client data
- **Dates:** 2026-11-10 to 2026-11-11
- **Code under test:** the functions in this repository. Test hooks were turned on only for the failure-injection cases (T14, T27, T28) and then reset to their defaults.

## Summary

- 30 test cases are planned (T01–T31; T21 was dropped during planning).
- **28 were run, and all 28 passed.** T01 and T07 failed on the first run. Their root causes were fixed in the code, and both passed on re-test.
- **2 were not run** (T06 and T26). The reasons are in the table below.

## Results

| ID | Case | Result | Notes |
|---|---|---|---|
| T01 | Create parent with rows | Passed | First run failed: a type check in the code rejected every subform value. Passed after the check was fixed |
| T02 | Add a row | Passed | Confirmed through the workflow-triggered diff sync |
| T03 | Edit a row's values | Passed | Confirmed through the workflow-triggered diff sync |
| T04 | Delete a row | Passed | Mirror deleted; snapshot kept in `Deleted_Snapshot`; following rows re-indexed |
| T05 | Delete all rows | Passed | `ALL_DELETE_CONFIRMED` via COQL count; roll-ups reset to 0 / 0 / empty; parent subform left untouched |
| T06 | Subform field missing | Not run | Reproducing it needs the subform field deleted, which would destroy other test data. Guard condition checked in code review |
| T07 | Delete parent → orphan cleanup | Passed | First run failed: a deleted record's GET returns 204 with no error code, which the code didn't expect. Passed after the fix |
| T08 | Duplicate rows (same content) | Passed | Two mirrors with separate keys; COUNT 2 |
| T09 | Delete a row, then re-add the same content | Passed | Old mirror deleted, new mirror added; other mirrors unchanged |
| T10 | 100-row limit | Passed | 100-row parent synced from the button; a 1-row edit synced from the workflow |
| T11 | Reorder rows | Passed | Only `Row_Index` updated; mirror IDs and roll-ups unchanged |
| T12 | Empty values | Passed | Empty `Amount` counts as 0; empty dates are left out of MAX; mirrors keep `null` (not 0 or a default date) |
| T13 | Report on mirrors | Passed | Report total on the mirror module matched the parent roll-up (909.50) |
| T14 | Partial failure (injected) | Passed | Only the targeted mirror failed; result `partial`, parent `needs_resync`, no duplicates |
| T15 | No re-trigger from write-backs | Passed | Workflow function run count unchanged across button and reconcile write-backs |
| T16 | Reconcile repairs drift | Passed | A deleted mirror, a tampered mirror, and a fake mirror were all repaired in one reconcile run |
| T17 | Profile permissions | Passed (observation) | Standard profile has no access to the four SFR modules by default |
| T18 | Subform-only edit | Passed | Editing only the subform fired the workflow (seen in T04 and T11) |
| T19 | Back-to-back saves | Passed | Two saves ran one after the other (42 s apart), not at the same time; no duplicate mirrors |
| T20 | Orphan cleanup (other path) | Passed | No delete trigger in this design; covered by the reconcile path tested in T07 |
| T21 | — | Dropped | Removed during planning |
| T22 | Re-sync button | Passed | 6-row and 100-row parents |
| T23 | Clone parent | Passed | Clone got new mirrors with new keys; the original's mirrors were unchanged |
| T24 | Restore from recycle bin | Passed | The restore didn't fire the workflow. Row IDs were the same after the restore, and the reconcile recreated the mirrors |
| T25 | Import | Passed | The import didn't fire the workflow. The reconcile synced the imported records |
| T26 | Concurrent edit → HTTP 412 | Not run | One attempt was made, but the edit didn't overlap a write-back, so no 412 occurred. The retry logic was checked in code review only |
| T27 | Parent write-back failure (injected) | Passed | Mirrors updated, parent left stale, result `partial`. The button restored it after the hook was turned off |
| T28 | `out_of_spec` (row count over `MAX_ROWS`) | Passed | Nothing written; fail count not increased; back to `synced` after `MAX_ROWS` was restored |
| T29 | API credit usage | Passed | See below |
| T30 | Failure limit | Passed | After 3 consecutive failures the parent became `failed`, and the reconcile skipped it from then on |
| T31 | Bulk-replace guard | Passed | Deleting 3 of 5 rows and adding 1 stopped with `BULK_REPLACE_GUARD`; no mirrors deleted |

## Key measurements

- **100-row sync:** the 100-row parent synced from the re-sync button (100 mirrors added; SUM 5050.00, COUNT 100). The button didn't time out. The logs show time only to the minute, so the exact duration couldn't be measured. Based on the log timestamps, the run took at most about 53 s.
- **Reconcile run time:** one full on-demand reconcile of the test org finished in 11.84 s. The org had 12 parent records at the end of testing.
- **API credits:** in the 24-hour window covering the main test run, credits by function were 69 for `sfr_recon_run`, 42 for `sfr_reconcile`, 33 for `sfr_sync_mirror`, and 33 for `sfr_resync_button`: 177 in total. Credits for `sfr_sync_core` are counted under the function that calls it. These figures come from Zoho's API credit usage dashboard.
- **Report vs. roll-up:** a report on `SFR_Line_Mirrors`, filtered to one parent and grouped by `Line_Item` with SUM of `Amount`, totaled **909.50**. That matches the parent's `Rollup_Total_Amount` (909.50, `Rollup_Line_Count` 6). The report screen rounds the total to ¥910 because the org displays yen without decimals; 909.50 is the sum of the record values.

## Other observations

- In the test org, two of the hourly reconcile runs (00:30 and 02:30 on 2026-11-11) left no log and no state update. The runs before and after them worked normally. The cause wasn't identified.
- When a parent write-back fails, the button's status label can still say "synced", because the label is set before the write-back. The `Sync_Result` in the log is the accurate result.
