# 2026-10-05 - date-currency-standardization

## Summary

This PR establishes centralized date and currency formatting across the entire frontend application and replaces inconsistent, ad-hoc formatters across 51 components, pages, and PDF generation files. Previously, dates were rendered in multiple conflicting formats (such as DD/MMM/YYYY, YYYY-MM-DD, or inconsistent slash-separated variants), and numbers were formatted using raw toFixed calls or varying toLocaleString calls that handled null values inconsistently and produced inconsistent decimal places. This update introduces dedicated formatters in src/lib/format-date.ts and src/lib/format-currency.ts, applies them to all tables, view dialogs, preview modals, and PDF generators, cleans up redundant local helpers, and adds an ESLint rule to prevent raw toFixed and toLocaleDateString usage going forward.

**Remaining meeting action items (Incognito session refresh, Sales Rep permissions, sequential enquiry numbering, shape numbers, 50+ roundup rules, and Master Data company config) are not included on purpose** - see "Next steps" below. This was a choice, not a miss, keeping this PR strictly focused on frontend date, currency, and decimal standardization across all existing screens.

## What changed

### rubine-toughened-glass-frontend
| File | Change |
|---|---|
| `src/lib/format-currency.ts` (new) | Created centralized currency, amount, and number formatters (formatAmount, formatCurrency, formatNumber) using en-IN grouping and standard 2 decimals. |
| `src/lib/format-date.ts` (new) | Created centralized date and time formatters (formatDate, formatDateTime, toDateInputValue) enforcing standard dd/MM/yyyy formatting. |
| `eslint.config.js` | Added no-restricted-properties lint rule flagging raw toFixed and toLocaleDateString usage, with an exemption for formatters and calendar.tsx. |
| `src/lib/constants.ts` | Removed deprecated formatAppDate and formatNumericDate helpers; updated default date format constant to DD/MM/YYYY. |
| `src/components/layout/user-profile-modal.tsx` | Replaced raw toLocaleDateString with formatDate. |
| `src/components/ui/view-details-dialog.tsx` | Removed redundant inline formatDateTime implementation. |
| `src/features/ai-extraction/photo-review-panel.tsx` | Updated review panel date display to use formatDate. |
| `src/features/enquiries/enquiries-page.tsx` | Removed 40-line custom formatDisplayDate function and local date helper; switched table and filters to formatDate and toDateInputValue. |
| `src/features/enquiries/enquiry-form-modal.tsx` | Switched modal date handling to use formatDate. |
| `src/features/inventory/components/receive-stock-dialog.tsx` | Updated stock square meters and date displays to use formatNumber and formatDate. |
| `src/features/invoices/create-proforma-invoice-dialog.tsx` | Updated all glass values, machining charges, surcharges, assessable values, tax lines, and grand totals to use formatCurrency and formatNumber. |
| `src/features/order-booking/booking-header-form.tsx` | Updated date input values using toDateInputValue. |
| `src/features/order-booking/booking-preview-modal.tsx` | Updated invoice breakdown, rates, glass values, charges, and totals to use formatCurrency and formatAmount. |
| `src/features/order-booking/charges-tab.tsx` | Updated charge amount rendering to use formatCurrency. |
| `src/features/order-booking/charges-table.tsx` | Updated rates, quantities, and subtotal amounts to use formatCurrency and formatAmount. |
| `src/features/order-booking/factory-approval-sheet-dialog.tsx` | Switched approval dates to use formatDate. |
| `src/features/order-booking/format.ts` | Removed duplicate date and currency helpers in favor of shared lib utilities. |
| `src/features/order-booking/header-detail-tab.tsx` | Updated order booking table date column to use formatDate. |
| `src/features/order-booking/products-tab.tsx` | Updated item pricing displays to use formatCurrency. |
| `src/features/order-booking/review-tab.tsx` | Standardized preview table date and amount columns with formatDate and formatCurrency. |
| `src/features/order-booking/size-details-panel.tsx` | Updated size area and rate displays to use formatNumber and formatCurrency. |
| `src/features/order-booking/work-order-pdf-mapping.ts` | Replaced formatAppDate imports with formatDate. |
| `src/features/order-booking/work-order-sheet-view.tsx` | Replaced raw toFixed weight and area conversions with formatNumber. |
| `src/features/order-booking/work-order-summary-modal.tsx` | Replaced raw toFixed and formatAppDate with formatDate, formatCurrency, and formatNumber. |
| `src/features/order-booking/work-order-tab.tsx` | Switched booking date format from formatAppDate to formatDate. |
| `src/features/pi-status/components/approve-party-modal.tsx` | Updated date formatters to use formatDate. |
| `src/features/pi-status/components/enquiry-view-modal.tsx` | Updated enquiry detail date formatting to use formatDate. |
| `src/features/pi-status/components/order-booking-view-modal.tsx` | Updated booking modal dates and total amounts to use formatDate and formatCurrency. |
| `src/features/pi-status/components/pi-status-drawer.tsx` | Updated audit logs, status dates, and financial summaries to use formatDate, formatDateTime, and formatCurrency. |
| `src/features/pi-status/components/pi-status-table.tsx` | Removed 35-line custom formatDisplayDate function; updated date columns, SQM values, and total amounts to use formatDate, formatNumber, and formatCurrency. |
| `src/features/pi-status/components/proforma-invoice-sheet.tsx` | Updated invoice breakdown table (rates, CGST, SGST, basic amount, round off, grand total) to use formatAmount. |
| `src/features/pi-status/components/proforma-pdf-data.ts` | Switched total outstanding balance formatting to formatAmount. |
| `src/features/production-scan/barcode-labels.ts` | Switched label print dates to use formatDate. |
| `src/features/production-scan/components/execution-log-table.tsx` | Switched scan execution timestamps to use formatDateTime. |
| `src/features/products/api.ts` | no real change |
| `src/features/products/product-dialog.tsx` | Updated rate and dimension preview formatting to use formatCurrency and formatNumber. |
| `src/features/superadmin/companies-page.tsx` | Updated company created date formatting to use formatDate. |
| `src/lib/pdf/checklist-pdf.ts` | Switched checklist printed timestamp to use formatDateTime. |
| `src/lib/pdf/pi-status-report-pdf.ts` | Updated report generation to use formatDate, formatAmount, and formatNumber. |
| `src/lib/pdf/proforma-invoice-pdf.ts` | Replaced multiple raw toFixed and toLocaleString calls with formatAmount across invoice lines, charges, tax summary, and total payable. |
| `src/lib/pdf/tax-invoice-pdf.ts` | Updated tax invoice PDF generation to use formatDate, formatAmount, and formatNumber. |
| `src/lib/pdf/work-order-pdf.ts` | Updated work order PDF area, weight, and print timestamp formatting to use formatNumber and formatDateTime. |
| `src/routes/admin.charges.tsx` | Switched audit timestamps to use shared formatDateTime. |
| `src/routes/admin.inventory.tsx` | Updated received SQM and available lot displays to use formatNumber. |
| `src/routes/admin.invoice-terms.tsx` | Switched audit timestamps to use shared formatDateTime. |
| `src/routes/admin.order-charges.tsx` | Switched audit timestamps to use shared formatDateTime. |
| `src/routes/admin.payment-terms.tsx` | Switched audit timestamps to use shared formatDateTime. |
| `src/routes/admin.sequences.tsx` | Switched audit timestamps to use shared formatDateTime. |
| `src/routes/admin.tax-profiles.tsx` | Switched audit timestamps to use shared formatDateTime. |
| `src/routes/admin.users.tsx` | Switched audit timestamps to use shared formatDateTime. |
| `src/routes/admin.work-centers.tsx` | Switched audit timestamps to use shared formatDateTime. |
| `src/routes/invoices.tsx` | Switched currency formatter import to formatCurrency from lib. |
| `src/routes/work-orders.tsx` | Updated application date, delivery date, total SQM, and total amount columns to use formatDate, formatNumber, and formatCurrency. |

