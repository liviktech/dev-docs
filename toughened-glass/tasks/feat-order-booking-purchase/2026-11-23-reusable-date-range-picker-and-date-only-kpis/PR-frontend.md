# 2026-11-23 - reusable-date-range-picker-and-date-only-kpis (frontend)

## Summary

This PR introduces a reusable, unified **Date Range Picker** in the page header across all applicable desktop list screens and 16 reports, standardizing how date filters and KPI cards behave across the entire application.

Previously, screens either had disparate Start Date and End Date input fields scattered in their table filter toolbars, or lacked quick presets. Moreover, KPI cards on several screens either responded to non-date filters (such as status, search, or party), creating stale or confusing summaries, or were hardcoded to all-time totals.

What this PR changes:
- **Shared Date Range Picker**: Reusable two-calendar picker with month/year navigation, quick date ranges (Today, This Week, This Month, Last 30 Days, This Quarter, This Financial Year), and a configurable Reset button. On mobile/phone screens, it collapses to a single month with arrow navigation.
- **Header Placement across List Screens**: Added to the page headers of Enquiries, PI Status, Order Booking, Work Orders, Final Invoices, Dispatch, FIR, Sales Visits, Archive, and the Production Plan tab.
- **Inventory Header Slot**: Since Inventory tabs share one page header, a header portal slot was added to `inventory.tsx` allowing the active tab (Stock, Purchase, Party Material, Movements, Indent, and PO) to render its date range picker cleanly into the header.
- **16 Reports Integrated**: Integrated into 16 reports through a new `dateRange` option on `ReportLayout`.
- **Date-Only KPI Scoping**: KPI cards now strictly reflect the selected date range, ignoring table-specific filters (such as status tabs, party, and search), ensuring clear and consistent business metrics.
- **Non-Destructive Filter Migration**: Old Start Date and End Date table filter inputs are commented out rather than deleted, keeping business forms intact.

`tsc` compiles with zero errors across both frontend and backend. Backend route and RBAC test suites pass. In the frontend test suite, 115 of 116 tests pass (the single failing test is the pre-existing, unrelated `obParty` case in `dashboard-filters.test.ts`). Manual verification in a live browser has not yet been performed.

---

## What changed

| File | Change |
|---|---|
| `src/components/ui/date-range-picker.tsx` (new) | The reusable date range picker component: two-month calendar on desktop, single month on mobile, month/year selector grids, quick range presets, and configurable default reset. |
| `src/components/ui/header-date-range.tsx` (new) | Portal wrapper and trigger button for mounting the date range picker into page header slots. |
| `src/lib/date-range.ts` (new) | Shared date calculation helpers for quick ranges (Today, This Week, This Month, Last 30 Days, This Quarter, This Financial Year) and formatting. |
| `src/features/dashboard/components/dashboard-date-range.tsx` | Refactored to use the new shared `DateRangePicker` component with "This Month" default. |
| `src/features/dashboard/period.ts` | Aligned with shared date calculation logic. |
| `src/routes/inventory.tsx` | Added `dateSlot` state and portal anchor in the inventory header so the currently active inventory tab can render its date range picker into the header. |
| `src/routes/work-orders.tsx` | Integrated header date range picker for Work Orders and Production Plan tab. |
| `src/routes/orders.tsx` | Integrated header date range picker for Order Booking. |
| `src/features/enquiries/enquiries-page.tsx` | Integrated header date picker; commented out old table date inputs; KPI cards scoped to date range only. |
| `src/features/pi-status/pi-status-page.tsx`, `pi-status-table.tsx` | Integrated header date picker; updated KPI cards to use separate date-only counts returned by the backend. |
| `src/features/order-booking/header-detail-tab.tsx` | Integrated header date picker; commented out old date inputs; KPI cards scoped to date range only. |
| `src/features/dispatch/dispatch-page.tsx` | Integrated header date picker; commented out old date inputs; KPI cards scoped to date range only. |
| `src/features/final-invoices/final-invoices-page.tsx` | Integrated header date picker; commented out old date inputs; KPI cards scoped to date range only. |
| `src/features/fir/fir-page.tsx` | Integrated header date picker; commented out old date inputs; KPI summary scoped to date range only. |
| `src/features/sales-visits/sales-visits-page.tsx` | Integrated header date picker; commented out old date inputs; KPI cards scoped to date range only. |
| `src/features/archive/archive-page.tsx` | Integrated header date picker; commented out old date inputs. |
| `src/features/inventory/stock-tab.tsx` | Renders date picker into inventory header slot; card totals count lots received within range. |
| `src/features/inventory/purchase-tab.tsx` | Renders date picker into inventory header slot; KPI summary receives date range. |
| `src/features/inventory/customer-material-tab.tsx` | Renders date picker into inventory header slot; card totals scoped to date range. |
| `src/features/inventory/movements-tab.tsx` | Renders date picker into inventory header slot (cards follow all filters). |
| `src/features/procurement/procurement-ui.tsx`, `procurement-list-state.ts` | Renders date picker into inventory header slot for Indents and Purchase Orders. |
| `src/features/production-plan/production-plan-panel.tsx` | Renders date picker into header for the Production Plan tab. |
| `src/features/reports/components/report-layout.tsx` | Added `dateRange` configuration option to render the shared date range picker in report headers. |
| `src/features/reports/*` (16 report files) | Adopted `dateRange` option on `ReportLayout`: Department Daily, Enquiry Follow-Up, FIR, Batch Consumption, Offcut Register, Order Status, Sales Rep Achievement, Department, Production Detail, Creditors, Daily Invoice, Debtors, Location Sales, PI Tracking, Sales Rep History, Visit Register. |
| `tests/pi-status-capabilities.test.ts` | Updated capability test assertions. |

