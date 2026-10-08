# 2026-10-08 - work-order-and-list-polish (frontend)

## Summary

The Work Order screens had a lot of rough edges: a modal for stages, text buttons in the actions column, fields nobody uses, and a Review sheet that showed two approval dates and only two sides of the glass. The Enquiry, PI Status and Order Booking lists also looked different from each other (alignment, fonts, loaders, filter bars, tab counts). This PR cleans up the Work Order screens first, then brings the Enquiry, PI Status and Order Booking lists in line with them, and adds one shared confirmation dialog for approve, reject, reopen and convert actions.

None of this was checked in a browser. I ran the TypeScript check after each change, but I did not click through the screens. See "Test plan".

**Photo scanning with AI on the New Enquiry modal is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/routes/work-orders.tsx` | Stages opens the full page instead of a modal. Actions are round icon buttons with tooltips, and a kebab menu (always last) holds Print, Edit and Delete. The view modal opens only from the Work Order number. Tabs are Completed, Draft, Pending Approval, In Progress, All and the page opens on Completed. KPI cards and tab counts keep their last values while a tab loads. Booking No and PI No are centered, Party Name is left. `font-mono` replaced by `tabular-nums`. The loader fills the table area. |
| `src/features/order-booking/work-order-row-menu.tsx` (new) | The kebab menu with Print Work Order, Print Scan Code, Edit and Delete. |
| `src/features/order-booking/work-order-print-menu.tsx` | Print logic moved into a shared `useWorkOrderPrint` hook, used by both the old Print dropdown and the kebab menu. |
| `src/features/order-booking/work-order-pieces-modal.tsx` | Removed, the Stages modal is no longer used. |
| `src/features/order-booking/work-order-edit-dialog.tsx` | Removed Completion Date, Del. Period, Business Area, Packing and Reference. Added disabled detail fields (WO No, PA No, Status, Party, Project, Application Date). Tighter padding. Approve keeps its label and shows a spinner. |
| `src/features/order-booking/work-order-form-fields.tsx` | Only Delivery Date, Dispatch Type and Remark are left. Takes an optional `className`. |
| `src/features/order-booking/work-order-tab.tsx` | Removed the read-only Completion Date, Del. Period, Business Area and Packing. Remark takes the full row. |
| `src/features/order-booking/work-order-summary-modal.tsx` | Dispatch Type moved into the top card. An empty delivery date shows a dash. |
| `src/features/order-booking/factory-approval-sheet-dialog.tsx` | Dropped the removed-field checks. Approve keeps its label with a spinner. Approve now asks for confirmation. No more "today" fallback for the approval date. |
| `src/features/order-booking/work-order-sheet-view.tsx` | One Approval Date field. Four side columns (Top Width, Bottom Width, Left Length, Right Length). Amendment No removed. |
| `src/features/order-booking/work-order-pdf-mapping.ts` | The route lists only real work centers (HOLE left out). Project shows a dash when empty. Single approval date (a dash until approved). Four side values with a fallback to actual width and height. |
| `src/lib/pdf/work-order-pdf.ts` | Printed Work Order now has one Approval Date, four side columns and no Amendment No. Header and total row widths adjusted. |
| `src/features/pi-status/components/print-options-modal.tsx` | Same single approval date and dash fallbacks for the print data. |
| `src/features/production-scan/scan-detail-page.tsx` | Shows the shared `Loader` while the page loads. |
| `src/components/ui/confirm-action-modal.tsx` (new) | One reusable "are you sure" dialog for approve, reject and reopen. Takes a tone (success, danger, info), a message, a description and an optional icon. |
| `src/features/enquiries/enquiries-page.tsx` | Apply and Reset removed, dates and party filter apply right away. Actions are right-aligned icon buttons with tooltips. Remarks and Customer Ref columns removed, Source and Last Updated added, Party and Project in bold. Clicking the enquiry number opens the edit modal in view-only mode. Approve, Reopen and Convert ask for confirmation. Start and End dates cannot be in the future. Loader fills the table area. |
| `src/features/enquiries/enquiry-form-modal.tsx` | Bigger labels, darker placeholders. New enquiries can attach photos (uploaded when the enquiry is saved). View-only mode for the number click. Approve and Convert in the footer ask for confirmation. |
| `src/features/pi-status/pi-status-page.tsx` | All PIs are fetched once and filtered in the browser, so tab counts and KPI cards no longer change when you switch tabs. Toolbar order is tabs, search, dates, plant. Filters apply right away (before, the date and plant pickers changed a draft that was never applied). Table loader. Start and End dates cannot be in the future. KPI numbers match Order Booking size. |
| `src/features/pi-status/components/pi-status-table.tsx` | Approve is now two icon buttons (cyan check for party, green shield for admin) and nothing is shown once fully approved. Delete is hidden for approved PIs. Remarks, Admin Approval Remark, Project, ASQM, CSQM and Plant columns removed. Alignment follows the new table skill. Plain text values share one weight. Balance amount text is bigger. `font-mono` replaced by `tabular-nums`. |
| `src/features/pi-status/components/pi-status-drawer.tsx` | Removed the inline Hanken Grotesk and Inter fonts and `font-mono`, so the modal uses the app font. Headings are 10px, content is 13px, and tables, fields and amount rows have borders. The layout is unchanged. |
| `src/features/pi-status/components/approve-party-modal.tsx` | PI Total no longer shows 0. It falls back to the PI detail total, then to the booking amount plus GST. |
| `src/features/order-booking/header-detail-tab.tsx` | Loader fills the table area like the Enquiry page. |
| `src/components/layout/app-sidebar.tsx`, `src/components/layout/superadmin-sidebar.tsx` | The selected menu item is stronger (30% cyan background, white text, brighter border and glow). |
| `.claude/skills/table-column-alignment/SKILL.md` (new) | Short skill describing the table alignment rules (first left, last right, text left, numbers and dates centered). |

## Why

The Work Order screens were the main ask, and each fix was a small request from using the screens (remove unused fields, show all four sides, one approval date, icons instead of text). Once those were done, the other lists were visibly different (different fonts, alignments and loaders), so the same rules were applied to them. The shared confirmation dialog came from the rule that approve, reject, reopen and convert should always ask first. The real work was the PI Status page, because its tab counts depended on the server and dropped to 0 while the next tab loaded.

## How

**Single approval date**: a work order has one approval that stamps both date columns together, so the sheet and the PDF show one "Approval Date" (the approved date, or a dash until it is approved).

**Four sides**: the sheet and PDF show Top Width, Bottom Width, Left Length and Right Length. The database only stores one actual width and one actual height, so the top and bottom show the width and the left and right show the height, the same as Order Booking.

**Process route**: it is built from the work centers on the item, and HOLE is left out because it is a system station that the Work Centers page hides.

**Tab counts on PI Status**: the page loads every PI once and works out the counts from the filtered list, so the counts do not depend on the tab. The table slices the filtered list for paging.

**Alignment**: `DataTable` already left-aligns the first column, right-aligns the last and centers the rest. Text columns in the middle opt in with `meta: { align: 'left' }`.

**Confirmations**: `ConfirmActionModal` is used for enquiry approve, reopen and convert and for the final work order approve. Delete and deactivate keep using `ConfirmDeleteModal`. The work order reject dialog and the PI approval modals already had their own confirm step, so they were not changed.

## Decisions made

**Photos on the New Enquiry modal are attach only, with no AI scan.** Enquiries have no line items, so there is nowhere to put the rows the AI reads, and the AI read is built for Order Booking. Photos are picked in the modal and uploaded after the enquiry is saved, so no backend change was needed.

**PI Status filters in the browser.** The server paging and counts were replaced with one fetch of all PIs, as the rest of the screens do. This is simple and keeps counts stable, but it will get heavy if the PI list grows into thousands.

**Table text uses `tabular-nums` instead of a monospace font.** Numbers still line up, and the font matches the rest of the app.

**The "locked from editing" banner only shows for status-locked enquiries.** In the new view-only mode the banner was wrong, so it is hidden there.

**Hiding Delete on approved PIs is only a screen change.** See the first bug below.

## Bugs found while testing (already there before, not fixed here)

**1. Approved PIs can still be deleted through the API.** The backend only blocks the delete when a work order exists. A PI with no work order can still be deleted. Only the button was hidden here.

**2. Some PIs have a total of 0 on the list.** The list API returns 0 for them while the details screen works the total out from the booking. The approval modal now falls back the same way, but the list still shows a dash in Total Amt for those PIs.

**3. The four sides are not stored separately.** Only one actual width and height are saved per size, so true top, bottom, left and right values need new columns.

**4. Existing items still have HOLE last in their route.** The printed route now hides HOLE, but the saved routing was not migrated.

**5. The Proforma Invoice print still shows "Amend: 0".** The Amendment No column is not used anywhere, and it was only removed from the Work Order.

## Notes

- Several of these changes were already committed or merged by the time this document was written, so the file list comes from the work done in this task, not from a fresh diff.
- The new loader and table rules were applied only to Work Orders, PI Status, Enquiries and Order Booking. Other pages (Party, Products, Admin) still have the old loader and alignment.
- Order Booking, the PI Status popups and some other screens still use `font-mono` or an inline Inter font in places.

## Next steps

- Block deleting approved PIs on the backend as well.
- Apply the table alignment skill and the new loader to the remaining list pages.
- Remove the leftover `font-mono` and Inter overrides in the Order Booking and PI Status popups.
- Add real top, bottom, left and right columns if the four sides must be stored.

## Test plan

**Automated tests (no test files for these screens):**
- [x] `npx tsc --noEmit -p tsconfig.app.json`: passed after each change. It was not re-run after the final merges.

**Manual checks (local):**
- [ ] Work Orders: Stages opens the full page, the kebab holds Print, Edit and Delete, the page opens on Completed, and switching tabs does not flash the KPI cards. Not checked in a browser.
- [ ] Review and Approve: Approval Date shows a dash before approval, the four side columns show, and Approve asks for confirmation. Not checked in a browser.
- [ ] Print a Work Order PDF: the header and table columns line up. Not checked.
- [ ] PI Status: tab counts stay the same when switching tabs, the date pickers filter, future dates are blocked, and approved rows show no approve or delete button. Not checked in a browser.
- [ ] Enquiries: the number click opens a disabled modal, the pencil opens an editable one, new photos upload after Save, and Approve, Reopen and Convert ask first. Not checked in a browser.
- [ ] PI Total in the party approval modal matches the details screen for a PI with a stored total of 0. Not checked.