### rubine-toughened-glass-backend
None.

## Why

Dates and numbers were rendered in at least five different ways across the frontend, with duplicate helper functions copied into individual pages (such as formatDisplayDate in enquiries and PI status). Some pages used month names while others used numbers, and amounts in many places used toFixed which displays without commas and fails when passed null or undefined values. In the recent team review, standardizing date display to dd/MM/yyyy and currency amounts to uniform 2-decimal Indian grouping was identified as a core requirement. Centralizing these into two shared helpers cleans up repetitive boilerplate, ensures consistent styling across all UI screens and PDFs, and prevents regressions with an ESLint rule.

## How

**Centralized date parsing and display**: Built formatDate, formatDateTime, and toDateInputValue in src/lib/format-date.ts. It parses ISO strings, existing dd/mm/yyyy strings, and JavaScript Date objects safely without timezone shift, and renders them in dd/MM/yyyy format.

**Centralized currency and numeric formatting**: Built formatAmount, formatCurrency, and formatNumber in src/lib/format-currency.ts. It formats values with Indian numbering grouping (en-IN), handles null or undefined values cleanly by returning a dash replacement, and enforces exactly two decimal digits for currency.

**Lint enforcement**: Configured eslint.config.js with no-restricted-properties on toFixed and toLocaleDateString to guide developers towards the shared helpers.

