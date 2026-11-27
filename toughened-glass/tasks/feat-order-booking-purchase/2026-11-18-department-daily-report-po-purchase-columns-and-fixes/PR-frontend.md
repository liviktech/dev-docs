# 2026-11-18 - department-daily-report-po-purchase-columns-and-fixes (frontend)

## Summary

This PR is the work from one day. The main piece is a new **Department-wise Daily Report** under Reports. The rest is Purchase Order and Purchase screen work (a rename, new table columns, a clickable Purchase Order link, a new layout for amounts and charges) plus two fixes: a server error on Admin Approval and a section that should not show in Party Approval.

Before this PR there was no report showing how many Enquiries, Order Bookings, PIs, Work Orders, Final Invoices and Dispatches a day has. "Assign Supplier" was the name of the Purchase Order action that really records the order. The Purchase table could not show the Purchase Order behind a purchase or its additional charges and last payment date. The Indent and Purchase Order view modals left gaps in their General Details. Admin Approval failed with "Internal server error" when the promised delivery date was today. Party Approval showed Delivery Details & Surcharges, which only the admin needs.

Each change keeps the existing flows working. All changes were tested in the browser. Frontend and backend type checks pass apart from one error that was already there (see "Bugs found while testing"). The new report's query and the Admin Approval query were also run on a scratch local Postgres built from all project migrations.

