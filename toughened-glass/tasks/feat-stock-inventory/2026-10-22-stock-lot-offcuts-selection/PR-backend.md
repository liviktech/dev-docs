# 2026-10-22 - stock-lot-offcuts-selection (backend)

## Summary

Stock was tracked per lot only as a total area and a sheet count. When a Work Order was approved, the glass was taken from the lots with no memory of the leftover pieces, no choice of which lot to use, and no way to pick the glass type. Movements could also be created and edited by hand, and editing a lot could reset its available stock without any movement. This PR adds leftover pieces (offcuts) per lot, per-piece stock issue at Work Order approval, optional colour and glass type matching, manual lot selection for a Work Order, a read-only movement history, and a lot edit that records a movement when the quantity changes. The Adjust Stock endpoint stays in the API but the UI no longer uses it.

**Database-backed tests are not included on purpose** - see "Test plan" below. They are written as pending (todo) cases, not skipped silently.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/079_inventory_offcuts.sql` (new) | Adds the `inventory_lot_offcut` table (width, height, quantity, status, source Work Order). Adds `glass_type_code` and `legacy_unshaped_sqm` to `inventory_lot`. Old purchase lots get the purchase's glass type. Append only, no existing column is changed or removed. |
| `src/database/opening-balance-correction.ts` (new) | One-time script. Dry run by default, `--apply` writes. For each lot whose movements do not add up to its available stock, it writes one "Opening balance correction" ADJUSTMENT for the difference. The lot's own quantities are not changed. |
| `src/modules/inventory/cutoff-engine.ts` (new) | The pure logic that decides which sheet or offcut each piece is cut from and what leftover pieces come out of it. |
| `src/modules/work-order/work-order-stock.service.ts` (new) | Stock candidates for a Work Order, saved colour and glass type choices, and saved manual lot selections. |
| `src/modules/inventory/inventory.service.ts` | Lot create and edit now handle glass type, offcuts and legacy area. Lot edit no longer resets available stock. When the received quantity or sheets change, the available stock moves by the same difference and one ADJUSTMENT movement ("Received quantity edited") is written. Lowering the quantity below what was already used is rejected. Purchase linked lots can now have their quantity edited (before it was locked). Sheet size, product, thickness, brand, colour, supplier and glass type stay locked on a purchase lot, and a lot with history keeps its size and product locked. Added lot history and a ledger report. |
| `src/modules/inventory/inventory.repository.ts` | Queries for offcuts, glass type, lot history, grouped movements with pagination, and the ledger check. |
| `src/modules/inventory/inventory.controller.ts`, `inventory.routes.ts`, `inventory.schema.ts`, `inventory.types.ts` | New `GET /lots/:id/history` and `GET /ledger-report`. The manual `POST /movements` and `PATCH /movements/:id` routes and their schemas are removed, so movements are read only. `POST /lots/:id/adjust` is kept. Types and schemas updated for glass type and offcuts. |
| `src/modules/work-order/work-order-inventory.service.ts` | Rewritten approval and reject stock flow. Stock is issued per piece with one ISSUE movement per piece (with the Work Order item id). Pieces are cut from offcuts first, then sheets, and the leftovers are saved as offcuts. Honours saved manual selections. Matches on product, thickness, glass type, brand and colour. Low stock errors name the piece size. A rejected piece that was broken stays consumed. |
| `src/modules/work-order/work-order.controller.ts`, `work-order.routes.ts`, `work-order.schema.ts` | New routes `GET /:id/stock-candidates`, `GET` and `PUT /:id/material-choice`, `GET` and `PUT /:id/stock-selection`, with APPROVE permission for the writes and VIEW for the reads. |
| `src/modules/work-order/work-order.service.ts` | Small change to pass the new stock options into approval. |
| `src/modules/inventory-purchase/inventory-purchase.repository.ts`, `inventory-purchase.service.ts` | Purchase now carries the glass type onto the lot it creates. |
| `tests/inventory-cutoff.test.ts` (new) | Unit tests for the cut-off engine, plus pending database-backed cases. |
| `package.json` | Added the `test` script (`tsx --test tests/*.test.ts`). |
| `package-lock.json` | Lock file update only, no real change. |

## Why

The plan is in `info/docs/stock-lot-cutoff-and-lot-selection-plan.md`. The real work was the per-piece issue and the offcut bookkeeping. The glass type column, routes and schemas were the easy part. Making movements read only and making the lot edit write its own movement means the stock total can be checked against the movement history, which is what the ledger report and the opening balance script rely on.

## How

**Stock totals**: `quantity_available_sqm` stays the total (full sheets, offcuts and `legacy_unshaped_sqm`). `quantity_available_pieces` now means full sheets only. Old lots that cannot be tied to sheets or offcuts keep their area in `legacy_unshaped_sqm` and show as "Other (size unknown)".

**Cutting**: for each piece the engine looks at the offcuts of the chosen lot first, then a full sheet. The leftover is stored as new offcut rows. The same size from two sheets is one row with quantity 2. A lot with no sheet size is issued by area only and makes no offcuts.

**Manual selection**: if the Work Order has a saved stock selection, approval takes exactly from those lots. If not, it picks automatically. Both approve paths use the same code.

**Lot edit**: the lot row is locked in a transaction. The difference in received sqm and sheets is applied to the available stock and one ADJUSTMENT movement is written. Edits that change nothing in quantity write no movement.

**Kept the same on purpose**: the Adjust Stock endpoint (`/lots/:id/adjust`), its schema and its rules (reason required, no negative stock) are untouched. Only the UI stopped using it.

## Decisions made

**Purchase lot quantity is now editable.** Before, the quantity of a purchase linked lot was locked and had to be fixed with Adjust Stock. The Adjust Stock button was removed from the UI, so the lot edit now allows the change and writes the movement. Everything else on a purchase lot stays locked. This does not change the purchase record itself.

**Movements cannot be created or edited by hand.** The two routes were removed instead of being hidden, so no client can bypass the history.

**The Adjust Stock endpoint was not deleted.** It is no longer called by the app, but removing it is a separate cleanup and could affect other clients.

## Notes

- Migration 079 and the opening balance script have not been run on the shared dev database. Run `npm run migrate` first, then `npx tsx src/database/opening-balance-correction.ts` and read the dry run before using `--apply`.
- Restart the backend after pulling, unless it runs in watch mode.
- Nothing was tested through the running app or against a database in this session.

## Next steps

- Write the pending database-backed tests in `tests/inventory-cutoff.test.ts` once a test database is available.
- Decide whether to remove the unused `/lots/:id/adjust` endpoint.

## Test plan

**Automated tests (13 passing, 19 pending):**
- [x] `tests/inventory-cutoff.test.ts`: cut-off engine cases, 13 passing.
- [ ] `tests/inventory-cutoff.test.ts`: 19 database-backed cases are marked todo (two orders approved at the same time, edit received quantity writes an ADJUSTMENT, below used is rejected, glass type and colour matching, movements grouping and pagination, and others). They are not run because they need a database.
- [x] `npx tsc --noEmit`: no errors.

**Manual checks (local):**
- [ ] Run migration 079 on a dev database - not done, it is the shared Neon dev database and needs approval first.
- [ ] Approve a Work Order with and without a manual selection, then reject a piece - not done, not checked against a running backend.
- [ ] Edit the received quantity of a purchase lot and see the ADJUSTMENT in the history - not done, covered only by the pending test and the type check.
