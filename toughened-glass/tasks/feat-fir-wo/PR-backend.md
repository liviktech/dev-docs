# 2026-11-24 - fir-ro-fixes (backend)

## Summary

Work orders created from a FIR (Replace Orders) were formatted as `RO-<number>` because the generator sliced off the `WO` prefix from the original work order. They now format properly as `RO-WO-<number>`. In addition, when approving a rework work order, the raw glass row was missing from the approval dialog because rework items were explicitly excluded from the stock candidates query. This PR includes rework pieces in the stock candidates endpoint, attaches their party material lot, and flags them as rework so the approval dialog can display them in a locked state.

## What changed

| File | Change |
|---|---|
| `src/modules/fir/fir-penalty.ts` | Updated `replaceOrderCode` to keep `WO` in the Replace Order code, producing `RO-WO-<number>` while keeping `-2`, `-3` counter suffixes for repeat complaints. |
| `src/modules/fir/fir-penalty.test.ts` (new) | Unit tests verifying `replaceOrderCode` across hyphenated, slashed, unhyphenated and plain number formats. |
| `src/modules/work-order/work-order-inventory.service.ts` | Added `is_rework` and `rework_party_lot_id` to `WorkOrderItemSpec` and `Group`. Joined `sales.fir_item rw` in `loadItemSpecs`, and updated `buildGroups` to group rework pieces under party material. |
| `src/modules/work-order/work-order-stock.service.ts` | Removed the exclusion of rework pieces in `loadGroups`. Updated candidate query in `getStockCandidates` to match the FIR party material lot directly, return `isRework` and `reworkLotCode`, and avoid false low stock flags. |

## Why

Two issues were reported:
1. Work orders created from FIR should have code like `RO-WO-<format number>`, but they only had `RO-<format number>`.
2. When approving a rework work order, the item correctly fetches from the party material created for it on save, but it did not show up in the approval dialog.

## How

**RO Code Format**: In `replaceOrderCode`, the logic previously tested `/^WO/i` and ran `RO${sourceCode.slice(2)}`, which dropped `WO`. It now checks the pattern and prefixes `RO-` to the existing code (or inserts `RO-WO-` if missing), keeping the base intact.

**Rework Pieces in Stock Candidates**: In `loadGroups`, the condition `AND NOT EXISTS (... rw.resolution_type = 'REWORK')` was removed. `loadItemSpecs` now inspects `sales.fir_item rw` to pick up `rework_party_lot_id` and mark `is_rework = true`. `buildGroups` treats rework items as customer/party material.

**FIR Party Lot Match**: In `getStockCandidates`, the query now accepts the specific rework party lot ID as an optional parameter (`$7`) to match the exact inventory lot created by the FIR. It marks the returned group with `isRework: true` and `reworkLotCode`.

## Decisions made

**Preserve repeat counter suffixes.** Repeat ROs for the same work order continue to use `-2`, `-3` (e.g. `RO-WO-20261119-177-3`), ensuring database code uniqueness without changing how past complaints were recorded.

**Match party lot by ID over general party code.** FIR creates an exact party lot for rework pieces on save. Passing `reworkPartyLotId` into the candidate query guarantees that the exact lot created for the piece is returned first.

## Test plan

**Automated tests (all passing):**
- [x] `src/modules/fir/fir-penalty.test.ts`: formats `WO-0042` to `RO-WO-0042` (with `-2`, `-3` suffixes), `WO/2026/001` to `RO-WO/2026/001`, and unhyphenated codes.
- [x] TypeScript check (`npx tsc --noEmit`): clean, zero errors.

**Manual checks (local):**
- [x] Ran database queries verifying existing records for work order `20261119-177`, confirming that repeat Replace Orders increment suffixes correctly (`RO-20261119-177`, `RO-20261119-177-2`, `RO-WO-20261119-177-3`).
