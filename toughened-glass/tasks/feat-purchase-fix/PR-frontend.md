# 2026-11-28 - feat-purchase-fix (frontend)

## Summary

This PR updates the desktop and mobile procurement/purchase screens to enforce the Indent Request -> Approval -> Purchase Order -> Approval -> Purchase workflow: removes direct purchase creation and purchase cloning, makes Color optional in Indent Request, adds a Clone option for approved Indent Requests, locks carry-forwarded Purchase Order product fields in the Purchase form, removes the Purchase Order Change History button from Purchase Details, and hides Edit Payment from the Purchase row kebab menu until a payment is recorded.

## What changed

| File | Change |
|---|---|
| `src/features/inventory/new-purchase/new-purchase-page.tsx` | Removed the Purchase Type (`From Purchase Order` / `Direct`) mode selector, the Direct Purchase card, and the direct `PurchaseDialog` so new purchases only start from an approved Purchase Order. |
| `src/features/inventory/purchase-tab.tsx` | Removed the `Clone` row action (`cloningPurchase`) from the Purchase table kebab menu, and hid `Edit Payment` until `state.editPayment` is true (`paymentRecorded === true`). |
| `src/features/pwa/inventory/purchase-tab.tsx` | Removed the mobile `New purchase` FAB that linked to the direct purchase form (`/m/inventory/new-purchase`). |
| `src/features/inventory/components/purchase-dialog.tsx` | Disabled click-to-edit unlocking on `requestLocked` / `requestValue` so carry-forwarded fields (Product, Thickness, Glass Type, Color, Width, Height, Qty) stay read-only, hid `Add Row` when creating from a Purchase Order, and blocked deleting or removing Purchase Order rows. |
| `src/features/inventory/components/purchase-view-dialog.tsx` | Removed the `Change History` header button and `<PurchaseRequestHistoryDialog />` from Purchase Details. |
| `src/features/procurement/indent-dialog.tsx` | Removed the mandatory `*` indicator and required validation for `Color` on indent rows, and added `cloneFrom` support to prefill a new indent from an approved one. |
| `src/features/procurement/types.ts` | Updated `IndentInput` item `colorCode` type to `string | null | undefined`. |
| `src/features/procurement/procurement-tabs.tsx` | Added a `Clone` option to the Indent Request kebab menu for approved indents (`caps.indent.add && !state.pending`) and wired `cloningQuery` to `<IndentDialog />`. |

## Why

Purchases should only be created through the two-step approval workflow (Indent Request -> Purchase Order -> Purchase) without altering approved product specifications afterward. Additionally, Color is not always applicable on Indent Requests, approved Indent Requests needed a Clone action for repeat orders, and Edit Payment should only appear in the Purchase kebab menu once a payment has actually been recorded.

## How

**Remove direct purchase entry points**: `NewPurchasePage` now renders the Purchase Order selection table directly without a mode toggle. On the Purchase list, `Clone` was removed because cloning created a purchase without a `purchaseRequestId`, and on mobile PWA the `New purchase` FAB was removed.

**Lock Purchase Order details in Purchase**: In `PurchaseDialog`, `requestLocked` is always true for rows with `requestItemId`, and `requestValue` renders static text instead of a button that previously set `requestOpen: true`. `Add Row` is hidden when `allRequests.length > 0`, and `Delete` / `Remove row` are hidden for `row.requestItemId` rows. Since product details from the PO can no longer be changed, the `Change History` option was removed from `PurchaseViewDialog`.

**Show Edit Payment only after payment**: In `purchase-tab.tsx`, `Edit Payment` is now gated by `if (state.editPayment)` (which checks `caps.purchase.paymentEdit && purchase.paymentRecorded`) instead of always pushing a disabled item when `paymentRecorded` is false.

**Optional Color and Clone on Indent Request**: `IndentDialog` allows saving rows with an empty `colorCode` (sent as `null`). `IndentTab` adds `Clone` for non-pending indents (`APPROVED` or `PR_CREATED`), fetching the full indent via `useIndent(cloningId)` and opening `IndentDialog` with `cloneFrom` (resetting `requestDate` to today and clearing `requiredDate`).

## Test plan

**Automated tests:**
- [x] TypeScript build check (`npx tsc -b`): passed with exit code 0 across all frontend files.

**Manual checks (local):**
- [ ] Open New Purchase, confirm only Purchase Order selection is shown (no Direct mode), and verify Product/Thickness/Glass Type/Color/Width/Height/Qty are read-only in the Purchase dialog - not run in browser in this session.
- [ ] On Purchase list, verify `Edit Payment` is hidden before payment and visible after payment, and `Clone` is absent on all rows - not run in browser in this session.
- [ ] On Indent Request, verify saving without Color works and `Clone` appears only on approved indents - not run in browser in this session.
