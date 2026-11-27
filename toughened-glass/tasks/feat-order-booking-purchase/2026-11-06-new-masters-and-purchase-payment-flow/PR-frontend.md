# 2026-11-06 - new-masters-and-purchase-payment-flow (frontend)

## Summary

Two main pieces of work, plus a set of smaller fixes, all in the same branch pair.

1. **Four new Master Data screens: Layer, Port, Destination and Logo.** The Add / Edit Product form in Order Booking had Layer, Port, Destination and Logo as a free number box, two free text boxes and a fixed list of four logos. They are now searchable dropdowns fed by these masters, with a **+ Add** (Quick Add) at the end of each. The new masters start empty.
2. **Add to Stock and Payment are now two separate purchase actions.** Before, Add to Stock asked for the payment mode and bank accounts and saved them together with the stock transfer. Now **Add to Stock** shows only General Details and Product Details, and a new **Payment** action in the kebab menu records the bank and payment details. Either can be done first, and Payment can be done days after the stock transfer.
3. **Smaller fixes** across Purchase, Order Booking, Work Order, Production Plan, Proforma / Final Invoice, FIR and Master Data screens (listed in the tables below).

The author checked all of these changes in the browser. Both projects compile with no type errors. No automated tests were added (see "Test plan").

**Seed data for the four new masters is not included on purpose** - see "Decisions made" below. The records are added by hand later. This was a choice, not a miss.

## What changed

**New masters (Layer, Port, Destination, Logo)**

| File | Change |
|---|---|
| `src/features/stock-masters/config.ts` | Four new definitions (`LAYER_DEF`, `PORT_DEF`, `DESTINATION_DEF`, `LOGO_DEF`) with endpoint, icon, labels and maximum name length. Layer is flagged as whole-number-only |
| `src/features/stock-masters/queries.ts` | Hooks for the four masters, and their lists in the central lookup |
| `src/features/stock-masters/stock-master-dialog.tsx` | Layer name must be a whole number from 1 to 999 (Port, Destination, Logo unchanged) |
| `src/lib/lookup.ts` | `layers`, `ports`, `destinations`, `logos` added to the central lookup type, and to the list of master paths that refresh it after a write |
| `src/routes/admin.layers.tsx`, `admin.ports.tsx`, `admin.destinations.tsx`, `admin.logos.tsx` (new) | The four screens, built on the same shared master page as Suppliers and Glass Colors (KPI cards, Active / Inactive / All tabs, search, paging, view, add, edit, activate / deactivate) |
| `src/routeTree.gen.ts` | Regenerated for the new routes (generated file) |
| `src/components/layout/nav-items.ts` | Layers, Ports, Destinations and Logos added under Product Catalog |
| `src/features/auth/accessible-routes.ts` | The four routes added to the ordered list used to pick a user's first page |
| `src/features/order-booking/master-select-field.tsx` (new) | One dropdown field for the product form: active records, a "None" choice, a Quick Add item for users with the Master Data add right, and a value not in the master stays shown |
| `src/features/order-booking/product-detail-panel.tsx` | Layer, Port and Destination now use the new field; new `onQuickAdd` prop |
| `src/features/order-booking/remarks-others-panel.tsx` | Logo uses the new field; the old fixed list is commented out |
| `src/features/order-booking/add-product-modal.tsx` | Opens the master's Add dialog over the product form, selects the new record when it is saved, and keeps the form values if it is cancelled |

**New purchase flow (Add to Stock and Payment)**

