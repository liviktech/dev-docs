# 2026-11-04 - party-redesign-order-status-master-fixes (backend)

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
| `src/database/migrations/120_drop_payment_condition_restriction_type.sql` (new) | Drops `master.payment_condition` and `master.restriction_type` |
| `src/database/seeds/003_master_reference_data.sql` | Starter rows for the two dropped tables removed, so a fresh database does not fail after migration `120` |
| `src/modules/master/payment-term/condition/`, `restriction/` | Deleted |
| `src/modules/master/payment-term/payment-term.routes.ts` | Only `/modes` left |
| `src/modules/superadmin/superadmin.repository.ts` | New-company copy no longer copies the two dropped lists |
| `src/modules/proforma-invoice/proforma-invoice.schema.ts` | `paymentConditionId` and `restrictionTypeId` removed from the PI request (they were accepted but never saved) |
| `src/modules/proforma-invoice/proforma-invoice.repository.ts`, `proforma-invoice.types.ts` | `sales_rep_name` added to the PI list (lean table query and full query) and the PI detail |
| `src/modules/report/order-status/order-status-report.repository.ts`, `order-status-report.service.ts` | New columns. Item-less Work Orders included. Per-Work-Order allocation of ordered qty, area, holes and cutouts. In-process qty. Current stage. Storage rack. PI balance and advance date. PI amounts on every row with a first-row flag |
| `src/modules/inventory-purchase/inventory-purchase.repository.ts` | Search matches numbers with the ₹ sign, commas and spaces ignored, and amounts also with 2 decimals |
| `src/modules/order-booking/order-booking.repository.ts` | New PI order picker: sales person is the enquiry's sales rep first; without one, the old rule (Direct for a direct booking, else the party's sales person) |

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

- Process: booking item Type.

- Order Notes: booking item Note plus booking Remarks.

- Customer Phone: party phone.

- Delivery Date: the Work Order's own.

- Storage Rack: where the finished pieces were stored.

## Decisions made

**Ordered Qty is not a plain piece count.** The audit suggested `SUM(woi.quantity)`. Live data has 17 booking sizes with fewer pieces than booked (for example 903 pieces for 1000 booked), so that would understate rows that are correct today. The allocation above gives identical ordered totals to the old report for every company, and still splits correctly; a split was tested on in-memory data.

**Tick / confirm pattern on Party addresses and bank accounts was added and then reverted** at the user's request. Only Projects keeps it (it had the tick before today). "Add Project" stays disabled until the open row is ticked.

## Bugs found while testing (already there before, not fixed here)

**1. Booking sizes with fewer Work Order pieces than booked.** 17 sizes (for example WO-20261006-102: 1000 booked, 903 pieces). This was already in the data before today. The report handles it (see "Decisions made"), but whatever creates the pieces may need a look. It affects anyone counting pieces per booking.

## Notes

- Run `npm run migrate` in the backend to apply `120`. It permanently drops the two Payment Terms tables and their rows. It was written but not run.
- The backend runs with `tsx watch` and restarts on every save. A charges request sent during a restart is what showed "Could not calculate charges" after picking a logo; the charge logic itself was correct. The new message makes this case clear.
- No split Work Orders or item-less Draft Work Orders exist in today's data. The two item-less Work Orders that exist are Completed; both now appear in the report.
- The backend `docs/` files (`DATABASE_SCHEMA.md`, `DATABASE_SCHEMA_GUIDE.md`, `SCHEMA_MIGRATION_TASKS.md`, `database_schema.sql`) still describe the two dropped tables.

## Next steps

- Decide whether the "Payment Slip / Receipt" upload moves from party approval to admin approval for PI and Final Invoice. This was discussed today and parked for the team.
- Update the backend `docs/` schema files for the dropped tables.
- Decide whether the blank "Payment Conditions:" line on the tax invoice should go.
- Look into why some booking sizes got fewer Work Order pieces than booked.

## Test plan

**Automated tests:**
- [x] ESLint on the new or changed bank-account and order-status files: clean.
- [x] Read-only script, Order Status report against live data, all companies:
- [x] Read-only script, each existing Order Status filter (status, dispatch, thickness, search, location, date) and paging: rows match the filter, no overlap between pages.
- [x] In-memory SQL check of the split allocation: 3 + 2 pieces of a size booked as 5, with one replacement, gave 6 and 2 pieces.
- [x] Read-only script, Purchase search: "₹11,350.00", "11,350.00", "11350.00" and "11350" all find PUR-20261104-134; number and supplier searches still work.
- [x] Charge preview with and without a logo (rolled back): with "Saint Gobain Cobranding" it returns the LOGO charge plus POLISH, no error.

**Manual checks (local):**
- [x] Payment Terms page with Payment Modes only, and creating a PI afterwards.
- [x] Work Order view "Order No", the Purchase search, and the Add Product charges message.
- [ ] Migration `120` on a real database - not run yet; it only drops two tables nothing references.
