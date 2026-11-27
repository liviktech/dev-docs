# 2026-11-04 - party-redesign-order-status-master-fixes (frontend)

## Summary

A day of fixes and features across master data, PI and reports. Grouped by area:

1. **Product master (Add / Edit Product):** the modal is split into three tabs: Basic Details & Routing, Rates, and Charges. The inputs and dropdowns in the Rates and Charges rows are now the same height.
2. **Party modal:**
   - Redesigned into two columns: Party Information and Projects on the left, Addresses and Bank Accounts on the right.
   - Addresses are tabs, and bank accounts are collapsible cards.
   - Mobile is now mandatory, and the only address can no longer be deleted.
   - The footer button reads "Save Party Account".
3. **Supplier modal:**
   - It gets the same GSTIN "Fetch Details" lookup as the Party modal, using the shared lookup and its cache.
   - Phone is now mandatory.
4. **All Master Data edit modals:** the main button says **Save** instead of **Edit**.
5. **Default bank account and default address cards:** no blue outline any more.
6. **Payment Terms:** the Conditions and Restrictions lists are deleted completely: screen, API, PI form and database tables. Nothing in the app used them.
7. **Order Status Register report:**
   - 11 new columns.
   - Four data bugs fixed: missing Draft Work Orders, split Work Order quantities, progress during production, and PI amounts missing on later rows.
8. **PI Status:**
   - The table has a new **Sales Rep** column.
   - The New PI order list now shows the sales rep chosen on the order's enquiry, not the party's default.
9. **Small fixes:**
   - The Work Order view says "Order No" instead of "Order ID".
   - The Purchase search finds amounts typed as the table shows them ("₹11,350.00").
   - The Add Product (order booking) charges box says when the server could not be reached.

The author tested all of these in the browser. Both projects type-check. Several backend changes were also checked against live data with read-only scripts (see "Test plan").

**The Payment Terms tables are not dropped until migration `120` runs** - see "Notes". The code no longer uses them, but the data stays until the migration is applied.

## What changed

| File | Change |
|---|---|
| `src/features/products/product-dialog.tsx` | Three tabs (Basic Details & Routing / Rates / Charges), with the tab holding the first error opened on a failed save. Rate and charge inputs set to `h-8` to match the dropdowns. Footer button says "Save" in edit mode |
| `src/features/parties/party-dialog.tsx` | Two-column layout. Address tabs with the default address starred, replacing the address cards. Projects list scrolls inside its card. Row counts in the card titles. Title shows the party name when editing. "Save Party Account" footer. Mobile required. Delete hidden when only one address is left. "Add Project" disabled while a project row is unticked. Default address panel no longer outlined in blue |
| `src/features/parties/gstin-lookup-bar.tsx` | Optional `inputId` prop (defaults to `party-gstin-lookup`, so the Party modal is unchanged) so the Supplier modal can reuse it |
| `src/features/parties/party-view-modal.tsx` | Default address card no longer outlined in blue |
| `src/features/stock-masters/stock-master-dialog.tsx` | Suppliers only: GSTIN lookup bar at the top, filling name, PAN, address, city, state, pincode and GST state code with the Party modal's rules. The old GSTIN input in the tax grid is commented out. Phone required. Footer says "Save" when editing (also for Glass Types / Colors / Finishes) |
| `src/features/bank-accounts/bank-accounts-section.tsx` | New opt-in `collapsible` layout (used by the Party modal only). Each card has a summary header (bank, masked number, Default, Active switch) and expand / collapse. Default account no longer outlined in blue. The Supplier dialog's layout is unchanged |
| `src/features/bank-accounts/types.ts` | no real change |
| `src/features/company/company-bank-accounts.tsx` | Default account card no longer outlined in blue |
| `src/features/charges/charge-dialog.tsx`, `invoice-terms/invoice-term-dialog.tsx`, `order-charges/order-charge-dialog.tsx`, `shifts/shift-dialog.tsx`, `glass-thickness/glass-thickness-dialog.tsx`, `tax-profiles/tax-profile-dialog.tsx`, `sequences/sequence-dialog.tsx`, `items/item-dialog.tsx`, `sales-reps/sales-rep-dialog.tsx`, `roles/right-dialog.tsx` | `submitLabel="Save"`, so the edit modal no longer says "Edit" |
| `src/routes/admin.payment-terms.tsx` | Only Payment Modes left. Conditions and Restrictions removed. The one-tab bar removed |
| `src/features/payment-terms/payment-term-dialog.tsx`, `types.ts`, `api.ts`, `queries.ts` | Condition and Restriction parts removed. Dialog footer says "Save" |
| `src/features/invoices/create-proforma-invoice.tsx` | No longer loads or auto-picks a Condition / Restriction, and no longer sends their ids |
| `src/features/pi-status/components/proforma-pdf-data.ts`, `pi-status-drawer.tsx` | Removed reads of a "payment condition" name that nothing ever filled. Printed output is unchanged |
| `src/features/pi-status/components/pi-status-table.tsx`, `pi-status/api.ts`, `pi-status/types.ts` | New "Sales Rep" column after Party (left-aligned), mapped from `sales_rep_name` |
| `src/features/reports/order-status/api.ts`, `order-status-report.tsx` | 11 new columns. PI amounts on every row, with headers prefixed "PI". Each column is now defined once and drives the table, Excel, PDF and totals. PDF totals count each PI once. PDF printed on A3 |
| `src/lib/pdf/report-table-pdf.ts` | Optional `format` (`a4` / `a3`) and `fontSize`. The defaults are today's values, so the other reports are unchanged |
| `src/features/order-booking/work-order-summary-modal.tsx` | "Order ID" label changed to "Order No" |
| `src/features/order-booking/add-product-modal.tsx` | When the charges request gets no answer (network or server down), the message says "Could not reach the server..." instead of a plain "try again" |