## Covered Screens

### Main Modules (9)
1. **Enquiries** (`/enquiries`)
2. **PI Status** (`/pi-status`)
3. **Order Booking** (`/orders`)
4. **Work Orders** (`/work-orders`)
5. **Final Invoices** (`/final-invoices`)
6. **Dispatch** (`/dispatch`)
7. **FIR** (`/fir`)
8. **Sales Visits** (`/sales-visits`)
9. **Archive** (`/archive`)

### Inventory Sub-Tabs (6)
10. **Stock** (`/inventory?tab=stock`)
11. **Purchase** (`/inventory?tab=purchase`)
12. **Party Material** (`/inventory?tab=customer-material`)
13. **Movements** (`/inventory?tab=movements`)
14. **Indent Requests** (`/inventory?tab=indent`)
15. **Purchase Orders** (`/inventory?tab=po`)

### Production & Planning (1)
16. **Production Plan** (`/work-orders?tab=plan`)

### Reports (16)
17. **Department Daily Report**
18. **Enquiry Follow-Up Report**
19. **FIR Report**
20. **Batch Consumption Report**
21. **Offcut Register Report**
22. **Order Status Report**
23. **Sales Rep Achievement Report**
24. **Department Report**
25. **Production Detail Report**
26. **Creditors Report**
27. **Daily Invoice Report**
28. **Debtors Report**
29. **Location Sales Report**
30. **PI Tracking Report**
31. **Sales Rep History Report**
32. **Visit Register Report**

---

## Why

1. **Inconsistent User Experience**: Different screens implemented date filtering differently. Some had manual Start Date and End Date text inputs in table toolbars, while the Dashboard had an interactive two-month popup picker. Bringing the Dashboard date picker pattern to all list screens and reports unifies the UX.
2. **Distorted KPI Metrics**: On many screens, KPI cards were calculating metrics across all-time records or were unintentionally filtered down by status tabs and search inputs. KPI cards should reflect overall business performance for the selected time horizon, while table filters refine the tabular records below.
3. **Cluttered Filter Bars**: Moving the date range picker to the screen header frees up toolbar space for domain-specific filters (Party, Status, Representative, etc.).

---

## How

**Picker Architecture (`date-range-picker.tsx` & `header-date-range.tsx`)**:

- Desktop: Displays two side-by-side calendar months with independent month and year navigation grids, plus quick selection buttons (Today, This Week, This Month, Last 30 Days, This Quarter, This Financial Year).

