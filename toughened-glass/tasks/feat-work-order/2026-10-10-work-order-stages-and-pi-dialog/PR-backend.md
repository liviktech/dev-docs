# 2026-10-10 - work-order-stages-and-pi-dialog (backend)

## Summary

Every work order used to get a stage row for each work center on its route, including Dispatch and HOLE. Dispatch has its own module and HOLE is a system station, so operators should not scan or complete either of them. This PR stops creating those two stages on new work orders, hides them on existing ones, and makes sure a piece and its work order can still complete without them.

I did not run this against live data, because the scan and complete steps write to the database. See "Test plan".

## What changed

| File | Change |
|---|---|
| `src/modules/work-order/system-stage.ts` (new) | `isSystemStage(code, name)`. A stage is a system stage when its code is `HOLE` or its name is `DISPATCH`. Dispatch is matched by name because its code differs between companies (`DIS` in two companies, `STR` in another). |
| `src/modules/work-order/work-order.repository.ts` | When a work order is created, stage rows are no longer made for system stages. |
| `src/modules/production-scan/production-scan.repository.ts` | `lockItemProcessesInTx` also returns the work center code. New `skipProcessesInTx` marks stages as `SKIPPED`. |
| `src/modules/production-scan/production-scan.service.ts` | System stages are left out of the Production Line stage list, the pieces list (current stage and piece status) and the piece detail stages. Scanning and completing a stage only consider real stages. When the last real stage is completed, any system stages still open (older work orders) are marked skipped, and the piece is marked produced. |

## Why

Dispatch happens in the Dispatch module, so a piece should be finished once its production stages are done. If the screens had only hidden the Dispatch and HOLE rows, finishing the last real stage would have started a hidden Dispatch stage, and the work order would never reach COMPLETED.

## How

**New work orders**: the stage rows are created from the booking item's active work centers, minus the system stages.

**Existing work orders**: their Dispatch and HOLE rows stay in the database. They are filtered out wherever stages are shown, and ignored when choosing the current stage. When the last real stage is completed, the leftover system rows are set to `SKIPPED`, which the work order status check already counts as done, so the work order can complete as before.

**Nothing is deleted**: no rows were removed and there is no data migration.

## Decisions made

**Dispatch is matched by name, not by code.** A fixed list of codes was rejected earlier as hard-coded, and the Dispatch code is not the same in every company. The cost is that a work center renamed from "DISPATCH" would no longer be recognised.

**Old rows were hidden instead of migrated.** A one-off script to mark them `SKIPPED` would have changed production data without being asked, so older work orders are fixed gradually, as their last real stage is completed.

## Bugs found while testing (already there before, not fixed here)

**1. The Work Orders list still reads all stages.** The stage data used by the Work Orders list and the work order detail still includes Dispatch and HOLE for older work orders, because those queries were not changed.

**2. Deleting an approved PI is not blocked without a work order.** This is also described in the earlier backend PR.

## Notes

- A teammate's commit also changed `src/modules/order-booking/order-booking.repository.ts`. I did not write or review that change, and this document does not cover it.
- The backend needs a restart to pick up the changes.
- The frontend PR (`PR-frontend.md`) is the other half of this work. The screens assume Dispatch and HOLE are not stages.

## Next steps

- Test the older-work-order path: complete the last real stage on an existing piece and check that the piece and the work order become completed.
- Decide whether to run a one-off migration that marks old Dispatch and HOLE rows as skipped.
- Apply the same filter to the Work Orders list and detail queries.

## Test plan

**Automated tests (no test files for these modules):**
- [x] `npx tsc --noEmit -p .`: passed after the change.

**Manual checks (local):**
- [ ] Create and approve a work order, and check that no Dispatch or HOLE stage rows are made. Not checked, it writes to the database.
- [ ] Open an older piece and check that Dispatch and HOLE are not in the stepper or the stage counts. Not checked in a browser.
- [ ] Complete the last real stage of an older piece and check that the leftover rows become skipped and the work order completes. Not checked.
