# 2026-11-20 - fir-rework-replacement (backend)

## Summary

A FIR could only be a plain complaint. There was no way to fix the glass from it. This PR adds a second FIR kind, Rework / Replacement. Saving one creates Replace Orders (RO), one per resolution type. An RO is a work order with `order_type = 'RO'` that is worked on the shop floor, then dispatched through the normal dispatch and gate pass flow with no PI, no final invoice and no e-way bill. The FIR status follows its ROs on its own (OPEN, IN_PROGRESS, RESOLVED, CLOSED).

Each piece also carries its own configuration (size, glass, holes, cutouts, extra label). The faulty glass is saved in inventory as party material, and a rework takes its glass from that lot when its RO is approved.

Company fault records a fixed 50% penalty. Client fault records a typed charge. Both are records only, nothing is deducted or invoiced.

**The new SQL has not been run against a database.** Migrations 166 and 168 are not run yet, so every new query is untested live. This was a choice, the user runs migrations.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/166_fir_rework_replacement.sql` (new) | FIR kind, fault, penalty and charge columns. New `sales.fir_item` and `sales.fir_penalty_employee`. RO columns on `production.work_order`. Rework copies may share a barcode. |
| `src/modules/fir/fir-penalty.ts` (new) | Pure money rules: 50% penalty, piece amount from the final invoice, else the booking. |
| `src/modules/fir/fir-ro-status.ts` (new) | Moves the FIR status from the RO dispatch and gate pass state. |
| `src/modules/fir/fir-ro.repository.ts` (new) | Reads and writes FIR pieces, penalty employees and RO links. |
| `src/modules/fir/fir-ro.service.ts` (new) | Creates the FIR pieces and one RO per resolution type on save. |
| `src/modules/fir/fir.controller.ts`, `fir.routes.ts`, `fir.schema.ts`, `fir.service.ts`, `fir.repository.ts`, `fir.types.ts` | New FIR fields and inputs, routes for pieces and penalty employees, validation rules. |
| `src/modules/work-order/replace-order.repository.ts`, `replace-order.service.ts` (new) | Insert the RO as DRAFT, number it (RO prefix, `-R<n>` barcodes), copy the chosen pieces and stages. |
| `src/modules/work-order/replace-order-detail.repository.ts` (new) | Read one RO with its pieces and stages for the view and progress windows. |
| `src/modules/work-order/work-order.routes.ts`, `work-order.controller.ts` | New `GET /:id/replace-order`. |
| `src/modules/work-order/work-order.repository.ts`, `work-order.service.ts`, `work-order.types.ts`, `work-order-stock.service.ts`, `work-order-inventory.service.ts` | RO fields. A rework takes no stock. A replacement picks its glass lot at approval. |
| `src/modules/dispatch/dispatch.repository.ts`, `dispatch.service.ts`, `dispatch.types.ts` | RO shipments are ready to approve once all pieces finish all stages. RO progress. One newest-first list for pending PI and RO dispatches. Total Amount for RO rows. |
| `src/modules/dispatch/dispatch-progress.ts`, `dispatch-shipment.repository.ts`, `dispatch-shipment.service.ts` | RO handling in the progress and shipment steps. |
| `src/modules/production-scan/production-scan.repository.ts` | Scan flow works for RO pieces. |
| `src/modules/report/fir/*`, `src/modules/report/order-status/order-status-report.repository.ts`, `src/modules/report/sales-rep/sales-rep-report.repository.ts` | The FIR report knows the new kind. The other reports leave ROs out so they only count real work orders. |
| `src/database/migrations/168_fir_item_config_party_lot.sql` (new) | Piece configuration columns on `sales.fir_item` and the party lot link. `source_fir_id` on `public.inventory_lot`. |
| `src/modules/fir/fir-config.ts` (new) | Pure rules: hole work, the route of a piece, what a replacement is made as. |
| `src/modules/fir/fir-party-lot.ts` (new) | Creates the party material lots on save, issues a rework's sheets at approval, empties the lots on cancel. |
| `src/modules/fir/fir-ro.service.ts`, `fir-ro.repository.ts`, `fir.schema.ts`, `fir.types.ts` | Piece `config` in and out. Lots created on save. Cancel returns stock. |
| `src/modules/work-order/work-order-inventory.service.ts`, `work-order-stock.service.ts`, `replace-order.repository.ts` | A rework RO issues from its party lot, not stock. A replacement uses the typed glass and size. Rework pieces are no longer listed in the lot picker. |
| `src/modules/work-order/work-order.repository.ts`, `replace-order-detail.repository.ts`, `src/modules/production-scan/*` | An RO piece shows its own size and hole counts. |
| `tests/fir-piece-config.test.ts` (new) | Tests for the route, replacement spec, lot grouping and config input. |
| `tests/fir-rework-replacement.test.ts` (new) | Tests for penalty, numbering, status, readiness and input rules. |
| `tests/fir-route-rights.test.ts`, `tests/work-order-route-rights.test.ts` | Added the new routes to the rights lists. No other change. |

## Why

The business needed a clean way to fix faulty glass from a complaint, with the cost of the mistake recorded, without faking a new order, PI or invoice. Reusing the work order, scan and dispatch flow was easy. The real work was keeping the rework barcode, the readiness rule and the dispatch list correct for a shipment that has no PI.

## How

**FIR kind**: `fir_kind` is GENERAL (old behaviour, unchanged) or RO. Only RO FIRs have pieces, fault, penalty or charge.

**One RO per type**: on save, rework pieces go to one RO and replacement pieces to another. Each starts as DRAFT and points at the original work order and the FIR.

**Rework vs replacement**: a rework keeps the original barcode, runs only the chosen stages, and takes its glass from the FIR party lot. A replacement gets a `-R<n>` barcode, the full route, and its glass lot is chosen in the normal approve work order step.

**Piece configuration**: a rework keeps its glass and size and can ask for extra holes, cutouts and a label. A replacement can change size, glass type, thickness and colour. Any hole or cutout adds the Fabrication stage (found by name) after Grinding, else after Cutting, else first. Extra label is a flag only.

**Party material**: on save, pieces are grouped by glass, thickness, colour and size. Each group becomes one lot (one sheet per piece) that holds the FIR number and the reasons. A group is one lot, not one lot per FIR, because a lot can describe only one size. Pieces with no recorded size get no lot.

**Stock movement**: approving a rework RO takes one sheet per piece from its lot (an area movement, so rejecting the RO puts it back). Cancelling a FIR returns what its ROs took and sets its lots to zero.

**Ready to approve**: an RO shipment is ready when its pieces are live, not rejected, and every stage is completed or skipped. There is no invoice check.

**Dispatch list**: RO DRAFT shipments are fetched separately and merged with the PI pending rows, sorted by created time, newest first. They are left out of the paged query so none is listed twice.

**FIR status**: recalculated from the RO shipments and gate passes. It never goes backwards. Closing frees the pieces for a later FIR.

## Decisions made

**Penalty is a record only.** No split between employees and no salary deduction. The fixed 50% lives in one constant.

**Own progress for RO.** The PI progress did not fit an RO, so RO progress counts pieces with all stages done.

**Merged pending list reads at most 500 RO drafts.** Simple and enough for now. It is not a paged merge.

## Notes

- Migrations 166 and 168 are append-only and safe to re-run, but it was not run here.
- The Approve Dispatch dialog on an RO shipment may still ask for invoice or e-way bill fields. Not checked.
- A rework's last stage scan may add stock a second time. Not checked, flagged for review.

## Next steps

- Run migration 166 and walk the full flow once (FIR, approve RO, scan, dispatch, gate pass).
- Show the penalty and charge as columns in the FIR report.
- Check the duplicate stock risk on the last rework stage.

## Test plan

**Automated tests:**
- [x] `tests/fir-rework-replacement.test.ts`: penalty is 50% (1000 gives 500), piece amount from invoice then booking, RO and `-R<n>` numbering, FIR status from dispatch and gate pass, approval readiness, input rules.
- [x] `tests/fir-route-rights.test.ts` and `tests/work-order-route-rights.test.ts`: every new route has a right.
- [x] TypeScript check: clean.
- [ ] Full suite: 7 tests fail in purchase, PI reminder and list paging. They were failing before this work and are not touched here.

**Manual checks (local):**
- [ ] Not possible yet: nothing was run against a database because migration 166 is not run. Needs a full walk after the migration.
