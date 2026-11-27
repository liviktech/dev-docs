# 2026-11-26 - dispatch-eway-fir-fixes (backend)

## Summary

An audit of the Dispatch flow found gaps that the screens hid but the server allowed: a PI could be confirmed without Admin approval, a PI with shipped goods could be cancelled, one invoice could be split over several trucks although it has one e-Way Bill, and two requests at once could create two bills or approve a dispatch twice. This PR closes these on the server, corrects the e-Way Bill ship-to address and old-invoice HSN lookup, removes e-Way Bill validity extension, and fixes the open FIR follow-ups (date rules, 48-hour delay, delete in one step, rejected Replace Order pieces). Most dispatch changes were checked live on the local app, the FIR date, delete and rejected-piece rules were not (see the test plan).

**Shipping to credit customers before payment is not included on purpose** - a shipment still needs its invoice fully paid. Whether credit customers may be dispatched earlier is a business decision. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/modules/proforma-invoice/proforma-invoice.repository.ts` | The generic status change can only cancel a pending PI or move a cancelled one back to pending, and no longer writes approval stamps. Approval and decline only work on a pending PI. Auto-cancel skips a PI unlocked by an approved change request, and returns the company with each PI code. |
| `src/modules/proforma-invoice/proforma-invoice.service.ts` | A bare "confirm" without Party or Admin approval is refused (`PROFORMA_INVOICE_APPROVAL_REQUIRED`). A change the current status does not allow is refused (`PROFORMA_INVOICE_INVALID_STATUS_CHANGE`). Cancel, decline and auto-cancel check for shipped goods and close the dispatch side in the same transaction. |
| `src/modules/proforma-invoice/pi-dispatch-close.ts` (new) | Refuses to cancel or decline a PI that has an approved shipment (`PROFORMA_INVOICE_HAS_SHIPMENT`), and cancels its unapproved shipments. |
| `src/modules/dispatch/dispatch-progress.ts` | New `ON_HOLD` and `CLOSED` states worked out from the PI status. New whole-invoice rules (`splitInvoiceReasons`, `shippablePieces`). |
| `src/modules/dispatch/dispatch-shipment.repository.ts` | New PI row lock, list of approved shipments of a PI, and cancel of its draft shipments. Removed `ensureMissingPiDispatches` and `assignMissingParentCodes`, so opening the list no longer writes anything. |
| `src/modules/dispatch/dispatch-shipment.service.ts` | Creating and approving a shipment respect the PI hold and the whole-invoice rule. A change request now shows the pending dispatch as On Hold. Cancelled and declined PIs can be listed as Closed. The shipment detail returns `deliveryAddress` from the PI. |
| `src/modules/dispatch/dispatch.repository.ts` | The table returns `hold`, a held or closed row is never ready to approve, the "ready" check includes the whole-invoice rule, and approval only updates a DRAFT shipment. |
| `src/modules/dispatch/dispatch.service.ts` | A second approval is refused (`DISPATCH_ALREADY_APPROVED`). Approval locks the PI before the shipment. Closed dispatches are listed under Cancelled with matching counts and paging. |
| `src/modules/dispatch/dispatch.schema.ts`, `src/modules/dispatch/dispatch.types.ts` | New `ON_HOLD` and `CLOSED` values, `hold` and `deliveryAddress` fields. |
| `src/modules/e-way-bill/e-way-bill.service.ts` | Only one generation per invoice can run at a time (`EWAYBILL_IN_PROGRESS`). A bill needs an approved dispatch (`EWAYBILL_DISPATCH_NOT_APPROVED`). Extend validity removed. |
| `src/modules/e-way-bill/e-way-bill.repository.ts` | New advisory lock per invoice and approved-dispatch check. Removed the validity update. |
| `src/modules/e-way-bill/e-way-bill.controller.ts`, `e-way-bill.routes.ts`, `e-way-bill.schema.ts` | Removed `POST /extend-validity` and its schema. |
| `src/shared/gst/gst-provider.types.ts`, `mock-gst.provider.ts`, `sandbox-gst.provider.ts`, `gst-api-log.repository.ts` | Removed the extend validity provider call and log operation. |
| `src/shared/gst/gst-invoice-source.repository.ts` | New `findOrderDeliveryAddress` (PI consignee address, else the order's). A line with no saved HSN now takes it from sales finished goods matching product, thickness and colour. |
| `src/shared/gst/gst-invoice-snapshot.ts` | Ship-to uses the order's delivery address first, then the party's. Items with different HSN codes stop the document (`AMBIGUOUS_ITEM_HSN`). |
| `src/shared/gst/line-tax-resolver.ts` | New `resolveFallbackHsn`. |
| `src/modules/fir/fir-ro.service.ts` | New `assertFirDateAllowed`: the FIR date cannot be in the future or before the dispatch date (invoice date when nothing is dispatched). The Replace Order cancel now runs inside the caller's transaction (`cancelRoFirWorkInTx`). |
| `src/modules/fir/fir.service.ts` | Create and update apply the date rule. The 48-hour delay is measured up to the FIR date. Delete locks the FIR, checks its status again, and cancels the Replace Orders and the FIR in one transaction. |
| `src/modules/fir/fir.repository.ts` | The dispatch date is now also found when the dispatch carries the invoice through its pieces (first approved one). The delay clock stops at the end of the FIR date, or now if sooner. New row lock, and the delete runs inside the transaction. |
| `src/modules/fir/fir.schema.ts`, `src/modules/fir/fir.types.ts` | The reference dates query accepts `firDate`. |
| `src/modules/work-order/replace-order.service.ts`, `src/modules/work-order/work-order.service.ts` | A piece rejected on a Replace Order leaves its waiting shipment at once, so the shipment is not blocked and the remake can join. |
| `src/modules/dispatch/dispatch-hold.test.ts` (new) | Test for hold, closed and release from the PI status. |
| `src/modules/dispatch/dispatch-whole-invoice.test.ts` (new) | Tests for the whole-invoice rule. |
| `src/shared/gst/line-tax-resolver.test.ts` (new) | Tests for the HSN fallback. |
| `tests/dispatch-shipment-schemas.test.ts` | Removed the extend validity test. |

## Why

The dispatch changes come from an end to end audit of the Dispatch, e-Way Bill and PI flow. The screens already blocked most of these, so the real work was making the server refuse the same things, including when two requests arrive together. The FIR changes finish items left in the "Next steps" of the `feat-fir-fixes` PR, plus the FIR dispatch date, which showed empty for dispatches made from pieces and wrongly asked for a delay reason.

## How

**Hold is worked out, never stored**: a PI in DRAFT (change request) puts its dispatch On Hold, a cancelled or declined PI shows it as Closed. Approving the PI again brings the same pending dispatch back with its current quantities, so no second dispatch is created.

**One lock order**: the PI row is locked before the shipment row, both on approval and on cancel or decline. A shipment cannot be approved while its PI is being cancelled.

**Status guards in SQL**: approval, decline and the generic status change filter on the current status in the UPDATE itself, so a stale screen or a second request changes nothing and gets a clear error.

**An invoice ships whole**: all its pieces still to ship must be in the same shipment, because it has one e-Way Bill listing all its goods. Rejected pieces and pieces already on an approved shipment do not count. One dispatch can still carry several invoices, each with its own bill.

**One bill at a time**: a database advisory lock per invoice covers the "already exists" check and the portal call, across server instances.

**Reads do not write**: the dispatch list no longer creates missing pending dispatches or numbers them. Admin approval already creates them.

## Decisions made

**Extend validity was removed, not hidden.** The user asked for the whole option to go (screen, route, service and provider call). No database change was needed.

**Unapproved shipments are cancelled when a PI is cancelled or declined, but the pending dispatch is kept.** It shows as Closed under the Cancelled filter and is not counted under All or Pending. Old-flow shipments are only held, not cancelled.

**Completed PIs stay hidden.** A PI in CLOSED status was completed, so its pending dispatch has nothing left to show.

**Different HSN codes are never guessed between.** The e-Way Bill or e-Invoice stops and names the codes, so the items can be corrected.

## Bugs found while testing (already there before, not fixed here)

**1. Nine tests in `tests/` fail.** Dashboard route rights, new purchase schema, list paging schemas, PI reminder slabs and runner, and sales visit route rights. The same nine fail on a clean copy of the last commit.

**2. The server does not force Party approval before Admin approval.** The order is only enforced by the screen.

**3. A confirmed PI cannot be cancelled or declined directly.** Only a pending one can, so a confirmed PI must go through a change request first.

**4. The e-Way Bill does not handle delivery to another state.** It sends the same value for the to state and the actual to state with transaction type 1. None of the current local PIs deliver to another state.

**5. The rate of an already invoiced product can still change after a change request.**

**6. A bill created on the portal but not saved on our side is not recorded.** This affects any user if saving fails right after the portal call.

## Notes

- No migration is added by this PR.
- Draft shipments made earlier that carry only part of an invoice can no longer be approved. Cancel them and create one shipment with the whole invoice.
- A PI without a pending dispatch is no longer repaired by opening the list. On the local database there were none, and no pending dispatch without a number. Worth running the same check on other databases.
- The e-Way Bill lock holds one database connection while the generation uses others. About ten generations at the same moment could use up the connection pool.
- The GST provider on the local setup is the mock one, so nothing was checked against the real portal.
- A delay reason already saved on a FIR is not cleared by the dispatch date fix.

## Next steps

- check that Party approval is done before Admin approval on the server.
- handle delivery to another state on the e-Way Bill.
- test e-Way Bill generation against the real GST provider.
- add tests for the FIR date rule, the one-step delete and the rejected Replace Order piece.
- fix the nine old failing tests.

## Test plan

**Automated tests (new and related tests passing, 9 old failures in the full folder):**
- [x] `src/modules/dispatch/dispatch-hold.test.ts`: a PI's status holds or closes its dispatch, and approval releases it.
- [x] `src/modules/dispatch/dispatch-whole-invoice.test.ts`: whole invoices can be selected, part of one is refused, rejected or already approved pieces are not needed, an unapproved shipment is named, an invoice with one piece not ready offers none.
- [x] `src/shared/gst/line-tax-resolver.test.ts`: colour picks the HSN, different codes are reported, one shared code is used, no matching item falls back to the product.
- [x] `npm test`: 28 passing, including the three files above.
- [x] `tests/dispatch-*.test.ts` and `tests/fir-*.test.ts`: 90 passing.
- [ ] `tests/*.test.ts` (whole folder): 1166 passing, 9 failing, 19 todo. The 9 failures are the old ones listed above.
- [x] Type check (`tsc --noEmit`): no errors.

**Manual checks (local):**
- [x] Sent direct requests for an invalid confirm, reopen, decline and approve again: all refused.
- [ ] Cancel or decline of a PI that has an approved shipment: not recorded as checked live, covered only by the type check.
- [x] Tried to ship part of an invoice: refused.
- [x] Fully shipped PIs closed automatically (checked for new-flow dispatches only).
- [x] Sent two e-Way Bill requests together: one bill was created. Sent two dispatch approvals together: one went through.
- [x] Put a PI through a change request: the dispatch was held and the same one came back after approval.
- [x] Chose a different delivery address on a PI: e-Way Bill, gate pass and packing slip used it.
- [x] Tried an e-Way Bill before dispatch approval: refused. A dispatch with two invoices went from 0 to 1 to 2 of 2 bills.
- [x] HSN conflict on an old-style invoice line: checked on real rows inside a transaction that was rolled back.
- [x] Opened the Cancelled filter after declining a PI: its dispatch showed as Closed and was absent from Pending and All.
- [x] FIR on `WO-20261027-164`: dispatch date 26/11 now found and no longer marked delayed. `FIR-20261124-211` shows 19/11.
- [ ] FIR date in the future or before dispatch, FIR delete with Replace Orders, rejected piece on a Replace Order: not checked live and no dedicated test, covered only by the type check.
- [ ] Real GST portal: not checked, local setup uses the mock provider.
