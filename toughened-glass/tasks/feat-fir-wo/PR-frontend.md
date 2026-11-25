# 2026-11-24 - fir-ro-fixes (frontend)

## Summary

When approving a rework work order, the raw glass row was not visible in the approval dialog. With the backend now providing rework groups in `stock-candidates`, this PR updates the frontend to display the rework glass row in the Work Order Approval dialog with a "Party Material (Rework)" badge, displays the party lot code, and presents the stock selection button in a locked, non-clickable state.

## What changed

| File | Change |
|---|---|
| `src/features/order-booking/work-order-stock-api.ts` | Added `isRework?: boolean` and `reworkLotCode?: string | null` to the `StockCandidateGroup` interface. |
| `src/features/order-booking/work-order-stock-section.tsx` | Added `Lock` icon import. Included rework distinction in `groupKey`. Updated `allPicked` to treat rework groups as satisfied. Rendered "Party Material (Rework)" badge with lot code in the allocation column, and a disabled "Locked" button in the stock column. |

## Why

Rework work orders take their glass from the customer material lot created on FIR submission. Previously, the raw glass row did not show up in the approval dialog, leaving the user with no visual confirmation of the allocated glass. The user requested that the item be displayed and locked so that it cannot be clicked or manually changed.

## How

**Rework Group Identification**: The `StockCandidateGroup` received from the API carries `isRework: true` and the lot code of the party material created during FIR creation. `groupKey` appends `|rework` to avoid merging with normal party stock.

**Locked Button State**: Instead of the interactive "Select stock" button, rework rows render a disabled button displaying a lock icon and "Locked" label, with `cursor-not-allowed` styling, `disabled={true}`, and a tooltip explaining that party material is locked for rework.

**Approval Readiness**: In `WorkOrderStockSection`, `allPicked` now checks `Boolean(g.isRework) || Object.values(picked[groupKey(g)] ?? {}).some(...)`, allowing the dialog's Approve button to activate without requiring manual stock picking for rework items.

## Decisions made

**Render a dedicated Locked button instead of hiding the action.** Displaying a styled disabled button with a lock icon makes it immediately obvious to the shop-floor manager that the piece already has its glass assigned and that no manual picking is needed or permitted.

## Test plan

**Automated tests (all passing):**
- [x] TypeScript check (`npx tsc --noEmit`): clean, zero errors.

**Manual checks (local):**
- [x] Verified that `work-order-stock-section.tsx` renders the disabled button with `disabled={true}` and no `onClick` handler when `isRework` is true.
- [x] Verified that `allPicked` resolves to true for rework groups so the Approve button in `WorkOrderEditDialog` is enabled.
