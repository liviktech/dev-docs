# 2026-10-30 - dispatch-plan-loader-pi-gstin (frontend)

## Summary

This PR holds all the uncommitted frontend changes on `feat/report-goods`. There are six groups.

1. **Loading screen.** Every list page now shows the same loader as PI Status: a solid white panel that fills the space under the filter row. Before, Dispatch, Final Invoice and a few more showed a see-through panel, and many other pages showed a short half see-through box.
2. **PI preview GSTIN, State and State code.** These were not read from master data. GSTIN showed a dash and the state always fell back to TAMILNADU and 33. They now come from the party's billing address.
3. **Dispatch Plan can cover several Final Invoices.** The plan list, the plan editor, a new View dialog and a new Approve Requests dialog are rebuilt around a plan that has its own number and lines from many invoices. It needs the matching backend change.
4. **Dispatch Plan number.** A "Dispatch Plan (DPL)" option is added to the Sequences master.
5. **E-invoice vehicle and driver.** The e-invoice sheet now prints the vehicle number and driver name entered when the dispatch was approved.
6. **Small UI changes.** The Enquiry dialog title shows the enquiry number, number columns in five report tables are right aligned, and the Production Plan editor buttons were restyled.

**UID/Aadhaar is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/components/ui/table-loading-panel.tsx` | Added `bg-white` to the panel. This one line fixes every page that uses it (Dispatch, Dispatch Plan, Final Invoice, Production Plan panel, Sales Rep report, and the dispatch plan and production plan modals) |
| `src/routes/admin.charges.tsx`, `admin.invoice-terms.tsx`, `admin.order-charges.tsx`, `admin.payment-terms.tsx`, `admin.sales-reps.tsx`, `admin.sequences.tsx`, `admin.shifts.tsx`, `admin.tax-profiles.tsx`, `admin.users.tsx`, `admin.work-centers.tsx`, `party.tsx` | Loader box changed from `h-64 ... bg-white/60 backdrop-blur-md` to `flex-1 min-h-0 ... bg-white` so it fills the space and is solid white |
| `src/features/attendance/attendance-page.tsx`, `daily-attendance-tab.tsx`, `payroll-tab.tsx` | Same loader change |
| `src/features/glass-thickness/glass-thickness-page.tsx`, `stock-masters/stock-master-page.tsx`, `roles/roles-access-page.tsx`, `superadmin/companies-page.tsx`, `products/product-details-tab.tsx` | Same loader change |
| `src/features/notifications/messages-tab.tsx` | Same loader change (it used `h-48` and `bg-white/60`) |
| `src/features/inventory/locations-tab.tsx`, `movements-tab.tsx`, `purchase-tab.tsx`, `stock-tab.tsx` | Removed the leftover `h-64` next to `flex-1 min-h-0`. They were already `bg-white` |
| `src/routes/invoices.tsx`, `src/features/company/company-page.tsx`, `src/features/order-booking/review-tab.tsx` | Loader is now solid `bg-white`, but kept at `h-64` (see "Decisions made") |
| `src/features/pi-status/api.ts` | Added `pickBillingAddress`, `resolvePartyTaxIdentity` and a state name to state code table. `gstin`, `state` and `stateCode` on the PI record now come from these instead of keys the backend never sent |
| `src/features/dispatch-plan/api.ts` | Types and calls moved from "one plan per invoice" to "plan by id". `DispatchPlan` has `id`, `planNo`, `createdByName`, `createdAt`. Lines carry their own invoice, product and quantities. `DispatchPlanListItem` has `planNo` and `invoices`. `EligibleInvoice` became `PlanInvoiceOption` with `plannedElsewhere`. `fetchPlanInvoiceOptions(excludePlanId)` calls `/dispatch-plans/invoice-options`. `saveDispatchPlan(null, body)` creates (POST), otherwise replaces (PUT). All calls use `/dispatch-plans/:planId` |
| `src/features/dispatch-plan/queries.ts` | Query keys and mutations work by plan id. `planInvoiceOptionsQueryOptions` is always fresh (`staleTime: 0`). New `useRefreshPlans` refreshes the list and the invoice options after every save, submit, approve, reject or delete |
| `src/features/dispatch-plan/dispatch-plan-tab.tsx` | The plan editor: one plan can hold lines from several invoices. A new plan starts with one line per product still left on the chosen invoices. A product's free quantity takes off other plans and the other rows of this plan. Picking a product proposes what is left of it. A new plan carries on editing the same plan after its first save |
| `src/features/dispatch-plan/dispatch-plan-modal.tsx` | First screen lets you tick one or more confirmed invoices that still have something to dispatch, then Next. The old View / Edit mode is gone |
| `src/features/dispatch-plan/dispatch-plan-panel.tsx` | New first column "Dispatch Plan No" (opens the View dialog). Invoice and Work Order columns show every invoice on the plan, with links that open the invoice and the work order summary. The page header's New Plan button opens the create dialog through `creating` and `onCreatingClose` |
| `src/features/dispatch-plan/dispatch-plan-view-modal.tsx` (new) | Read only view of one plan, with Approve and Reject for approvers |
| `src/features/dispatch-plan/dispatch-plan-approve-requests-dialog.tsx` (new) | Dialog listing the plans waiting for approval, with Pending / Approved / Rejected tabs and approve, reject and delete actions |
| `src/features/dispatch/dispatch-page.tsx` | On the Dispatch Plan tab the header shows "Approve Requests (n)" and "New Plan" buttons. Counts plans in `SUBMITTED` status for the badge |
| `src/features/sequences/sequence-dialog.tsx`, `types.ts` | Added the `DISPATCH_PLAN` option ("Dispatch Plan (DPL)") and type |
| `src/features/final-invoices/api.ts`, `types.ts` | Map `dispatch_vehicle_number` and `dispatch_driver_name` to `dispatchVehicleNumber` and `dispatchDriverName` |
| `src/features/e-invoice/components/einvoice-sheet-data.ts` | The sheet's vehicle number falls back to `record.dispatchVehicleNumber`, and the driver name uses `record.dispatchDriverName` (it was always empty) |
| `src/features/enquiries/enquiry-form-modal.tsx` | View and Edit dialog titles now end with the enquiry number, for example "View Enquiry - ENQ..." |
| `src/features/production-plan/production-plan-editor.tsx` | Line remove uses the shared `DeleteRowButton`. The "Delete plan" button is now a small rose "Delete", "Save changes" is now "Save", and the buttons use the same compact style as the rest of the app |
| `src/features/reports/performance/sales-rep-performance-report.tsx`, `reports/sales-income/creditors-report.tsx`, `daily-invoice-report.tsx`, `debtors-report.tsx`, `location-sales-report.tsx` | Number columns changed from `align: 'center'` to `align: 'right'` |

## Why

- The loading screen should look the same on every list page. PI Status was the good one, written inline with `bg-white`. When `TableLoadingPanel` was added as a shared component it got the border and the spinner but no background. Many older pages had their own copy of a short half see-through box.
- For the PI preview, the sheet reads `gstin`, `state` and `stateCode` from the PI record, but the record mapping looked for them in places the backend does not fill. The state code also decides if the invoice uses CGST and SGST or IGST, so a missing state code made every customer look like an in-state customer.
- Dispatch plans: one dispatch often carries goods from more than one invoice, so the plan needed to be its own record with its own number.
- Vehicle and driver were empty on the e-invoice sheet even though the dispatch approval records them.
- Number columns read better right aligned.

## How

**Loader**: the shared panel now has `bg-white`. For the pages with their own box, the box uses `flex-1 min-h-0` so it grows to fill the page area under the filter row, the same as PI Status. Each of these pages has a flex column parent with a fixed height, so the box has room to grow.

**GSTIN, state, state code**: `pickBillingAddress` finds the billing address the same way the customer address text already did (the PI's own billing address, then the party's billing or default address). `resolvePartyTaxIdentity` then reads GSTIN and state from it. Anything the old code already read (`item.gstin`, `item.party.gstin`, `item.state_code` and so on) is still tried first, so nothing that worked before is lost. The state code is the first two digits of the GSTIN, and if there is no GSTIN it is looked up from the state name. The old `TAMILNADU` and `33` defaults in `proforma-pdf-data.ts` are untouched and now only apply when nothing is found.

**Dispatch plan**: every call goes by plan id. A new plan is created on its first save and then the dialog keeps editing that plan. After any save, submit, approve, reject or delete, the plan list and the invoice options are refreshed, because each change alters how much is left to plan.

**Quantity left**: for each product, the editor takes off what other plans already planned and what the other rows of the same plan took. It does not take off the row's own typed quantity, so the number does not shrink while typing. The row's own quantity is checked against it instead.

## Decisions made

**UID/Aadhaar is not fixed here.** Master data has no Aadhaar or UID column on the party or the address, and the party form has no field for it, so there is nothing to fetch. The preview still shows a dash. Fixing it needs a new field on the party (migration, party form, and returning it with the PI). I left that out and asked first.

**Three pages keep a short white box.** `routes/invoices.tsx`, the Company Profile page and the Order Booking review tab do not have a flex column parent, so a `flex-1` box there would shrink to nothing. They got the white background but keep `h-64`. `routes/invoices.tsx` is an older page that is not in the sidebar.

**Dialogs and modals keep their own small loaders.** Only the list page loading screens were changed.

**Pages that already matched were not touched.** PI Status, Enquiries, FIR, Work Orders and Sales Visits already had a solid white filling box.

## Notes

- `proforma-pdf-data.ts` and `proforma-invoice-sheet.tsx` have no change. They already read `activeRecord.gstin`, `state` and `stateCode`, which are now filled.
- The final invoice as proforma path (`final-invoice-as-proforma-data.ts`) already took the state code from the GSTIN and was not changed.
- The dispatch plan screens need the backend change and migrations 108 and 109 in `PR-backend.md`. Without them the new routes do not exist.
- Groups 3 to 6 in the summary were already in the working tree. They are described from the diff, not from a live run.

## Next steps

- Add a `uid_aadhaar` field to the party in master data if the PI sheet should show it.
- Move the other pages that still have their own loader box (PI Status, Enquiries, FIR and the inventory tabs) onto `TableLoadingPanel` so the look cannot drift again.

## Test plan

**Automated tests (type check passing, no unit tests cover this):**
- [x] `npx tsc --noEmit -p tsconfig.app.json`: no errors, run with all the changes above in the tree
- [ ] No test file covers the loader, the PI record mapping or the dispatch plan screens, so none was added

**Manual checks (local):**
- [ ] Open Dispatch, Final Invoice, Party and an admin page on a slow network and check the loading screen is a white panel filling the space under the filter row - **not done**, the pages were not opened in the browser. Covered only by the type check and by reading the parent layout of each page
- [ ] Open a PI preview for a party outside Tamil Nadu and check GSTIN, State and State code, and that tax shows IGST - **not done**, needs the backend restarted with the matching change
- [ ] Create a dispatch plan from two invoices, submit it, then approve and reject from the View and Approve Requests dialogs - **not done**, needs the backend migrations and routes
- [ ] Check the e-invoice sheet shows the vehicle number and driver name from an approved dispatch - **not done**
- [ ] Check the five report tables show number columns right aligned, and the Production Plan editor buttons look right - **not done**
