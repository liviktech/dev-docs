# 2026-11-03 - sales-rep-history-job-type (frontend)

## Summary

There was no way to see who had been the sales rep of a party over time, and the screens could not tell a Without Material order (customer glass) from a normal one. This PR adds a Sales Rep History report under Reports, adds a With Material / Without Material badge across the Order Booking, PI Status, Final Invoice and Dispatch views, and moves the Enquiry, Sales Visit, Storage Location and Customer Material view modals to the card layout of the Order Booking Details modal. It also stops the Add Product screen from putting the master material rate back on a Without Material product, shows a dash for empty cells in the Sales Rep Performance report, and lets the Purchase, finish stage and Customer Material location dropdowns show every active location.

`tsc --noEmit -p tsconfig.app.json` passes on the current tree. None of the screens were opened in a browser from here, and no automated tests were run or added.

**Several raw sheet sizes per Without Material product ("Add Pieces") is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/reports/sales-rep-history/api.ts` (new) | API calls for the report and its filter options. Only filters that are set go in the query string. |
| `src/features/reports/sales-rep-history/sales-rep-history-report.tsx` (new) | The report screen. Filters: date range (assignments that overlap it), party, sales executive, status (Current or Past). Columns: Party, Sales Executive, From, To ("Present" if current), Days, Status. Paging, Excel and PDF download like the FIR report. |
| `src/features/reports/report-menu.ts` | Adds "Sales Rep History" under Sales Rep. |
| `src/features/reports/registry.ts` | Attaches the new screen to `sales-rep:history`. |
| `src/features/reports/performance/sales-rep-performance-report.tsx` | Empty figures show a dash in the Enquiry Wise and Amount Wise tables, including the bold Total row, and in the Excel and PDF downloads. Before, the Total row and the downloads were blank. |
| `src/features/reports/performance/sales-rep-summary-reports.tsx` | The Target column of the all reps Amount summary shows a dash when empty, on screen and in the Excel export. |
| `src/components/ui/detail-cards-dialog.tsx` (new) | Shared read-only view dialog in the Order Booking Details layout: header with icon and code pill, bordered cards with an uppercase title and "Label : value" fields in three columns, Cancel footer, optional Edit button. |
| `src/features/enquiries/enquiry-view-modal.tsx` (new) | Read-only Enquiry view with General Details, Remarks and Photos cards. Clicking a photo opens the same photo window as the Photos button on the list. |
| `src/features/enquiries/enquiries-page.tsx` | "View" now opens the new view modal. Its Edit button switches to the existing edit form (only when the user can edit and the enquiry is not approved or converted, as before). Create and edit still use the form. |
| `src/features/order-booking/header-detail-tab.tsx` | The enquiry opened from Order Booking now uses the new view modal instead of the disabled form. |
| `src/features/sales-visits/sales-visit-view-dialog.tsx` | Rebuilt on the card dialog (General Details, Location, Linked Records, Remarks). Same props, so both callers keep working. Expense now shows with the rupee symbol. |
| `src/features/inventory/locations-tab.tsx` | Storage Location view uses the card dialog. |
| `src/features/inventory/customer-material-tab.tsx` | Customer Lot view uses the card dialog (General Details, Glass Details, Storage and Balance). The location dropdown now lists every active location, not just Raw Glass. |
| `src/features/inventory/components/purchase-dialog.tsx` | Storage Location dropdown lists every active location, not just Raw Glass. |
| `src/features/production-scan/components/finished-location-dialog.tsx` | "Store finished glass" lists every active location, not just Finished Goods. Empty message now says "No storage locations yet. Add one to continue." |
| `src/features/order-booking/job-type-badge.tsx` (new) | `JobTypeBadge` (slate "With Material", amber "Without Material", sky "Mixed" for PI Status), `jobTypeLabel` (plain text for print sheets) and `customerGlassText` (for example "5000 × 5000 mm · 10 sheets"). |
| `src/features/order-booking/order-details-modal.tsx` | Badge on each product header bar and in the Job Type field. A Without Material product shows a Customer Glass field with the raw sheet, and its Material Charges row reads "Customer Glass" with a dash amount instead of being hidden. |
| `src/features/order-booking/booking-preview-modal.tsx` | Job Type as plain text, a Customer Glass line, and the same Material Charges row for Without Material products. |
| `src/features/order-booking/order-booking-pdf.ts` | Job Type as plain text and the Customer Glass row in the product table. |
| `src/features/order-booking/add-product-modal.tsx` | Without Material now keeps the item rate at 0. The master rate auto fill, the previous order pricing lookup and its question, and the draft row rate fill are all skipped for that job type. |
| `src/features/pi-status/types.ts` | `jobType` on the PI record, and `job_type` on the raw item. |
| `src/features/pi-status/api.ts` | Maps `job_type` to `jobType`. |
| `src/features/pi-status/components/order-booking-view-modal.tsx` | Job Type tile with the badge. |
| `src/features/pi-status/components/work-order-view-modal.tsx` | Job Type tile with the badge in both states of the modal. |
| `src/features/pi-status/components/pi-status-drawer.tsx` | Badge next to the Work Order No in each order header of the PI view. |
| `src/features/final-invoices/types.ts` | `jobType` on the record, `job_type` on the raw item. |
| `src/features/final-invoices/api.ts` | Maps `job_type`. |
| `src/features/final-invoices/components/final-invoice-detail-modal.tsx` | Badge next to "Billed Glass Line Items". |
| `src/features/dispatch/types.ts` | `jobType` on the record, `job_type` on the raw item, and the `DispatchCustomerBalance` type. |
| `src/features/dispatch/api.ts` | Maps `job_type`. `approveDispatch` sends `returnBalanceGlass`. New `fetchDispatchCustomerBalance`. |
| `src/features/dispatch/components/dispatch-detail-modal.tsx` | Job Type field with the badge in the Invoice Reference card. |
| `src/features/dispatch/components/approve-dispatch-modal.tsx` | When the dispatch has customer balance glass, shows its list and a tick box "Balance glass returned to the customer with this dispatch" (off by default). The tick is sent with the approval. |
| `src/features/dispatch/queries.ts` | New `useDispatchCustomerBalance`. Approving also refreshes the inventory queries so returned glass leaves the Customer Material On Hand list. |
| `src/features/dispatch/dispatch-page.tsx` | Passes `returnBalanceGlass` from the approve popup to the mutation. |
| `src/features/production-scan/components/cutting-entry-dialog.tsx` | When the leftover glass is already planned for the next piece, the offcut fields are disabled with a note. Otherwise a note says whether the offcut goes back to stock or stays with the customer. |
| `src/features/production-scan/types.ts` | `offcutReused` and `customerSupplied` on the cutting plan. |
| `src/features/order-booking/product-detail-panel.tsx` | The label "Sheets Provided" now reads "Pieces Provided". |
| `src/features/enquiries/enquiry-form-modal.tsx`, `src/features/order-booking/work-order-edit-dialog.tsx`, `src/features/pi-status/components/approve-party-modal.tsx`, `src/features/final-invoices/components/approve-final-invoice-modal.tsx`, `src/features/pwa/enquiry/mobile-enquiry-form.tsx`, `src/features/pwa/final-invoices/final-invoice-approval-sheet.tsx`, `src/features/pwa/pi-status/pi-approval-sheet.tsx` | The only text change is the remarks placeholder, "(optional)..." removed. The rest of each diff is formatting (quote style, semicolons, line wrapping), no logic change. |
| `src/features/dispatch-plan/dispatch-plan-modal.tsx`, `src/features/dispatch-plan/dispatch-plan-panel.tsx`, `src/features/dispatch-plan/dispatch-plan-view-modal.tsx`, `src/features/dispatch/components/dispatch-table.tsx`, `src/features/final-invoices/components/final-invoices-table.tsx`, `src/features/final-invoices/components/create-final-invoice-panel.tsx`, `src/features/pi-status/components/pi-status-table.tsx`, `src/routes/work-orders.tsx`, `src/features/pwa/order-booking/mobile-order-detail.tsx`, `src/features/pwa/pi-status/mobile-pi-detail.tsx`, `src/features/pwa/work-orders/mobile-piece-detail.tsx` | A wrapper element or class (`inline-flex items-center gap-1.5`) around an existing cell or code. Layout only, no visible change on its own. |
| `src/features/order-booking/api.ts` | One comment line added, no real change. |
| `src/features/parties/party-view-modal.tsx` | no real change. A sales history section was tried in this dialog and then taken out when the report was built, so the file is back to its original content apart from one blank line. |

## Why

The Sales Rep History report answers "who looked after Company A and when", which the system could not do before. The card layout for the view modals was requested so these views match the Order Booking Details modal. The job type badge was needed because a Without Material order is billed and stocked differently, and reviewers could not tell the two apart on any screen. The Add Product fix was a regression: the master rate came back after the rate had been set to 0, so the Material Total showed a real amount for customer glass. The easy parts were the view modals. The real work was finding every place the master rate could be written back.

## How

**Report**: The screen follows the FIR report pattern (shared `ReportLayout`, `ReportTable`, `ReportActions`). The table shows one page. The downloads ask the server for every matching row. The party filter uses the slim party lookup and only loads when the dropdown is used.

**Card dialog**: One shared component takes a list of sections (title plus "Label : value" fields, or free content such as the photo grid) so the Enquiry, Sales Visit, Storage Location and Customer Material views are small and look the same. It uses the same class names as `order-details-modal.tsx`.

**Enquiry view**: Mounted only while open. Edit hands back to the existing form, so the form's own read-only rules for approved and converted enquiries are untouched.

**Job type badge**: One component, used wherever there is room for a badge. Print sheets and PDFs use plain text from `jobTypeLabel`. PI Status, Final Invoice and Dispatch read `jobType` from the new backend column. The Order Booking views read it from each product (`prod.item.jobType`) that was already loaded.

**Without Material rate**: The panel sets the rate to 0 when you pick Without Material, but that ran once. The add product modal now also checks the job type inside the master rate effect, the previous order pricing query and question, and the draft row rate fill, so none of them can write a material rate back. Switching back to With Material fills the master rate again.

**Locations**: The dropdowns call the same location lookup without a type, so they list every active location.

## Decisions made

**Order level badge on Order Booking Details was removed.** It was added first and then taken out. Only the per product badge stays, and its text is just "Without Material" (the "Customer Glass" wording stayed in the Customer Glass field and the Material Charges row).

**The view modals use the Order Booking Details card layout, not the Product view tabs or the PI Status tile cards.** Several layouts were tried before the card layout was picked from the screenshot of Order Booking Details.

**Receive Stock dialog on the Stock tab still shows Raw Glass locations only.** Only Purchase, the finish stage and Customer Material were asked for.

**The "+ Add Location" shortcut in the finish dialog still defaults a new location to Finished Goods.** Left as is.

**Several raw sheet sizes per product ("Add Pieces") is not included.** The request was a button in Customer Glass Details that adds another row with width and height. That needs several customer lots per product, and work order cutting, dispatch and returns assume one lot per product. It was stopped before anything was built, so no part of it is in this PR.

## Notes

- The menu says "Sales Rep History" but the report's own Excel sheet name and PDF title say "Sales Executive History". Worth making them match.
- The Enquiry view opens one photo window with all the photos, it does not jump to the photo you clicked, because the photo window has no option for that yet.
- Several type files have a comment `Without Material (customer glass) order` above fields it does not describe. These look like leftovers and have no effect. Left as they are.
- The Dispatch return balance tick box, the cutting entry changes and the wrapper elements were already in the working tree. They are listed from the diff and were not changed by this task.
- `jobType` on the PI, Final Invoice and Dispatch records only has a value once the backend change is deployed.

## Next steps

- Make the report names match (menu, Excel sheet, PDF title).
- Show the badge on the Work Orders list and the Production scan screens if wanted, they were not part of this change.
- Decide how Add Pieces should work before building it.
- Remove the stray comment lines and the layout only wrapper elements if they are not needed.

## Test plan

**Automated tests (none run, type check passes):**
- [x] `npx tsc --noEmit -p tsconfig.app.json`: passes with no errors on the current tree.
- [ ] No frontend unit tests were run or added for the report, the card dialog or the badge.

**Manual checks (local):**
- [ ] Open Reports, then Sales Rep, then Sales Rep History, and check the filters, paging, Excel and PDF - not run, not opened in a browser, needs the backend change and migration 115.
- [ ] Change a party's sales person and check the report shows the old and new person with the right dates - not run, same reason.
- [ ] Open the Enquiry view, click a photo and check the photo window opens - not run.
- [ ] Open the Sales Visit, Storage Location and Customer Material views and check the card layout and the Edit buttons - not run.
- [ ] In Add Product choose Without Material, change thickness or type, and check Material Total stays 0.00 and the Polish charge stays - not run.
- [ ] Check the badge on Order Booking Details, the preview and PDF, PI view, Final Invoice view and Dispatch view for a with material, a without material and a mixed order - not run, needs the backend change.
- [ ] Open the Purchase dialog, the finish stage dialog and the Customer Material location dropdown and check all active locations show - not run.
- [ ] Open Reports, Sales Rep Performance, with a rep that has no target and check dashes show on screen and in both downloads - not run.