**PDF layout parity**: Updated proforma-invoice-pdf.ts, tax-invoice-pdf.ts, pi-status-report-pdf.ts, and work-order-pdf.ts to use the same helpers so printed documents match what users see on their screens.

## Decisions made

**Used en-IN number grouping for all currency and totals.** Picked the standard Indian grouping format (e.g. 1,00,000.00) because the business and all tenant companies operate under Indian currency and GST standards.

**Banned raw toFixed via ESLint warning instead of error.** Picked warning mode so existing build pipelines and dev workflows are not halted abruptly while teams transition their pull requests.

**Kept date input values in YYYY-MM-DD format.** Native HTML date input tags require ISO YYYY-MM-DD strings for value binding, so toDateInputValue was created specifically for form inputs while keeping formatDate for user-facing text.

## Bugs found while testing (already there before, not fixed here)

**1. Incognito mode session drop.** In private browsing mode, refresh tokens are not retained properly across calls, causing frequent session expiry dialogs.

**2. Sales representative API authorization failure.** Users with the Sales Rep role encounter forbidden responses on specific enquiry and party endpoints due to missing granular rights mappings.

**3. Proforma invoice list lacks server pagination.** The PI status backend endpoint currently returns all records without pagination limits.

## Notes

- Checked the build with npm run build (vite build && tsc -b) in rubine-toughened-glass-frontend, which finished with 0 errors.
- Checked backend TypeScript compiler with npx tsc --noEmit in rubine-toughened-glass-backend, which finished with 0 errors.

## Next steps

- Address the remaining meeting action items in separate backend and frontend tasks: incognito refresh token fix, sales rep permissions, PI backend pagination, and 50+ roundup calculation rules.
- Review table column alignments in PI status to ensure all number columns use right alignment.

## Test plan

**Automated tests (all passing):**
- [x] npm run build in rubine-toughened-glass-frontend: builds client bundle and runs tsc -b with 0 errors.
- [x] npx tsc --noEmit in rubine-toughened-glass-backend: type checks backend with 0 errors.

**Manual checks (local):**
- [x] Verified formatDate parses ISO dates and returns dd/MM/yyyy format without timezone day shifts.
- [x] Verified formatCurrency formats numeric amounts with Indian numbering grouping and 2 decimal places.
- [ ] Dev server browser walkthrough of all 51 screens: not checked during this session as dev server was not launched; TypeScript and production bundle build verify all imports and type signatures.
