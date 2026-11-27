# 2026-11-28 - feat-item-flow-order-booking (frontend)

## Summary

This branch implements the frontend **Item Flow & Order Booking** updates: Purchase Products master screen and picker, decoupled Item / Indent / Purchase forms (matching raw items by glass details), redesigned Product Details & Rates/Charges tabs with layer and rate history support, Order Booking stock-shortage warning/blocking dialog with "Indent Request" prefill, product search & count-based charges, and hiding internal `code` fields across master screens (including Party, Supplier, and Storage Locations) with name-click view modals.

## What changed (major item-flow changes)

| Area / File | Change |
|---|---|
| `src/routes/admin.purchase-products.tsx`, `src/features/stock-masters/config.ts`, `src/features/products/queries.ts` | Added Purchase Products master route and `usePurchaseProductChoices` / `useStockProductLookup` hooks. |
| `src/features/products/product-dialog.tsx`, `product-details-tab.tsx`, `rates-tab.tsx`, `rate-form-dialog.tsx`, `rate-utils.ts`, `charges-footer.tsx` | Rebuilt Product dialog with separate Details and Rates tabs, layer support, rate-charge calculation types, and dated rate/charge history. |
| `src/features/items/item-dialog.tsx`, `item-view-modal.tsx`, `items-page.tsx`, `src/features/inventory/purchase-item-match.ts` | Updated Item Master UI for decoupled raw/finished items and glass-detail matching (`glassTypeCode`, `colorCode`, `thickness`). |
| `src/features/procurement/indent-dialog.tsx`, `purchase-request-dialog.tsx`, `src/features/inventory/components/purchase-dialog.tsx`, `use-previous-purchase-fill.ts`, `src/features/pwa/inventory/mobile-purchase-form.tsx` | Switched Indent and Purchase product pickers to Purchase Products (`usePurchaseProductChoices`), added previous purchase line auto-fill and per-line tax amount handling. |
| `src/features/order-booking/stock-warning-dialog.tsx`, `stock-warning.ts`, `stock-shortage.ts`, `work-order-stock-section.tsx`, `src/routes/work-orders.tsx`, `src/routes/inventory.tsx` | Added `StockWarningDialog` for `ORDER_BOOKING_STOCK_WARNING` / `ORDER_BOOKING_STOCK_BLOCKED` and updated all "Indent Request" shortage shortcuts (`toIndentPrefill`, `toPurchasePrefill`, `WorkOrderStockSection`) to prefill glass details without passing the order's sales `productCode` (with optional `productCode` in `PurchasePrefill` and `parsePurchaseItems`). |
| `src/features/order-booking/header-detail-tab.tsx`, `product-detail-panel.tsx`, `product-search-bar.tsx`, `product-search.ts`, `charges-table.tsx`, `order-details-modal.tsx` | Added product search bar, updated order-booking product/charge handling, and added direct PI support. |
| `src/routes/party.tsx`, `src/features/parties/party-dialog.tsx`, `src/features/stock-masters/config.ts`, `stock-master-page.tsx`, `src/features/inventory/locations-tab.tsx`, `inventory-location-dialog.tsx`, `src/features/pwa/inventory/locations-tab.tsx` (+ other master screens) | Removed/hidden internal `code` columns, form inputs, and view-modal code badges on Party, Supplier, and Storage Locations (and other master screens), and wired clicking the record's Name (`CodeLinkButton`) to open the view modal. |

## Why

- **Align UI with decoupled item flow**: Indents and Purchases select from Purchase Products instead of Sales Products, and Items match by glass details.
- **Fix shortage-to-indent prefill**: Passing the order's sales `productCode` into `IndentDialog` caused `INDENT_PRODUCT_NOT_FOUND` on submit; leaving `productCode` unset lets the user pick a valid Purchase Product while keeping glass type, thickness, colour, and sheet size prefilled.
- **Cleaner master screens**: Internal codes are auto-generated on the server and used in backend print/documents, so hiding them from tables/forms/view modals simplifies data entry and opens details on name click.

## How

- **`usePurchaseProductChoices`**: Offers active Purchase Products on Indent and Purchase forms while still displaying legacy sales product codes on existing lines.
- **Optional `PurchasePrefill.productCode`**: `parsePurchaseItems` in `src/routes/inventory.tsx` keeps shortage prefills that have glass/sheet details without a `productCode`, and `IndentDialog` initializes `productCode` to `""`.
- **Code-hidden master pattern**: Tables use `CodeLinkButton` on the Name column to open the view modal; add/edit forms omit the code input.

## Notes

- Pairs with the `feat/item-flow-order-booking` backend PR.

## Test plan

- [x] Type check (`tsc --noEmit` on `rubine-toughened-glass-frontend/tsconfig.json`): passed (exit code 0).
- [ ] Manual check: open "Indent Request" from Order Booking stock warning, Work Order approval shortage, and Work Order stock section; verify glass details prefill and a Purchase Product can be selected and saved.
- [ ] Manual check: verify Party, Supplier, and Storage Locations hide codes in table/form/view and open the view modal on name click.