## Why

- **Product modal tabs and heights:** asked for, so the long form is easier to work through.
- **Party redesign:** built from the provided wireframe.
- **Supplier GSTIN lookup:** asked for, reusing the Party integration so no extra provider calls or second integration were added.
- **"Save" label, plain cards, "Order No", mandatory phone:** UI consistency requests.
- **Payment Conditions / Restrictions:** an investigation showed nothing used them:
  - No foreign keys point at the tables.
  - The PI form loaded them but never showed them.
  - The PI form sent their ids only if they looked like UUIDs, which they stopped being after migration `040`, so nothing was ever sent.
  - The backend never saved them.

  The user asked for a full delete.
- **Order Status Register:** `docs/order-status-register-report-analysis.md` was the spec. The real work was getting the quantities right without breaking rows that were already correct (see "Decisions made").
- **Sales rep fixes:** order ORD-20261104-324 showed Bavin (the party's default) although Karthik Raja was chosen on its enquiry.
- **Purchase search:** checked whether it is a global search. It already was; only numbers typed with commas or ₹ did not match.

## How

**Supplier GSTIN lookup**: reuses `GstinLookupBar` and `GET /gstin/lookup/:gstin`, so the backend's 3-day cache is shared with the Party modal. The fill rules are the Party modal's: only blank fields, or fields the last fetch filled and the user has not touched.

**Party address tabs**: `selectedAddressId` chooses the tab, and null means the default address. A GSTIN fetch selects the address it filled. A failed save opens the first tab missing Address Line 1. The fill logic of `handleGstinFetched` is otherwise unchanged.

**Collapsible bank cards**: an opt-in prop on the shared section, so the Supplier dialog renders exactly as before. Saved accounts still deactivate (Active switch) and are never deleted; unsaved ones can be removed.

**"Save" label**: set on each Master Data dialog. The shared `FormDialog` default ("Edit" when editing) is unchanged, so edit modals outside Master Data are not affected.

**Order Status - ordered qty / area / holes / cutouts**, per booking size:

- Pieces all in one Work Order: that Work Order gets the full booking quantity.

- Pieces split over several Work Orders: each gets only its own pieces (replacement pieces not counted again), with area pro rata.

- A booked size with no pieces yet: goes to its product's Work Order when only one covers it.

- Holes and cutouts: per piece on the booking size, so they are multiplied by the allocated quantity.

**Order Status - production**:

- Produced Qty is still pieces with every stage done.

- In Process Qty counts live pieces started but not finished, once each.

- Current Stage is the first stage in route order not done for every piece, with "done / total". After that comes Ready for Dispatch or Dispatched.

**Order Status - PI amounts**: Basic, Total and Balance are on every row of the PI. `isPiFirstRow` marks one row per PI, and only those rows go into PDF totals. Balance uses the Final Invoice module's rule: total − agreed advance − what live Final Invoices collected, floored at 0.

**Order Status - other columns**:

- Process: booking item Type.

- Order Notes: booking item Note plus booking Remarks.

- Customer Phone: party phone.

- Delivery Date: the Work Order's own.

- Storage Rack: where the finished pieces were stored.

**Sales rep rule**: the sales rep on the order's enquiry, else "Direct". This is the same rule in the PI Status table, the Order Status report and the New PI picker. The picker keeps its old party fallback when there is no enquiry rep.

## Decisions made

**Ordered Qty is not a plain piece count.** The audit suggested `SUM(woi.quantity)`. Live data has 17 booking sizes with fewer pieces than booked (for example 903 pieces for 1000 booked), so that would understate rows that are correct today. The allocation above gives identical ordered totals to the old report for every company, and still splits correctly; a split was tested on in-memory data.

**PI Balance is derived.** The audit names `pi.balance_amount`, but that column does not exist. The Final Invoice rule was used because it is what the app already treats as the amount still owed. The PI list's "total − advance" ignores money already collected on invoices.

**Order Notes uses Note and Remarks, not Description.** On the booking form, `description` and `remark` are labelled "Party Description" and "System Description". They describe the glass rather than give instructions.

**PI amounts repeat instead of showing on the first row only.** This keeps a later Work Order from looking unpriced. Totals still count each PI once. In the Excel export, adding up those columns by hand counts a shared PI more than once.

**Payment Conditions / Restrictions deleted, not commented out.** The user asked for a full delete. The tab bar on the Payment Terms page was removed too, because one tab is not a choice.

**Tick / confirm pattern on Party addresses and bank accounts was added and then reverted** at the user's request. Only Projects keeps it (it had the tick before today). "Add Project" stays disabled until the open row is ticked.

**Supplier phone is only checked for being filled in,** not for the 10-digit format the Party modal uses.

**The New Stock dialog still lists Raw Glass locations only.** The request about storage locations named New Purchase and the finished-glass dialog. Both already list every active location (commit `12fce02`), so no change was made there.

## Bugs found while testing (already there before, not fixed here)

**1. Booking sizes with fewer Work Order pieces than booked.** 17 sizes (for example WO-20261006-102: 1000 booked, 903 pieces). This was already in the data before today. The report handles it (see "Decisions made"), but whatever creates the pieces may need a look. It affects anyone counting pieces per booking.

**2. The tax invoice sheet prints a blank "Payment Conditions:" line.** `einvoice-tax-invoice-sheet.tsx` always prints it empty. It does not read the deleted master, so it was left alone. It affects printed tax invoices.

**3. Existing lint errors.** `party-dialog.tsx` (setState inside an effect, an `any`, a `let` that should be `const`) and `admin.payment-terms.tsx` (route export pattern, unused `err`) failed lint before today. They were not touched.

## Notes

- Run `npm run migrate` in the backend to apply `120`. It permanently drops the two Payment Terms tables and their rows. It was written but not run.
- The backend runs with `tsx watch` and restarts on every save. A charges request sent during a restart is what showed "Could not calculate charges" after picking a logo; the charge logic itself was correct. The new message makes this case clear.
- The Order Status PDF is A3 landscape at a smaller font because it has 28 columns.
- No split Work Orders or item-less Draft Work Orders exist in today's data. The two item-less Work Orders that exist are Completed; both now appear in the report.
- The User modal (Attendance › Users and the unlinked `/admin/users` page) still says "Edit". It was treated as outside Master Data.

## Next steps

- Decide whether the "Payment Slip / Receipt" upload moves from party approval to admin approval for PI and Final Invoice. This was discussed today and parked for the team.
- Decide whether the blank "Payment Conditions:" line on the tax invoice should go.
- Look into why some booking sizes got fewer Work Order pieces than booked.
- Decide whether the New Stock dialog should list every storage location, like New Purchase.

## Test plan

**Automated tests:**
- [x] Backend `tsc --noEmit` and frontend `tsc --noEmit -p tsconfig.app.json`: no errors.
- [x] ESLint on the new or changed bank-account and order-status files: clean.
- [x] Read-only script, Order Status report against live data, all companies:
- [x] Read-only script, each existing Order Status filter (status, dispatch, thickness, search, location, date) and paging: rows match the filter, no overlap between pages.
- [x] Read-only script, PI list (table view): every row has a sales rep (for example PI-20261104-240 → Arun Kumar).
- [x] Read-only script, New PI picker: ORD-20261104-324 → Karthik Raja, ORD-20261104-321 → Bavin, ORD-20261103-310 → Arun Kumar, direct bookings → Direct.
- [x] Read-only script, Purchase search: "₹11,350.00", "11,350.00", "11350.00" and "11350" all find PUR-20261104-134; number and supplier searches still work.
- [x] Charge preview with and without a logo (rolled back): with "Saint Gobain Cobranding" it returns the LOGO charge plus POLISH, no error.

**Manual checks (local):**
- [x] Product modal tabs, the Rates and Charges field heights, and saving.
- [x] Party modal redesign: layout, address tabs, bank cards, mandatory mobile, single-address delete hidden, GSTIN fetch, saving.
- [x] Supplier modal: GSTIN fetch fills the fields, mandatory phone, saving.
- [x] "Save" on the Master Data edit modals.
- [x] Plain default bank and address cards.
- [x] Payment Terms page with Payment Modes only, and creating a PI afterwards.
- [x] Order Status Register: new columns, filters, Excel and PDF.
- [x] PI Status Sales Rep column, and the New PI picker sales person.
- [x] Work Order view "Order No", the Purchase search, and the Add Product charges message.