**Purchase Return is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/reports/department-daily/api.ts` (new) | Types and the request for the new report. `todayFilters()` makes the report open on today. |
| `src/features/reports/department-daily/department-daily-report.tsx` (new) | The report screen: date range filter, Reset, table (Department, Entries Created, Completed, Pending, Cancelled), a Total row, Excel and PDF downloads. |
| `src/features/reports/components/report-table.tsx` | New optional `footerRows` setting, passed to the shared `DataTable` footer. Other reports do not use it. |
| `src/features/reports/report-menu.ts`, `src/features/reports/registry.ts` | New "Department Daily" group with the "Department-wise Daily Report". |
| `src/features/pwa/reports/department-daily-report.tsx` (new), `src/features/pwa/reports/registry.ts` | The mobile version of the report. The mobile Reports home lists every menu entry, so it needs a screen. |
| `src/features/procurement/assign-supplier-dialog.tsx` | "Assign Supplier" is now "Order Status": title, success toast, error text and the "last updated by" note. |
| `src/features/procurement/procurement-tabs.tsx` | Row action renamed "Order Status", with its tooltips and toasts. Purchase Order table gets Products, Total Quantity (Sheets), Total Area (SQM), Order Status and Approved By columns. |
| `src/features/procurement/procurement-views.tsx` | View labels renamed (Supplier, Order Status Updated By, Order Status Updated Date). In the Indent and Purchase Order view modals, Remarks now comes last and spans the full width so the fields above it run on with no gaps. |
| `src/features/procurement/types.ts` | `totalArea` on the Purchase Order summary. `purchaseRequests` list on the purchase's procurement info. |
| `src/features/inventory/new-purchase/new-purchase-page.tsx` | Message text updated for the Order Status rename. |
| `src/features/inventory/purchase-tab.tsx` | New Additional Charges and Last Payment Date columns. Purchase Order Number moved to just after Purchase Number and made a link that opens the Purchase Order view. When a purchase came from several orders it shows the first with a "+N" list, like the Lot Code column in Stock. |
| `src/features/inventory/components/purchase-reference-cards.tsx` | New shared `PurchaseChargesBreakdown`. In the Payment modal, the product table now has only products and one Total row. |
| `src/features/inventory/components/purchase-view-dialog.tsx` | Amount & Charges section redesigned: charges breakdown on the left, amount summary on the right. Remark column is before Amount, and the total is in the last column. |
| `src/features/inventory/components/purchase-payment-dialog.tsx` | New "Charges" section using the same shared component. |
| `src/features/pi-status/components/approve-party-modal.tsx` | Delivery Details & Surcharges commented out of the Party Approval branch. The Admin Approval copy is untouched. |

## Why

The Department-wise Daily Report is a new reporting need: a daily count per department, without the production stages, which keep their own reports. The Order Status rename, the new columns and the Purchase Order link came from review feedback on the Purchase and Purchase Order screens. The Admin Approval error was reported while testing, and the Party Approval change is because only the admin sets delivery details.

Most of the work was in the report: deciding which date and which statuses mean "completed" for each department, and checking them against how the app really sets those statuses. The Admin Approval fix was a one-line query fix, but finding it needed a reproduction.

## How

**Department-wise Daily Report**: one query reads six tables, each with no joins, so a record is counted once, and each is filtered by the company. Enquiries, Order Booking, PI Status, Work Order and Final Invoice count on their own date: enquiry date, booking date, PI date, work order date and invoice date. Dispatch has no date of its own, so it counts on the day it was created (Asia/Kolkata). The date range follows the other reports: empty means no limit, and the report opens on today.

**Completed, Pending and Cancelled**: Completed and Cancelled come from a fixed list of statuses per department. Pending is everything else, so the three always add up to "Entries Created". Days with no records show 0.

**Total row**: `ReportTable` now accepts `footerRows` and hands it to the `DataTable` footer. The report uses the same padding as the body cells so totals sit under their columns. The Excel file and PDF get the same Total row.

**Purchase Order link**: the backend list now returns each order's id and number, so the frontend can show one link, or the "+N" list when there are several. It reuses `PurchaseNumbersCell` from the Stock tab and the read-only `PurchaseRequestViewDialog`.

**Amounts and charges**: one shared component, `PurchaseChargesBreakdown`, is used by the Purchase view and the Payment modal so their figures cannot differ.

**Removed UI**: the Party Approval copy of Delivery Details & Surcharges is commented out, not deleted, so it can be restored.

## Decisions made

**Department report counts by each record's own date, not by when it was saved.** The first version used the system creation time. An enquiry dated 17/11 but entered on the 18th then showed on the 18th, which was wrong for users. Dispatch is the one exception, because it has no date of its own.

**Status mapping.**

| Department | Completed | Cancelled | Pending |
|---|---|---|---|
| Enquiries | Converted | Closed | the rest |
| Order Booking | Confirmed (and Completed) | Cancelled | Draft |
| PI Status | Confirmed, Closed | Cancelled, Declined | Draft |
| Work Order | Completed | Cancelled (shown as Rejected) | the rest |
| Final Invoice | Confirmed | Declined, Cancelled | Draft |
| Dispatch | Confirmed | none (always 0) | Draft, Postponed |

Order Booking Confirmed is Completed because the app only ever sets Confirmed; the Completed status is never set. PI Closed is Completed because the system closes a PI by itself when it is fully invoiced, dispatched and paid. An early version counted Closed as Cancelled and was corrected after a PI closed that way showed as Cancelled.

**New "Department Daily" menu group.** There is already a "Department-Wise" report under Production Plan for production stages. A separate group avoids confusion with it.

**Column names.** Department, Entries Created, Completed, Pending, Cancelled. These were chosen with the requester. "Entries Created" sits over counts that are by each record's own date, which is slightly looser than the name suggests.

**No pagination footer on the Department report.** It always returns six fixed rows, so the table shows them all on one page (`showAllRows`). The server also returns all six rows together, so there is no server paging.

**Order Status column labels.** "Order Placed" (supplier and date saved) and "Awaiting Order" (approved but not saved) were chosen by the author. They are blank while the order is pending approval.

**No product charges in the Payment modal.** The request mentioned product charges, but a purchase has none. Its charges are the itemised additional charges, which are shown.

**New columns on the Purchase Order and Purchase tables cannot be sorted.** This avoids more backend sort changes.

**Purchase Return is not included.** It was planned but is on hold until the team decides whether received, accepted and rejected quantities belong per product or per purchase.

## Bugs found while testing (already there before, not fixed here)

**1. Express with a promised date of today triggers the same-day capacity check.** The screen only shows the capacity check for Same-Day, so an admin can get a "capacity exceeded" error with no warning for Express. This is existing behaviour. It affects admins approving PIs with a promised date of today.

**2. Order Booking Completed and Cancelled statuses are never set.** Bookings are confirmed or deleted. Deleted bookings, PIs and Work Orders are gone from the database, so they appear in no report column.

**3. Type check and lint errors in files this PR did not change.** The type check fails on a missing `leaflet` module in `src/features/parties/party-location-map-picker.tsx`. Lint reports errors in `approve-party-modal.tsx` (`any` types and a raw `toFixed`) and a React hook dependency error in `report-table.tsx` (the existing dependency list).

## Notes

- All changes were tested in the browser.
- Deleted Order Booking, PI and Work Order records cannot appear in the report. Showing them would need a deletion log.

## Next steps

- Decide whether Express should skip the same-day capacity check, and whether the screen should warn before the server refuses.
- Add Completed Qty and Completed SQM columns to the Production Plan Daily Production report once the meaning (same as Actual, cumulative, or fully completed pieces) is decided.
- Revisit Purchase Return once the QA quantity question (per product or per purchase) is answered. A debit note is for another team member.
- Fix the existing `leaflet` type error and the lint errors in `approve-party-modal.tsx` and `report-table.tsx`.

## Test plan

**Automated tests:**
- [x] Frontend type check (`npx tsc --noEmit -p tsconfig.app.json`): only the existing `leaflet` error in `party-location-map-picker.tsx`.
- [x] Lint on the new report files and the mobile report: no errors.

**Manual checks (local):**
- [x] Reports → Department Daily: opens on today, date filter changes all six rows, Reset, Total row aligned under its columns, Excel and PDF downloads. Tested in the browser.
- [x] Purchase Order list: Products, Total Quantity, Total Area, Order Status and Approved By columns. Tested in the browser.
- [x] Purchase Order row menu: "Order Status" dialog, toasts and view labels. Tested in the browser.
- [x] Purchase list: Additional Charges and Last Payment Date columns, Purchase Order Number after Purchase Number, link opens the order view, "+N" list for several orders. Tested in the browser.
- [x] Purchase view and Payment modal: charges breakdown on the left, amount summary on the right. Tested in the browser.
- [x] Indent and Purchase Order view modals: General Details run on without gaps. Tested in the browser.
- [x] PI Status: Party Approval has no Delivery Details & Surcharges; Admin Approval still has it. Tested in the browser.
- [x] Admin Approval with a promised date of today: approves without the server error. Tested in the browser.
- [ ] A PI with machining charges through the same-day capacity check - not checked with that data on the scratch database. The browser check of Admin Approval above is the only coverage for it.
