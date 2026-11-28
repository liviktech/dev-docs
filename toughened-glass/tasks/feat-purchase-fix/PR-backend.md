# 2026-11-28 - feat-purchase-fix (backend)

## Summary

This PR tightens the procurement-to-purchase flow in the backend so every new purchase must come from an approved purchase order (no direct purchases), carry-forwarded product details from the purchase order cannot be altered when saving a purchase, and the Color field on Indent Requests is optional. Existing pending or legacy purchases that do not have a `purchase_request_id` continue to work for viewing, editing, and adding to stock.

## What changed

| File | Change |
|---|---|
| `src/modules/inventory-purchase/inventory-purchase.schema.ts` | Added a schema refinement on `createInventoryPurchaseSchema` requiring `purchaseRequestId` (or `purchaseRequestIds`) when creating a purchase. |
| `src/modules/inventory-purchase/inventory-purchase.service.ts` | Enforced in `createPurchase` that a purchase order id is present, and rejected any save where `computePurchaseRequestChanges` detects changes to carry-forwarded purchase order lines instead of logging modification history. |
| `src/shared/constants/error-messages.ts` | Added `purchaseOrderIsRequired` and `purchaseOrderDetailsCannotBeModified` under `ERROR.PURCHASE`. |
| `src/modules/procurement/procurement.schema.ts` | Made `colorCode` optional and nullable on `indentItemSchema` (normalizing empty strings to `null`) and updated the duplicate-line key in `assertDatesAndUniqueItems` to handle missing colors. |
| `src/modules/procurement/procurement.repository.ts` | Updated `replaceIndentItems` to accept optional/nullable `colorCode` and store `null` when omitted. |
| `src/modules/master/item/item.repository.ts` | Updated `findUnknownGlassDetail` to validate `colorCode` against `master.glass_color` only when a color is provided. |
| `src/modules/inventory-purchase/purchase-item-match.ts` | Updated `MatchableLine.colorCode` to accept `string | null`. |
| `tests/inventory-purchase-create.test.ts` | Updated the purchase test fixture with a `purchaseRequestId` and added a test verifying that direct purchases without `purchaseRequestId` are rejected. |

## Why

The team follows a two-approval procurement flow (Indent Request -> Approval -> Purchase Order -> Approval -> Purchase). Previously, the API allowed creating purchases without a purchase order (direct purchase), allowed modifying product details and quantities that had already been approved on the purchase order, and required a color on every indent request line even when not applicable.

## How

**Require Purchase Order on purchase creation**: `createInventoryPurchaseSchema` now validates that either `purchaseRequestId` or `purchaseRequestIds` is provided, and `createPurchase` in `inventory-purchase.service.ts` also checks `prIds.length === 0` and throws `BadRequestError` (`PURCHASE_REQUEST_REQUIRED`). `updateInventoryPurchaseSchema` is left unchanged so existing pending purchases continue to work.

**Lock carry-forwarded Purchase Order lines**: In `createPurchase`, `computePurchaseRequestChanges` still compares the submitted items against the purchase order's lines on the server. When any difference (or added/removed line) is detected, `createPurchase` now throws `BadRequestError` (`PURCHASE_REQUEST_MODIFICATION_NOT_ALLOWED`) instead of saving with `modificationRemarks`.

**Optional Indent Color**: `indentItemSchema.colorCode` transforms blank strings to `null` and allows `null` or `undefined`. `findUnknownGlassDetail` skips the `master.glass_color` lookup when `colorCode` is empty, and `inventory.indent_request_item.color_code` is already nullable in PostgreSQL.

## Test plan

**Automated tests (13 passing):**
- [x] `tests/inventory-purchase-create.test.ts`: 13/13 tests passed via `node --import tsx --test tests/inventory-purchase-create.test.ts`, including the new test rejecting direct purchase payloads without `purchaseRequestId`.
- [x] TypeScript check (`npx tsc --noEmit`): passed with exit code 0.

**Manual checks (local):**
- [ ] Live API requests against the local server for creating an indent without color, creating a purchase without a purchase order, and attempting to modify PO line details on purchase creation - not run live via HTTP in this session; covered by schema/service code inspection and unit tests.
