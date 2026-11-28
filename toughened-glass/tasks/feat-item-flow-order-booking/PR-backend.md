# 2026-11-28 - feat-item-flow-order-booking (backend)

## Summary

This branch implements the new **Item Flow & Order Booking** architecture across master data, procurement, inventory, and order booking. Sales products (`is_purchase_product = FALSE`) and purchase products (`is_purchase_product = TRUE`) are separated while sharing `master.product`, raw material items are matched by glass details (`glass_type_code`, `color_code`, `thickness_mm`) rather than product, product rates support layers, dated rate/charge history, and calculation types, and order booking adds a stock-shortage check (`ORDER_BOOKING_STOCK_WARNING` / `ORDER_BOOKING_STOCK_BLOCKED`), direct PI creation, and count-based charges. It also fixes finished-good item matching for stock on purchase products and adds auto-generated codes for inventory locations and other masters.

## What changed (major item-flow changes)

| Area / File | Change |
|---|---|
| `src/database/migrations/194_product_type_master.sql`, `194_procurement_line_product.sql`, `195_purchase_product.sql`, `196_procurement_line_glass_details.sql` | Added Product Type master, Purchase Products (`master.product.is_purchase_product`), nullable `item_code` on purchase products and procurement lines, and glass detail columns (`glass_type_code`, `color_code`, `thickness_mm`) on indent and purchase request lines. |
| `src/database/migrations/195_product_rate_charges_history.sql`, `196_order_booking_direct_pi.sql`, `197_product_rate_layer.sql`, `197_purchase_line_tax_amount.sql`, `198_rate_charge_calculation_types.sql`, `199_stock_code_purchase_product_order.sql` | Added dated product rate/charge history, `layer` on product rates, rate-charge calculation types, `tax_amount` on purchase lines, direct PI support on order booking, and `{purchase product}-{thickness}-{glass type}-{colour}` stock code ordering. |
| `src/modules/master/stock-option/stock-option.config.ts`, `stock-option.routes.ts`, `master.routes.ts`, `product/product.repository.ts`, `product.service.ts`, `product-rate-history.ts` | Added `/master/purchase-products` and `/master/product-types`, separated sales vs. purchase product queries (`findActivePurchaseProductCodes`), and added product rate history & layer support. |
| `src/modules/master/item/item.repository.ts`, `item.service.ts`, `src/modules/inventory-purchase/purchase-item-match.ts` | Updated item matching (`pickItemForLine`, `itemFitsLine`) to match active `RAW_MATERIAL` items by glass type, colour, and thickness (with `product_code` only as a tie-breaker). |
| `src/modules/procurement/procurement.service.ts`, `procurement.repository.ts`, `src/modules/inventory-purchase/inventory-purchase.service.ts` | Validated indent and purchase lines against active Purchase Products (`findActivePurchaseProductCodes`), made `item_code` optional on procurement lines, and added `tax_amount` and previous purchase line retrieval. |
| `src/modules/order-booking/order-booking-stock-check.ts`, `order-booking-glass.repository.ts`, `order-booking.service.ts`, `count-charges.ts` | Added order-booking stock shortage check (`assertStockAllows` throwing `ORDER_BOOKING_STOCK_WARNING` / `ORDER_BOOKING_STOCK_BLOCKED`), product price lines endpoint, direct PI flow, and count-based charges. |
| `src/modules/work-order/work-order-inventory.service.ts` | Updated `recordFinishedItemInTx` to match `RAW_MATERIAL` items to the issued lot by `glass_type_code`, `color_code`, and `thickness_mm` via a deterministic `JOIN LATERAL ... LIMIT 1` (preferring `raw.product_code = l.product_code`, then `raw.product_code = woi.product_code`), so stock on purchase products still records `finished_item_code`. |
| `src/modules/master/inventory-location/inventory-location.schema.ts`, `inventory-location.repository.ts`, `inventory-location.service.ts` (+ other master schemas/services) | Made `code` optional on create and auto-generated unique codes from `name` (supporting code-hidden master UIs). |

## Why

- **Decouple buying from selling**: Procurement (Indents, Purchase Requests, Purchases) and raw stock lots use **Purchase Products** and glass specs (`glass_type_code`, `color_code`, `thickness_mm`), while Order Booking and Sales use **Sales Products**.
- **Fix finished-goods recording on purchase-product stock**: Matching `RAW_MATERIAL` items strictly by `raw.product_code = l.product_code` failed whenever a lot was stocked under a Purchase Product.
- **Prevent stock surprises at Order Booking**: Check available sheets/cutoffs against order pieces at submit/save and return structured shortage details.

## How

- **Purchase Products on `master.product`**: `is_purchase_product` distinguishes purchase products from sales products while keeping foreign keys on `inventory_lot`, `indent_request_item`, and `purchase_request_item` intact.
- **Glass-detail item matching**: `pickItemForLine` and `recordFinishedItemInTx` match `RAW_MATERIAL` items on `(glass_type_code, color_code, thickness_mm)` and use `product_code` only for tie-breaking.
- **Stock check on Order Booking**: `findBookingStockShortages` groups order glass by `(thickness, glass_type_code, color_code)` (`mergeSameGlass`) and runs `allocatePieces` against candidate lots before allowing submission.

## Notes

- Includes migrations `194_*` through `199_*` in `src/database/migrations/`.
- Pairs with the `feat/item-flow-order-booking` frontend PR.

## Test plan

- [x] Type check (`tsc --noEmit` on `rubine-toughened-glass-backend/tsconfig.json`): passed (exit code 0).
- [ ] Run backend unit/integration test suite (`order-booking-stock-check.test.ts` and full `tests/`).
- [ ] Verify migrations `194`–`199` on a clean/staging database.
