# 2026-11-02 - without-material-job-work (frontend)

## Summary

The screens had no way to enter the glass a customer brings, to see it apart from Rubine's stock, or to return what is left. This PR adds a Customer Glass Details section to the product form, a locked "Customer Stock" allocation on the Approve Work Order screen, a Customer Material tab in Inventory with a Mark Returned action, and the customer's balance glass on the gate pass. It also hides the Material charges everywhere in the order booking flow for Without Material products, and fixes the Create Proforma Invoice page, which turned a material rate of 0 into a hard coded 1450.

`tsc --noEmit -p tsconfig.app.json` passes. I did not open the screens in a browser myself. The layouts were checked from the user's screenshots while building.

**Hiding the Material label on the PI status drawer, proforma sheet and PDF, and final invoice screens is not included on purpose** - see "Next steps" below. The amounts there are already 0, this was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/order-booking/schema.ts` | Form fields `rawSheetWidth`, `rawSheetHeight`, `rawSheetCount`. When Job Type is Without Material all three must be above 0. |
| `src/features/order-booking/api.ts` | Item types carry the three fields. `itemBody` sends them for Without Material and sends null otherwise. |
| `src/features/order-booking/product-detail-panel.tsx` | New "Customer Glass Details" block (icon and title like the other section titles, three ordinary inputs) when Job Type is Without Material. The Rate (per SQM) field is set to 0 and disabled for that job type, and cleared again when switched back. Most of this file's line count is reformatting by the editor on save, the real change is the block above. |
| `src/features/order-booking/add-product-modal.tsx` | Default and loaded values for the raw sheet. Passes the raw sheet in mm to the size panel. |
| `src/features/order-booking/size-details-panel.tsx` | When the tick on a size row is clicked, the size is checked against the raw sheet. A size that does not fit is not added, and one red line shows to the left of the Add Row button: "Size 1000 x 1000 mm does not fit the raw sheet 10 x 10 mm." |
| `src/features/order-booking/work-order-stock-api.ts` | `customerSupplied` flag on a stock candidate group. |
| `src/features/order-booking/work-order-stock-section.tsx` | A customer glass row shows plain text "Customer Stock . <lot code>" with the normal colour chip, and the Select stock button is disabled. Group keys include the customer flag so it cannot clash with a normal group. Most of this file's line count is editor reformatting. |
| `src/features/order-booking/format.ts` | `isWithoutMaterialJob` and `hideOrderMaterialCharges` helpers. |
| `src/features/order-booking/order-details-modal.tsx` | Material Charges row hidden per product, and the order total line hidden when every product is Without Material. |
| `src/features/order-booking/booking-preview-modal.tsx` | Same as above. |
| `src/features/order-booking/order-booking-pdf.ts` | Same as above for the printed order booking. |
| `src/features/order-booking/products-tab.tsx` | Order total Material line hidden when every product is Without Material. |
| `src/features/order-booking/review-tab.tsx` | Basic Amount tile hidden when every product is Without Material. |
| `src/features/pwa/order-booking/mobile-order-detail.tsx` | Same hiding on the mobile order detail. |
| `src/features/invoices/create-proforma-invoice.tsx` | A booked rate of 0 is kept as 0. Before, a falsy rate became the placeholder 1450. When every product is Without Material, the Base Rate and Material columns, the footer cells, the Total Material card and the Total Material row are hidden. |
| `src/features/final-invoices/components/create-final-invoice-panel.tsx` | The share of order level charges uses item count when the PI items have no glass value, the same as the server. |
| `src/features/final-invoices/api.ts` | `piItemCount` on the eligible items response. |
| `src/features/dispatch/types.ts` | `CustomerBalanceGlass` type and the field on the gate pass record. |
| `src/features/dispatch/api.ts` | Maps `customer_balance_glass` to `customerBalanceGlass`. |
| `src/features/dispatch/components/gate-pass-sheet.tsx` | New section "Customer's Own Glass (Job Work)" with Finished Toughened Glass and Customer Balance Glass Return. Shown only when the invoice has customer glass. |
| `src/features/inventory/customer-material.ts` (new) | Types and hooks for the customer material list, the storage location change and Mark Returned. |
| `src/features/inventory/customer-material-tab.tsx` (new) | The Customer Material tab. KPI cards, filter tabs (On Hand, Returned, Empty, All), search, a table with the shared `DataTable` and pager, a location picker, a clickable lot code that opens a details dialog, a Booking No badge that opens the order booking details, and a Mark Returned button with a confirm dialog. The Returned tab hides the Stored At and Action columns. |
| `src/routes/inventory.tsx` | New "Customer Material" tab. |
| `package-lock.json` | no real change. npm dropped some `libc` lines on a local install. Not part of this work, do not commit it. |

## Why

Without Material orders need the raw sheet size to cut from, a clear view of the customer's glass, and a billing screen that does not charge for glass the customer owns. The proforma page problem was found in testing: a PI for a Without Material order showed Material 58.00 and a total of 212.00 because the page treated a rate of 0 as missing and filled in 1450.

## How

**Customer Glass Details**: Three number fields (width mm, height mm, sheets) in the same grid as the other fields. They only appear for Without Material. Saving the product creates the customer lot on the server.

**Size check**: The check runs only when the tick on a size row is clicked, converts the row to mm with the row's unit, and tries both orientations. The message sits next to Add Row so it is one line and does not squeeze into a narrow input.

**Hidden material charges**: The amounts are already 0 from the server, these changes only hide the rows. At order level the row is hidden only when every product is Without Material, so a mixed order still shows the material total for its With Material products.

**Customer Material tab**: Built from the same pieces as the Storage Locations tab (`KpiCard`, `FilterToolbar`, `FilterTabs`, `SearchBar`, `DataTable`, `ViewDetailsDialog`, `ConfirmActionModal`, `CodeLinkButton`). Rows load once and are filtered and paged in the browser.

**Mark Returned**: Opens a confirm dialog that names the customer and lists exactly what is being handed back, and warns that it cannot be undone. The row then moves to the Returned tab.

## Decisions made

**The "exact pre-cut sizes" button was removed.** It was in the plan and was built first, then removed on request. Staff now type the raw sheet themselves. A warning that compared total area with the sheets provided was also removed.

**The size check blocks, the area check does not exist.** A piece larger than the sheet can never be cut, so it is not added. A total area check would be only approximate because of cutting waste, so it was dropped.

**The 1450 placeholder is removed only from the rate calculation.** Two other fallbacks of 1450 in the same file (in branches that build lines from the booking or enquiry list) were left alone, they only run when no rate exists at all.

## Notes

- The Rate field is disabled and set to 0 for Without Material. If it still shows a number, hard refresh, because the dev server may be serving an older build.
- Existing Without Material products keep their old rate until the product is opened and saved once.
- The three screens that were not changed (PI status drawer, proforma sheet and PDF, final invoice screens) still show their glass line, now at 0.00.

## Next steps

- Hide the Material label, columns and totals on the PI status drawer, proforma sheet and PDF, and final invoice screens for Without Material orders. They use the basic amount as the invoice total, so this needs care so GST and totals do not change.
- Add a Customer Material view to the mobile app.
- Let Mark Returned save the dispatch or gate pass number, receiver and a remark, and add a reopen option.

## Test plan

**Automated tests (none run):**
- [x] `npx tsc --noEmit -p tsconfig.app.json`: passes with no errors
- [ ] No frontend test files cover these screens, none were added

**Manual checks (local):**
- [ ] Product form: choose Without Material and see Customer Glass Details, Rate 0 and disabled. Switch back and see the block disappear: not run by me. The user's screenshots showed the block and the disabled Rate field
- [ ] Add a size larger than the raw sheet and click the tick, see the one line message left of Add Row and no row added: not run by me, the user's screenshot showed the message under the field before it was moved
- [ ] Approve Work Order screen shows "Customer Stock . <lot>" with the colour chip: not run by me. The user's screenshot showed the earlier pill version
- [ ] Customer Material tab: filters, Booking No badge, lot details, location change, Mark Returned and the Returned tab: not run. Needs migrations 112 and 113 and the new backend
- [ ] Order booking, preview, PDF, mobile detail and Create Proforma Invoice show no Material rows for a Without Material order, and still show them for a With Material order: not run
- [ ] Create Proforma Invoice no longer shows 58.00 material for a Without Material order: not run after the fix
- [ ] Gate pass sheet shows the Customer's Own Glass section for a Without Material dispatch only: not run
