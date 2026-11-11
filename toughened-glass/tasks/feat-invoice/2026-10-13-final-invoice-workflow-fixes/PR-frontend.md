# 2026-10-13 - final-invoice-workflow-fixes (frontend)

## Summary
This PR upgrades the Final Invoice page to server-driven pagination, cleans up list API requests to match PI Status, lazy-loads payment receipts and invoice details, and replaces static preview fallbacks with live customer and order booking data. Previously, the final invoice page loaded all rows client-side, sent redundant parameters like partyId=ALL and status=ALL, triggered unneeded parties master fetches on load, and used static placeholder addresses and GSTIN in invoice preview. The page now requests only display fields via view=table, runs server-side pagination and KPI cards, lazy-loads details and receipt photos on demand, and dynamically maps billing details and final invoice terms.

## What changed
| File | Change |
|---|---|
| `src/features/final-invoices/final-invoices-page.tsx` | Replaced client-side array slicing with server-side pagination (finalInvoicePageQueryOptions), normalized query filters to omit ALL values, integrated server-side KPI totals and status tab counts, updated CSV export to request unpaged full rows, and mounted FinalInvoiceDetailModal conditionally to prevent React hook order issues. |
| `src/features/final-invoices/components/final-invoices-table.tsx` | Integrated server pagination controls (manualPagination), updated payment proof column to check hasPaymentProof and lazy-load data URIs through extraPhotos, and added timestamp fallback for receipt photo review. |
| `src/features/final-invoices/api.ts` | Added fetchFinalInvoicePage endpoint caller, updated buildListEndpoint with page, limit, and view parameters, and updated mapApiToFinalInvoiceRecord to safely handle optional internal fields from lean table responses. |
| `src/features/final-invoices/types.ts` | Added FinalInvoicePage, ApiFinalInvoicePage, FinalInvoiceKpis, and FinalInvoiceFiltersState interfaces, and made workOrderId and createdAt optional in records. |
| `src/features/final-invoices/queries.ts` | Added finalInvoicePageQueryOptions query factory for TanStack Query caching. |
| `src/features/final-invoices/components/final-invoice-detail-modal.tsx` | Reordered React hooks so useQuery runs unconditionally at top level, and converted details view to lazy-load by ID on mount. |
| `src/features/final-invoices/components/final-invoice-preview-modal.tsx` | Integrated useMyCompany, useParties, finalInvoiceDetailQueryOptions, and fullOrderBookingQueryOptions to dynamically resolve customer billing address, GSTIN, phone, and terms. Set default sheet view to landscape, and removed payment receipt screenshot from formal printable invoice sheet. |
| `src/features/final-invoices/final-invoice-as-proforma-data.ts` | Replaced static placeholders with dynamic party address, GSTIN, phone, dynamic CGST/SGST splitting, and dedicated Final Invoice terms and conditions (removing proforma advance payment terms). |
| `src/features/pi-status/components/proforma-invoice-sheet.tsx` | Adjusted table border utility classes in payment summary section to prevent excessively thick borders. |
| `src/features/ai-extraction/photo-review-panel.tsx` | Added support for inline data URI photos in preview and thumbnail carousel. |
| `src/features/ai-extraction/photo-thumb.tsx` | Allowed rendering thumbnails from inline data URLs. |
| `src/features/ai-extraction/photos-button.tsx` | Supported custom photo loaders via extraPhotos configuration. |
| `src/features/ai-extraction/photos-column.tsx` | Enabled passing extraPhotos callback to photo column definition. |
| `src/features/ai-extraction/types.ts` | Added extraPhotos typing to PhotosColumnOptions. |

## Why
The final invoice list and preview had several issues: the table fetched every row in the company into memory and filtered locally, list API calls passed unnecessary filter defaults (partyId=ALL, status=ALL), preview sheets showed dummy static text for customer GSTIN, address, and advance terms, and large payment receipt images were bloated into the main list payload. Aligning with the PI Status architecture provides faster page loads, cleaner network requests, and accurate dynamic invoice printouts.

## How
**Server-driven pagination and filters**: Wired page, pageSize, status, and search into finalInvoicePageQueryOptions. Omitted ALL values from the request URL and sent view=table so the backend only sends lean display columns.

**Lazy photo and detail loading**: The table now checks boolean hasPaymentProof. When the user clicks the photos action, extraPhotos.load calls finalInvoiceDetailQueryOptions(invoice.id) to fetch the base64 receipt photo on demand without weighing down table rows.

**Dynamic preview data transformer**: In buildFinalInvoiceAsProformaData, customer information is resolved from partyRecord and fullBooking, terms and conditions use dedicated Final Invoice rules (such as balance payment due within agreed credit period rather than proforma advance terms), and taxes dynamically calculate CGST/SGST splits.

**Hook lifecycle safety**: In FinalInvoiceDetailModal, moved useQuery to the component top level and conditioned execution via enabled: Boolean(record?.id). In FinalInvoicesPage, mounted the modal conditionally when detailRecord is present.

## Decisions made
**Removed payment receipt image from printable preview sheet.** Payment receipts belong to accounting and verification workflows (accessible via the Photos column), not the formal customer-facing tax invoice sheet.

**Defaulted invoice preview orientation to landscape.** Final Invoices share the 13-column item layout with Proforma Invoices, which requires landscape orientation to avoid truncated descriptions and columns.

**Maintained backward-compatible export via unpaged query.** When exporting to CSV, handleExportCSV calls fetchFinalInvoiceList with view: undefined and unpaged parameters so full records including party codes export correctly.

## Notes
- Verified build and TypeScript compilation live with npm run build (vite build && tsc -b), passing with exit code 0.
- Confirmed preview layout and border thicknesses render cleanly in browser preview.

## Test plan
**Automated tests (all passing):**
- [x] npm run build: Vite production bundle and TypeScript build completed with exit code 0.

**Manual checks (local):**
- [x] Verified Final Invoice table renders page 1 with 10 items, tab counts, and KPI cards using server response.
- [x] Verified network tab shows /api/final-invoices?page=1&limit=10&status=DRAFT&view=table without partyId=ALL or status=ALL.
- [x] Verified clicking details opens modal with lazy-loaded item rows, charges, and taxes without React hook errors.
- [x] Verified clicking preview opens landscape sheet with dynamic party address, GSTIN, and Final Invoice terms.
- [x] Verified photos button displays payment receipt image on demand when clicked.
- [x] Verified CSV export downloads full matching records.

