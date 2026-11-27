# 2026-10-21 - pi-change-request

## Summary

Once a proforma invoice (PI) was approved, it could be edited freely, and the work order built from it could drift away from what the customer approved. This PR locks an approved PI and adds an edit approval flow. The team raises a change request with a reason and the changes needed. A manager approves or rejects it. Approving reopens the PI (back to pending approval) and its order booking(s) for one edit. After the booking is resubmitted and the PI is approved again (party and admin), the existing work order(s) are updated to match, and the change is saved in a change log with a before and after copy of the PI.

This work is on the `feat/edit-approval` branch in both repos (one commit each). It is documented here from the diff. **I did not run this feature myself**, so the test plan below only lists what the diff shows and what I could check by reading.

## What changed

### rubine-toughened-glass-frontend
| File | Change |
|---|---|
| `src/features/pi-status/change-requests.ts` (new) | Types for a change request and a change log entry. The calls to list, create, approve, reject and cancel requests, and to read a PI change log. Lists which roles can review (`MANAGER`, `ADMIN`, `SUPER_ADMIN`). |
| `src/features/pi-status/components/pi-change-dialogs.tsx` (new) | Three dialogs: **Request Change** (reason and the changes needed), **Change Requests** inbox for reviewers (approve or reject), and **Change History** (before and after of each applied change). |
| `src/features/pi-status/components/pi-status-table.tsx` | Row menu gets Request Change and View History. An approved PI shows its open request state. |
| `src/features/pi-status/pi-status-page.tsx` | **Change Requests** button with a pending count in the header. Opens the three dialogs. Opens a PI straight for editing when arriving with `editPi` in the URL. Shows how many work orders were updated after approval. |
| `src/features/pi-status/queries.ts`, `types.ts` | Message helper for updated work orders, and the extra response fields. |
| `src/routes/pi-status.tsx` | Accepts an `editPi` search value. |
| `src/features/order-booking/change-submit.ts` (new) | After a reopened booking is submitted, tells the user and sends them to that PI to update and approve it again. |
| `src/features/order-booking/add-product-modal.tsx`, `products-tab.tsx`, `review-tab.tsx` | Small changes to call the submit helper above. |

### rubine-toughened-glass-backend
| File | Change |
|---|---|
| `src/database/migrations/073_pi_change_request.sql` (new) | Tables `invoice.pi_change_request` (status PENDING, APPROVED, REJECTED, APPLIED, CANCELLED, with one open request per PI) and `invoice.pi_change_log` (before and after snapshots). |
| `src/database/migrations/074_pi_change_request_before_snapshot.sql` (new) | Adds `before_snapshot` to the request, saved when the request is approved. |
| `src/modules/proforma-invoice/pi-change-request.schema.ts` (new) | Validation for create, approve, reject (remarks required), list filter and id param. |
| `src/modules/proforma-invoice/pi-change-request.repository.ts` (new) | Queries for requests, the PI snapshot, reopening bookings, resetting the PI to pending and writing the log. |
| `src/modules/proforma-invoice/pi-change-request.service.ts` (new) | The rules: create, approve (reopens PI and bookings), reject, cancel, the locked-PI check on edit, and applying the change when the PI is approved again. |
| `src/modules/proforma-invoice/proforma-invoice.routes.ts`, `proforma-invoice.controller.ts` | New endpoints under `/proforma-invoices` for change requests and the change log. |
| `src/modules/proforma-invoice/proforma-invoice.service.ts`, `proforma-invoice.types.ts` | When a PI is approved again, applies the open change request and returns what changed. |
| `src/modules/work-order/work-order-sync.service.ts` (new) | Updates the existing work order(s) to match the edited booking: rebuilds, adds or removes pieces and stages, and returns the stock when needed. Skips finished work. |
| `src/modules/work-order/work-order.repository.ts` | Piece creation moved into a reusable `createPiecesForItemInTx` so the sync can use it too. |
| `src/modules/work-order/work-order.service.ts` | `syncWorkOrdersFromBookingInTx` refreshes party and project on live work orders. |
| `src/modules/work-order/work-order-inventory.service.ts` | New `force` option for issuing stock again after a rebuild. |
| `src/modules/order-booking/order-booking.controller.ts` | The submit response says which PI needs approving again when the booking was reopened by a change request. |
| `src/modules/production-scan/production-scan.service.ts`, `production-scan.repository.ts`, `production-scan.types.ts` | Piece numbers shown as "3 / 5" now count live pieces by position, so gaps from added or removed pieces do not show. |

## Why

An approved PI is a commitment to the customer. The team still needs to correct it sometimes, but a change should be asked for, approved by a manager, and recorded. Keeping the work order in step with the edited booking stops production from building the old version.

## How

**Lock**: saving an edit on a `CONFIRMED` PI is refused with `PROFORMA_INVOICE_LOCKED` until a request is approved.

**Request**: `POST /proforma-invoices/:id/change-requests` with a reason and the changes needed. Only a `CONFIRMED` PI needs one. Only one open request (pending or approved) per PI is allowed.

**Review**: `PATCH /proforma-invoices/change-requests/:requestId/approve` and `/reject` need the role `MANAGER`, `ADMIN` or `SUPER_ADMIN`. Rejecting needs remarks. Nobody can review their own request, except `ADMIN` and `SUPER_ADMIN`. The requester or a reviewer can cancel a pending request.

**Approve reopens**: approving saves a before snapshot of the PI, reopens the order booking(s), and sets the PI back to pending approval.

**Re-approval applies**: when the PI is approved again, the bookings must be submitted first. Then the existing work orders are synced, the after snapshot is saved, the change is logged with the request number, and the request becomes `APPLIED`.

**Follow-through in the UI**: submitting a reopened booking returns the PI code, and the screen sends the user to that PI to review, save and approve it again.

## Decisions made

**The work order is updated, not recreated.** On re-approval the existing work order(s) are synced to the booking, so barcodes, scans and history stay attached where the pieces still exist.

**Self-review is blocked for managers but allowed for admins.** A manager cannot approve their own request. Admin and super admin can, so a small team is not stuck.

**One edit per approval.** Approval unlocks the PI once. Approving the PI again closes the request.

## Bugs found while testing (already there before, not fixed here)

**1. The review roles do not include the sales or production manager.** The review endpoints use the exact role names `MANAGER`, `ADMIN` and `SUPER_ADMIN`. The database also has `SALES_MANAGER` and `PRODUCTION_MANAGER`, which cannot approve or reject. This is how the diff is written, not something this review changed. If those two should review, the role list needs them.

## Notes

- Written from the committed diff of `feat/edit-approval` against the common base. Not run by me.
- The dispatch delivery date request feature (separate PR file) is a different flow and shares no code with this one.
- Migration numbers 073 and 074 are used by both this branch and the dispatch date request branch (different file names, so both can exist, but they will run in filename order on a merged database).

## Next steps

- Run the full flow once against a real approved PI with a work order that has started production.
- Decide whether the sales and production managers should be able to review.

## Test plan

**Automated tests (no test framework in either repo):**
- [ ] Backend and frontend type check and lint on `feat/edit-approval`: not run by me.

**Manual checks (local):**
- [ ] Editing an approved PI is refused until a request is approved.
- [ ] A non-reviewer role cannot approve or reject. A requester cannot review their own request unless admin.
- [ ] Approve reopens the PI and the booking(s). Submitting the booking sends the user to the PI.
- [ ] Approving the PI again updates the existing work order(s), writes a change log entry with before and after, and marks the request applied.
- [ ] Piece numbers on the scan screen stay continuous after pieces were added or removed.
- [ ] None of the above were run by me. The diff was read only.
