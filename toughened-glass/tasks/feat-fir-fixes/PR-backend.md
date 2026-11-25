# 2026-11-25 - fir-workflow-fixes (backend)

## Summary

The FIR, Replace Order and Dispatch flow had several problems. A FIR could be raised on a piece that had not left the factory yet. A replacement in a new size kept the old piece's area. Replacement specs were not checked against the masters. Two people could approve the same Replace Order at once. A Complaint FIR's status could jump anywhere from the edit form. A Rework/Replacement FIR became Resolved on dispatch approval and Closed on gate pass, but nothing else ever used Closed. A Resolved FIR could still be edited and deleted. A cancelled Replace Order dispatch left its pieces with no way to ship. This PR fixes all of these on the server. It also keeps Replace Orders off Final Invoices and off the PI's pending dispatch, and fixes a broken search on the Archive list. Old FIR rows and old Replace Order codes are left exactly as they are. Everything was checked with the automated tests and the type check only, not live in the app.

**Some known FIR issues are not included on purpose** (E-Way Bill for Replace Order shipments, Invoice / PI on a Rework FIR, client charge billing, delay measured from today, FIR stuck when every Replace Order piece is rejected, delete not done in one step). The user asked to leave them for later. See "Next steps" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/modules/fir/fir-ro.repository.ts` | New `PIECE_DISPATCHED_SQL`: a piece counts as dispatched only when it is live on a CONFIRMED dispatch. The FIR work order list now only shows work orders with at least one such piece. Piece lists and the piece lock return `dispatched`. New `findUnknownReplacementSpec` checks product, thickness, colour and glass type against the active master records. |
| `src/modules/fir/fir-ro.service.ts` | Refuses a piece that is not dispatched (`FIR_PIECE_NOT_DISPATCHED`). Refuses a changed replacement whose spec is not in the masters (`FIR_REPLACEMENT_SPEC_INVALID`). Passes the new size to the Replace Order only when width or height really changed. |
| `src/modules/work-order/replace-order.repository.ts` | A replacement in a new size now gets its chargeable area worked out from the new size (plus the booking's charge-per allowance), and its `glass_code` is cleared so it is not tied to the old size. |
| `src/modules/work-order/work-order.repository.ts` | New `lockWorkOrderStatusInTx` (`SELECT status ... FOR UPDATE`). `approveWorkOrderInTx` no longer updates a work order that is already approved, in progress, completed or cancelled. |
| `src/modules/work-order/work-order.service.ts` | `approveWorkOrder` and `recordWorkOrderApproval` lock the row and check the status again inside the transaction, so a second approval at the same moment is refused and glass is issued only once. |
| `src/modules/fir/fir.schema.ts` | Processing type removed from create and update. Create always starts at OPEN. Update takes every form field as optional and no status. New `moveFirStatusSchema` and `FIR_STATUS_MOVES` (Open to In Progress, In Progress to Resolved). |
| `src/modules/fir/fir.repository.ts` | New `updateFirRow` that saves every editable field, only while the FIR is OPEN or IN_PROGRESS. New `moveGeneralFirStatusRow` that moves a Complaint FIR one step with a conditional UPDATE. |
| `src/modules/fir/fir.service.ts` | `updateFir` saves every field (a field left out keeps its saved value), refuses a Resolved FIR (`FIR_NOT_EDITABLE`), keeps the customer, invoice, PI and work order of a Rework/Replacement FIR, and allows a charge edit only on a client-fault one. New `moveFirStatus`. `deleteFir` refuses a Resolved FIR (`FIR_NOT_DELETABLE`). |
| `src/modules/fir/fir.controller.ts`, `src/modules/fir/fir.routes.ts` | New `POST /firs/:id/status`, behind `FIR_EDIT`. |
| `src/modules/fir/fir-penalty.ts` | `deriveRoFirStatus` now gives RESOLVED once every Replace Order has a gate pass, otherwise IN_PROGRESS. Closed is no longer produced. |
| `src/modules/fir/fir-ro-status.ts` | A Rework/Replacement FIR becomes Resolved on gate pass (not on dispatch approval) and its pieces are freed then. Resolved and old Closed rows are never moved again. |
| `src/modules/fir/fir.types.ts` | `FIR_STATUSES` is now Open, In Progress, Resolved. Closed stays only in `FIR_PROCESSED_STATUSES` so old rows still read correctly. |
| `src/modules/dispatch/dispatch-shipment.repository.ts` | New `reopenBlockersInTx` and `reopenShipmentInTx` for reopening a cancelled Replace Order dispatch. `loadPieceFacts` now only counts the order's own work orders, so Replace Order pieces no longer show on the PI's pending dispatch. |
| `src/modules/dispatch/dispatch-shipment.service.ts` | New `reopenShipment`: only for a cancelled Replace Order dispatch, refused with the reasons listed when the Replace Order or FIR is no longer live or a piece is on another live shipment. Puts it back to DRAFT and adds any finished piece it was missing. |
| `src/modules/dispatch/dispatch.controller.ts`, `src/modules/dispatch/dispatch.routes.ts` | New `POST /dispatches/:id/reopen`, same rights as cancel. |
| `src/modules/dispatch/dispatch.repository.ts`, `src/modules/dispatch/dispatch.types.ts` | The dispatch list now returns `source_fir_id`, so the screen can show the FIR number. |
| `src/modules/dispatch/dispatch.service.ts` | Comments only, to match the new "Resolved on gate pass" rule. No real change. |
| `src/modules/final-invoice/final-invoice.repository.ts`, `src/modules/final-invoice/final-invoice.service.ts` | A Replace Order can no longer be put on a Final Invoice (`FINAL_INVOICE_REPLACE_ORDER`), and it is left out of the work orders offered for invoicing. |
| `src/modules/archive/archive.service.ts` | The search and date filters used `${n}` instead of `$${n}`, so the SQL had a bare number in place of a parameter. Fixed. |
| `src/database/migrations/184_dispatch_reopen.sql` (new) | Adds `reopened_at` and `reopened_by` to `invoice.dispatch`. Safe to run again. Not run yet. |
| `tests/fir-processing-type.test.ts` | Rewritten for the new status rules (create starts Open, update has no status, the allowed status moves). |
| `tests/fir-rework-replacement.test.ts` | Replace Order code tests now expect `RO-WO-...` with a count suffix, and the status tests expect Resolved on gate pass. |
| `tests/fir-route-rights.test.ts` | Adds `POST /:id/status`. |

## Why

These are problems found in the FIR flow while testing it, and the decisions the user made while reviewing them. The status rules were the real work. The user decided that a Complaint FIR moves only by a menu action (Open to In Progress to Resolved), that the Closed tab goes away, and that a Rework/Replacement FIR is Resolved only once the gate pass is issued. Once Resolved, a FIR is final and cannot be edited or deleted. The dispatch reopen exists because a Replace Order has exactly one dispatch (`uq_dispatch_source_work_order`), so a cancel by mistake used to leave the pieces and the FIR stuck for good.

## How

**Dispatched means a confirmed dispatch**: one SQL fragment decides it everywhere (work order list, piece list, piece lock), so the list, the picker and the save all agree. A piece on a draft or postponed dispatch is not dispatched yet.

**New size, new area**: the replacement insert works out `actual_chargeable_area` as (width + allowance) x (height + allowance) / 1,000,000 x quantity, using the same charge-per allowance as the original booking. Rework and same-size replacements keep the original values as before.

**Master check on replacement specs**: only a changed replacement is checked. A value equal to the piece's own original is always accepted, so a piece booked before a master record was renamed or retired still works.

**Double approval**: the work order row is locked with `FOR UPDATE` before the status is read, and the UPDATE also refuses a work order that is already past approval. The second request waits, then sees the new status and stops.

**Status moves**: a Complaint FIR moves with `POST /firs/:id/status`, and the UPDATE only goes through when the FIR is still in the expected step, so two clicks cannot skip a step. A Rework/Replacement FIR refuses this route (`FIR_STATUS_FOLLOWS_RO`) because its status follows the gate pass.

**Editing**: every field can be edited while the FIR is Open or In Progress. On a Rework/Replacement FIR the customer, invoice, PI and work order stay as they were, because the Replace Orders were built from them.

**Reopen a cancelled Replace Order dispatch**: it goes back to DRAFT and its good pieces (not rejected or denied) come back. The FIR stays In Progress and moves on as usual at gate pass.

**Replace Order code format kept as it is**: the code is `RO-WO-<date>-<sequence>`, with `-2`, `-3` and so on when more FIRs come from the same order. The function signature did not change.

## Decisions made

**No change to old data.** The user asked that old rows stay as they were. A data-only migration that rewrote old Replace Order codes into the new format was written, then deleted. Old codes keep their old format, and old FIRs that read Closed still show as Closed.

**Closed is kept as a legacy value, not removed.** New FIRs never reach Closed, but the value stays readable so older rows do not break.

**Resolved is the end.** A Resolved FIR cannot be edited or deleted. The server refuses it, not only the screen.

**The FIR charge and its billing were left alone.** The user said their TL decides how the client-fault charge is collected, so nothing was added for it.

## Notes

- Migration `184_dispatch_reopen.sql` must be run before the reopen route is used. It was not run here, as the project rule is that the user runs migrations.
- The Archive search fix is a small side fix found while working here. Before it, searching or filtering by date on the Archive list sent broken SQL.

## Next steps

- decide whether Replace Order shipments need an E-Way Bill or delivery challan.
- save the invoice and PI picked on a Rework/Replacement FIR, or hide those pickers for it.
- decide how a client-fault charge is billed (TL to decide).
- measure the 48-hour delay from the FIR date instead of today.
- give a way out when every piece of a Replace Order is rejected, so the FIR and the original piece are not stuck.
- run the Replace Order cancel and the FIR delete in one transaction.

## Test plan

**Automated tests (206 passing):**
- [x] `tests/fir-processing-type.test.ts`: create always starts Open, update takes no status, only Open to In Progress and In Progress to Resolved are allowed.
- [x] `tests/fir-rework-replacement.test.ts`: Replace Order code is `RO-WO-...` (count 2 gives `-2`, count 3 gives `-3`), and a Rework/Replacement FIR is In Progress until every Replace Order has a gate pass, then Resolved.
- [x] `tests/fir-route-rights.test.ts`: every FIR route, including the new `POST /:id/status`, needs the right permission.
- [x] `tests/fir-piece-config.test.ts`, `tests/dispatch-*.test.ts`, `tests/final-invoice-*.test.ts`, `tests/work-order-route-rights.test.ts`: still pass.
- [x] Type check (`tsc --noEmit`): no errors.

**Manual checks (local):**
- [ ] Raise a FIR on a dispatched piece and try one that is not dispatched: not checked live, covered only by the type check and the shared SQL fragment.
- [ ] Two approvals of the same Replace Order at the same moment: not checked live, hard to reproduce by hand.
- [ ] Reopen a cancelled Replace Order dispatch: not checked live, needs migration 184 to be run first.
- [ ] Gate pass on a Replace Order moves its FIR to Resolved: not checked live, covered by the status tests.