| File | Change |
|---|---|
| `src/features/inventory/components/add-to-stock-dialog.tsx` | Now General Details and Product Details only, no payment fields or payment requests. Still exports the shared account card and payment mode labels other screens use |
| `src/features/inventory/components/purchase-payment-dialog.tsx` (new) | The Payment modal: General Details and Product Details for reference, then Bank & Payment Details. The fields, checks and submit logic were moved here from the old Add to Stock modal |
| `src/features/inventory/components/purchase-reference-cards.tsx` (new) | The read-only General Details and Product Details cards shared by both modals |
| `src/features/inventory/purchase-tab.tsx` | New **Payment** menu item (disabled only once payment is saved); Add to Stock follows stock status only; Edit and Delete disabled once payment is recorded |
| `src/features/inventory/purchase-api.ts`, `purchase-queries.ts`, `purchase-types.ts` | Add to Stock sends nothing; new calls and hooks for the payment options and for saving a payment; new `paymentRecorded` flag |
| `src/features/inventory/components/purchase-view-dialog.tsx` | The old "Payment History" card is now **Supplier Bill** (the row behind it is the invoice record, not a payment); Paid / Unpaid label added next to the stock label; wording changed to "Payment recorded" |
| `src/features/inventory/components/purchase-dialog.tsx` | **+ Add Item** on the Glass Type, Color and Thickness dropdowns of the products table (opens the Add Item modal); supplier invoice number required when editing as well; Actions column kept fixed while the other columns scroll |
| `src/features/inventory/components/purchase-sheet.tsx` | Invoice No and Invoice Date removed from the preview; Remarks moved up into the header box |

**Smaller fixes**

| File | Change |
|---|---|
| `src/features/parties/party-dialog.tsx`, `src/features/stock-masters/stock-master-dialog.tsx` | Address Line 2 hidden (commented out); Address Line 1 renamed to "Address" |
| `src/features/order-booking/add-product-modal.tsx` | The charges loader covers the whole Charges section again, not just the table |
| `src/features/pi-status/components/proforma-invoice-sheet.tsx`, `proforma-pdf-data.ts`, `src/lib/pdf/proforma-invoice-pdf.ts` | Job type (in Pascal case) added to the product heading |
| `src/features/fir/fir-page.tsx`, `src/features/inventory/purchase-tab.tsx` | KPI cards no longer change with the search and filters |
| `src/features/shifts/shift-select.tsx` | Shift options show formatted times instead of raw hours |
| `src/features/order-booking/work-order-sheet.tsx`, `work-order-sheet-view.tsx` | Logo and its note shown as two separate cells |
| `src/features/order-booking/work-order-summary-modal.tsx`, `api.ts` | New **Lot No** column in the View Work Order modal |
| `src/components/ui/form-dialog.tsx`, `src/features/users/user-dialog.tsx` | Edit dialogs use a **Save** button |
| `src/features/production-plan/production-plan-modal.tsx` | Only the content scrolls; Cancel and Next stay fixed |
| `src/features/production-plan/production-plan-editor.tsx` | Submit stays disabled until the required fields are filled, with the reason as a tooltip |
| `src/features/e-invoice/components/einvoice-tax-invoice-sheet.tsx` | "Party W/o No." removed from the final invoice preview |
| `src/features/products/product-dialog.tsx` | Deleting a routing, rate or charge row asks for confirmation (shared confirm dialog) |

## Why

- **New masters:** Layer, Port, Destination and Logo were typed in by hand or came from a fixed list, so spellings differed and the logo list needed a code change to grow. Masters give one clean list, and Quick Add means a user is not sent away from a half-filled product. The shared master list code already existed, so the real work was wiring them into the product form and keeping the stored values unchanged.
- **Purchase flow:** payment was tied to the stock transfer, so a purchase could not be received first and paid later, which is how it works in practice. The real work was moving the payment record out of the stock transaction and making the two actions independent.
- **Smaller fixes:** each was a user-reported problem found while using the screens.

## How

**New masters**: they reuse the shared list code (one repository, service and router builder), so they behave exactly like Glass Colors and Glass Finishes, including duplicate-name errors and soft deactivation. The product form reads them through the central `/lookup`, so only the lookup and master write paths needed the new names.

**Stored values stay the same**: the product form still saves the picked record's **name** in the same columns (`layer` integer, `port`, `destination`, `logo` text). Nothing in order booking, work orders or printing needed to change.

**Quick Add**: the product form hides itself while the master's Add dialog is open (as "+ Add Color" already does), so nothing typed is lost. After a save the lookup refreshes and the new value is selected. Cancel returns to the form as it was.

