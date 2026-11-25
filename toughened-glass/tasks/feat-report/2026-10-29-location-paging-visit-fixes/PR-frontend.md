# 2026-10-29 - location-paging-visit-fixes (frontend)

## Summary

The Storage Locations tab filtered and searched everything in the browser, the Sales Visits screens had several small problems, and name text was cased inconsistently across the reports. This PR moves the Storage Locations tab to one paged server call, adds an Add Location choice inside the finished glass dropdown, makes the location code open a details box, shows the location name in the Stock table, fixes the Sales Visits search, cards and visit form, limits the Enquiry visit picker to visits that can still be used, lists the two Sales Rep reports as sub items of the Reports menu, and applies Pascal case to names in the reports.

**The frontend does not compile cleanly right now.** `tsc --noEmit -p tsconfig.app.json` reports 11 errors, all the same one: about 11 party name places pass a `party` prop to `PascalText`, but `src/components/ui/pascal-text.tsx` is back on its old `capShort` version, so the prop does not exist. See "Decisions made" and "Bugs found while testing" below. I did **not open any of these screens in the browser**, so none of the changes were checked live.

**Sorting on the Storage Locations table is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/inventory/locations-tab.tsx` | Loads one page at a time from the server. The Active, Inactive and All tabs, the search (sent after a short pause), the page size and the page number all go to the server. The three cards read the totals from the same response. The location code is now a link that opens a read only details box (code, name, type, status) with an Edit button. Stepping back a page when a deactivate empties the last page. |
| `src/features/inventory/types.ts`, `api.ts`, `queries.ts` | New `InventoryLocationRow` (only the columns the table shows), the paged response type, `fetchInventoryLocationPage` and `useInventoryLocationPage`. The old list call is unchanged. |
| `src/features/inventory/components/inventory-location-dialog.tsx` | Takes the slim row type. New optional `defaultType` so a new location can start as Finished Goods. |
| `src/features/production-scan/components/finished-location-dialog.tsx` | The Storage location dropdown ends with "+ Add Location", which opens the Add Storage Location form with Finished Goods preselected. The empty message is shorter. |
| `src/features/inventory/stock-tab.tsx` | New Location column (location name) between Lots and Available (SQM). |
| `src/features/sales-visits/sales-visits-page.tsx` | "Customer Name" is now "Party Name", the filter reads "All Parties", and the Total Visits card is the Immediate plus Later count so it no longer follows the selected tab. The party name uses the Pascal rule for party names. |
| `src/features/sales-visits/sales-visit-form-modal.tsx` | "Customer" is now "Party Name". Order Type and Expected Order Date only show for the New Enquiry purpose. For other purposes the visit is saved as Immediate with no expected date. Visit Expense is one third wide on every purpose. |
| `src/features/sales-visits/sales-visit-view-dialog.tsx` | The label reads "Party Name". |
| `src/features/sales-visits/api.ts`, `queries.ts` | The visit options call sends `forEnquiry=true` and an optional `includeVisitId`. |
| `src/features/enquiries/enquiry-form-modal.tsx` | The Sales Visit picker passes the enquiry's own visit as `includeVisitId`. Saving an enquiry refreshes the sales visit lists so a used visit drops out at once. |
| `src/features/reports/report-menu.ts` | The Sales Rep group is no longer a tabbed page. Sales Visit Register and Sales Rep Performance show as sub items in the sidebar like the other groups. |
| `src/features/reports/visit-register/visit-register-report.tsx` | Sales rep, customer name and visiting purpose cells use Pascal case. |
| `src/features/reports/performance/sales-rep-performance-report.tsx` | Sales rep cell uses Pascal case. |
| `src/features/reports/fir/fir-report.tsx` | Customer name and department use Pascal case. |
| `src/features/reports/production/production-detail-report.tsx` | Shift, stage, customer and product specification use Pascal case. The Customer column is now "Party Name", the download filter line reads "Party:" and the search box reads "Search party...". |
| `src/features/reports/production/department-report.tsx` | Department uses Pascal case. |
| `src/features/reports/production/production-options.ts` | Shift and stage dropdown choices use Pascal case. |
| `src/features/reports/sales-income/pi-tracking-report.tsx`, `debtors-report.tsx`, `daily-invoice-report.tsx`, `creditors-report.tsx` | Customer or supplier name is Pascal cased when the row is built, so the table, the PDF and the Excel file match. |
| `src/features/reports/sales-income/location-sales-report.tsx` | The Party Location column, the PDF and the Excel file use Pascal case. |
| `src/features/reports/sales-income/filter-options.ts` | The location dropdown choices use Pascal case. |
| `src/lib/pascal-text.ts` | "mm" is always MM, also when written apart ("12 mm"). A number with st, nd, rd or th keeps a lowercase ending ("9th Cross"). |
| `src/features/enquiries/enquiries-page.tsx`, `src/features/fir/fir-page.tsx`, `src/features/final-invoices/components/final-invoices-table.tsx`, `src/features/dispatch/components/dispatch-table.tsx`, `src/routes/invoices.tsx`, `src/routes/party.tsx`, `src/features/order-booking/order-details-modal.tsx`, `src/features/pi-status/components/pi-status-table.tsx`, `src/features/pi-status/components/pi-status-drawer.tsx` | Party name cells pass `party` to `PascalText`. This does not compile until `PascalText` has the prop again. |
| `src/routes/admin.charges.tsx`, `admin.order-charges.tsx`, `admin.shifts.tsx`, `src/features/stock-masters/stock-master-page.tsx` | The `capShort={false}` prop is removed. This does not compile until `PascalText` is decided, because `capShort` is back. |
| `src/components/ui/pascal-text.tsx` | The `party` prop was added here, then the file went back to the old version on disk. It is the file that needs a decision. |

## Why

The Enquiry list pages on the server and sends only the fields it shows, so the Storage Locations tab should work the same way. The Sales Visits changes fix things people hit: cards that did not follow the filters, a search that ignored the purpose, a form that asked for an order type on every purpose, and a visit picker that offered visits that already had an enquiry or an order. Reports mixed upper case and mixed case names, and the Raw Glass reports showed "mm" as "Mm".

## How

**Location paging**: The tab keeps the page, page size, status tab and search text in its own state, because the inventory page already owns the URL for its tabs. The search text is delayed before it reaches the query. Changing the tab or the search goes back to page 1.

**Add Location in the dropdown**: The shared option dropdown already supports an add item (`onAddNew`), the same way the receive stock form adds a supplier. The finished glass box reuses it and the existing location form.

**Visit picker**: The picker asks the server for New Enquiry visits that have no enquiry. The enquiry being edited sends its own visit so it can still show as selected.

**Pascal case**: Table cells and the rows used for downloads call `toPascalText`. The option lists for dropdowns do the same where they were raw values.

## Decisions made

**The `party` rule for `PascalText` is not settled.** I added a `party` prop (only party names keep short words in capitals, everything else is plain Pascal case) and marked about 11 party name places with it. Then `pascal-text.tsx` changed on disk back to the older `capShort` version, so I did not put it back. The call sites still say `party`. Either `party` goes back into `PascalText` or the call sites go back to the old behaviour. This needs a decision before the PR is merged.

**Other purposes are saved as Immediate.** The database requires an order type on every visit, so hiding the field means the visit is stored as Immediate with no expected date. These visits show "Immediate" in the table and count in the Immediate card. Making the column optional would need a migration and was **not done on purpose**.

**The location view box reads the row already on screen.** It makes no extra call, so it only shows the fields the paged list returns.

**The Stock table shows only the first location name for a combined row.** A row that combines lots from different locations shows the alphabetically first name, because the server picks it with `MIN`.

## Bugs found while testing (already there before, not fixed here)

**1. Order type filter and counts include visits with no real order type.** Because other purposes are stored as Immediate, the Immediate count is higher than the number of true Immediate visits. This comes from the database column being required, not from this change alone.

## Notes

- This PR needs the backend PR of the same name, because the paged location list and the visit filters are served from there.
- The `edit-requests?status=ALL` call still runs on the Storage Locations tab. It comes from the inventory page, not the tab, so it was left alone.

## Next steps

- Decide the `party` rule for `PascalText`, then make `tsc` pass again.
- Make the visit order type optional with a migration if the Immediate count should only hold real Immediate visits.

## Test plan

**Automated tests (no new tests added):**
- [ ] `tsc --noEmit -p tsconfig.app.json`: fails with 11 errors, all "Property 'party' does not exist on type 'PascalTextProps'".

**Manual checks (local):**
- [ ] Storage Locations tab paging, tabs, search and the code details box - not run, the screens were not opened in a browser.
- [ ] Add Location inside the finished glass dropdown - not run for the same reason.
- [ ] Sales Visits cards, search by purpose, and the New Enquiry fields in the form - not run for the same reason.
- [ ] Enquiry Sales Visit picker after an enquiry uses a visit - not run for the same reason.
- [ ] Reports in Pascal case, MM, and the Sales Rep sub items in the sidebar - not run for the same reason.