- Mobile (Phone): Displays a single calendar month with left/right month arrow buttons.

- Trigger button: Displays the active date range formatted (e.g. `01/11/2026 ,  23/11/2026`) or "All dates" when no filter is applied.

- Reset: Clears or restores the configured default range. For most screens, default is "All dates" (no date filter). For daily screens (e.g. Daily Activity Summary), default is Today. For Dashboard, default is This Month.

**Inventory Header Slot (`inventory.tsx`)**:

- All Inventory sub-tabs (Stock, Purchase, Party Material, Movements, Indent, PO) share a single route and header.

- `inventory.tsx` establishes a portal slot ref (`dateSlot`). Whichever tab is actively mounted draws its own `HeaderDateRangePicker` into this shared slot, maintaining URL state synchronization per tab.

**Date-Only KPI Scoping**:

- **PI Status**: The API response returns both the filtered rows and a dedicated `dateOnlyCounts` payload. The "Approved This Month" card calculates the current calendar month count within the selected date window.

- **Stock Tab**: Summary counts now reflect lots received within the selected date range (`received_date BETWEEN fromDate AND toDate`).

- **Movements Tab**: Preserved as-is (Movements cards continue to track all active filters).

- **Table Data**: Table records continue to respect both the header date range and all toolbar filters (Party, Status, Sales Rep, Search, etc.).

**Preservation of Old Filters**:

- Old Start Date and End Date toolbar filter inputs have been commented out (not deleted) for easy regression verification.

- Underlying business forms containing Start/End date fields (e.g., contracts, order forms) remain completely untouched.

---

## Decisions made

1. **Header Slot for Inventory Tabs.**
   Because `/inventory` is a single layout component rendering dynamic tabs, child tabs could not easily render into the top header without a portal. Creating `dateSlot` in `inventory.tsx` allowed child tabs to inject their date picker cleanly into the header.

2. **Stock KPI Scoping.**
   Stock card totals now count lots received within the selected date range rather than all-time stock on hand, ensuring consistency with date range semantics across other modules.

3. **Commented Out vs Deleted Old Filters.**
   Old date inputs in table filter bars were commented out rather than deleted to ensure clear traceability during QA and simple rollback if needed.

4. **Movements KPI Preserved.**
   The Movements tab already filters its activity ledger across all selected criteria; its cards were kept following all active filters.

---

## Bugs found while testing (already there before, not fixed here)

1. **Dashboard test `obParty`**: `tests/dashboard-filters.test.ts` has a pre-existing test failure related to `obParty` URL filter state serialization. Unrelated to date range picker changes.

---

## Notes

- `tsc` passes without errors on both frontend and backend.
- Frontend test suite: 115 passing, 1 failing (pre-existing `obParty` test).
- Not yet visually verified in a live browser.

---

## Next steps

- Perform complete end-to-end testing in the browser across all 32 covered screens.
- Verify mobile phone responsiveness (single month calendar view, touch navigation).
- Verify quick preset ranges (Today, This Week, This Month, Last 30 Days, This Quarter, This Financial Year).
- Verify combined filter behavior (changing dates does not wipe party/status filters; changing party/status does not wipe dates).
- Check Stock KPI values against user expectations for received lots.

---

## Test plan

**Automated tests:**
- [x] Backend compilation & build: `tsc` passed with 0 errors.
- [x] Frontend compilation & build: `vite build && tsc -b` passed with 0 errors.
- [x] Frontend test suite: 115 of 116 passing (1 pre-existing failure in `dashboard-filters.test.ts`).

**Manual checks (local):**
- [ ] *Desktop verification*: Verify two-month layout, quick presets, Reset button, and KPI updates across all 9 main screens, 6 inventory tabs, and 16 reports.
- [ ] *Mobile/Phone verification*: Verify single-calendar drawer/popup, arrow navigation, and touch interactions on small viewports.
- [ ] *Combined filter check*: Verify that table filters (party, status, rep, search) operate alongside the header date range without resetting each other.