**Add to Stock vs Payment**: the existing `inventory_purchase_stock_posting` table now means "payment recorded". No new table was needed. Its primary key is the purchase, so a second payment is refused. Add to Stock only moves lots and sets the stock status. It never reads or writes the payment record, and the payment never reads the stock status.

## Decisions made

**Layer names must be whole numbers (1 to 999).** The order's layer column is an integer that feeds work orders, so free text in the master would have broken it. This is checked in the screen and on the server.

**Port, Destination and Logo save the name, not an id.** This keeps the existing columns and every report or print working without changes. The trade-off is that renaming a master record later does not change orders already saved.

**A value that is not in the master stays shown in the product form.** Older orders (for example the old logo "RUBINE TUFF"), deactivated records and renamed records keep showing until the user changes them. There is also a "None" choice, because these fields are optional and a dropdown cannot otherwise be cleared.

**Seed data for the new masters was not added.** The masters start empty, as asked. Until records are added, the dropdowns show only "None" and the Quick Add item.

**Payment record reuses `inventory_purchase_stock_posting`.** A new table was not created. Purchases that were already added to stock with payment details already have a row, so they show as paid. Older purchases with no row can still be paid.

**Quick Add on Glass Type, Color and Thickness opens the Add Item modal.** Those dropdowns only offer values that the product's items have, so a missing value is added by adding an item, not by adding a glass type, color or thickness alone.

**Payment History became Supplier Bill in the Purchase View.** The row it listed is the invoice record saved with the purchase (its amount and bill file), not a payment. It was making every new purchase look fully paid. A real payment now exists only after the Payment action is saved.

## Bugs found while testing (already there before, not fixed here)

**1. `paidAmount` still counts the invoice record.** The backend adds up the invoice record as paid money, so `paidAmount` and `remainingAmount` in the API are not true paid figures. This is why the old view showed a new purchase as paid. The view no longer shows them, and no list column uses them (those columns are already commented out), but the API fields are unchanged. It affects any future screen that reads them.

**2. Lint errors in files touched here.** `product-detail-panel.tsx` has an empty-interface error, and the payment and master dialogs have "setState inside an effect" and "fast refresh" warnings. The payment dialog's effects were moved over unchanged from the old Add to Stock modal. They were already there before.

## Notes

- Rights need no change: all `master_*` screens are covered by the single Master Data view / control rights.
- The old document `docs/PR-party-gstin-supplier-bank-add-to-stock.md` still describes the earlier Add to Stock that took payment details. It was left as history.
- On PUR-20261106-138 the Item column shows "Frosted Clear" because the item in Items master is named that. It is data, not a bug.

## Next steps

- Add the Layer, Port, Destination and Logo records in each environment after deploy.
- Decide whether to fix the `paidAmount` / `remainingAmount` meaning in the API (bug 1) before any screen uses it.
- Clean up the lint errors listed in bug 2 when those files are next touched.

## Test plan

**Automated tests:**
- [x] Frontend `tsc -b`: no errors
- [ ] ESLint on the touched files: the errors listed in bug 2 remain, and no new kind of error was added

**Manual checks (local):**
- [x] Layer, Port, Destination and Logo screens: open from the sidebar, start empty, add, edit, view, activate / deactivate, search and paging work like the other masters. Layer rejects a name that is not a whole number
- [x] Add / Edit Product: the four fields are dropdowns of active records, **+ Add** opens the master's Add dialog, the new record is selected after saving, and cancelling keeps the form values
- [x] Purchase, Add to Stock: the modal shows only General Details and Product Details, and the stock transfer works with no payment details
- [x] Purchase, Payment: the modal shows General Details, Product Details and Bank & Payment Details, saves once, and the menu item is then disabled. It works before and after Add to Stock
- [x] A purchase with a recorded payment cannot be edited or deleted, and Add to Stock still works for it
- [x] Purchase View: Supplier Bill card and Paid / Unpaid label for a new purchase
- [x] New Purchase: **+ Add Item** on Glass Type, Color and Thickness opens the Add Item modal
- [x] The smaller fixes listed above
