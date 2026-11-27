# 2026-11-23 - reusable-date-range-picker-and-date-only-kpis (backend)

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
| `src/modules/enquiry/enquiry.repository.ts` | `getEnquirySummary` accepts `(companyId, fromDate, toDate)`, computing KPI cards independently of status, party, or search filters. |
| `src/modules/order-booking/order-booking.repository.ts` | `getOrderSummary` accepts `(companyId, fromDate, toDate)` for date-only card counts. |
| `src/modules/dispatch/dispatch.repository.ts` | `getDispatchSummary` accepts `(companyId, fromDate, toDate)` for date-only card counts. |
| `src/modules/final-invoice/final-invoice.repository.ts` | `getInvoiceSummary` accepts `(companyId, fromDate, toDate)` for date-only card counts. |
| `src/modules/sales-visit/sales-visit.repository.ts`, `sales-visit.service.ts` | `getSalesVisitSummary` accepts `(companyId, fromDate, toDate)` for date-only card counts. |
| `src/modules/inventory/customer-lot.repository.ts` | `getCustomerMaterialSummary` accepts `(companyId, fromDate, toDate)` for date-only card counts. |
| `src/modules/inventory/inventory.repository.ts`, `inventory.service.ts`, `inventory.controller.ts` | Stock lot summary query accepts `fromDate` and `toDate` to count lots received within range. |
| `src/modules/proforma-invoice/proforma-invoice.repository.ts`, `service.ts` | `listProformaInvoices` computes separate date-only KPI counts (`dateOnlyCounts`) alongside the filtered list. |

## Why

1. **Inconsistent User Experience**: Different screens implemented date filtering differently. Some had manual Start Date and End Date text inputs in table toolbars, while the Dashboard had an interactive two-month popup picker. Bringing the Dashboard date picker pattern to all list screens and reports unifies the UX.
2. **Distorted KPI Metrics**: On many screens, KPI cards were calculating metrics across all-time records or were unintentionally filtered down by status tabs and search inputs. KPI cards should reflect overall business performance for the selected time horizon, while table filters refine the tabular records below.
3. **Cluttered Filter Bars**: Moving the date range picker to the screen header frees up toolbar space for domain-specific filters (Party, Status, Representative, etc.).

---

## How

**Date-Only KPI Scoping**:

- **Enquiries, Order Booking, Dispatch, Final Invoices, Sales Visits, Party Material**: Backend summary endpoints were updated to accept `fromDate` and `toDate` only. KPI queries run independently of table search, party, or status filters.

- **PI Status**: The API response returns both the filtered rows and a dedicated `dateOnlyCounts` payload. The "Approved This Month" card calculates the current calendar month count within the selected date window.

- **Work Orders, FIR, Purchase**: Their dedicated KPI summary queries now forward `fromDate` and `toDate`.

**Preservation of Old Filters**:

## Notes

- `tsc` passes without errors on both frontend and backend.
- Backend route and RBAC tests pass.

## Next steps

- Verify combined filter behavior (changing dates does not wipe party/status filters; changing party/status does not wipe dates).
- Check Stock KPI values against user expectations for received lots.

---

## Test plan

**Automated tests:**
- [x] Backend compilation & build: `tsc` passed with 0 errors.
- [x] Backend route and RBAC test suites: All passing.

**Manual checks (local):**
- [x] Backend API endpoints and database operations verified.
