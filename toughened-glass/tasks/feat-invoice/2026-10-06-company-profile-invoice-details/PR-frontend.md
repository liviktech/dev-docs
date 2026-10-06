# 2026-10-06 - company-profile-invoice-details (frontend)

## Summary
This PR introduces the Company Profile management page under Master Data, adds editing support for company banking and invoice statutory details, automatically generates dynamic UPI QR payment codes, and wires company details into Proforma Invoice PDF generation. Administrators can view and edit the company profile, bank accounts, and invoice headers directly from `/admin/company`. Generated proforma invoices and preview modals now read company details dynamically from this page with graceful fallbacks to default values when fields are not configured.

## What changed
| File | Change |
|---|---|
| `src/routes/admin.company.tsx` (new) | Created route component mounting the CompanyPage at /admin/company. |
| `src/features/company/company-page.tsx` (new) | Built Company Profile interface with KPI cards, Company Information, Contact Information, Address card, Bank Details with live dynamic UPI QR code, Invoice Details card with statutory badges, and an Edit Profile modal. |
| `src/features/company/api.ts` (new) | Added API functions to fetch and update the authenticated company profile via /api/v1/master/company. |
| `src/features/company/queries.ts` (new) | Added TanStack Query hooks (useMyCompany, useUpdateCompany) with cache invalidation for company profile data. |
| `src/features/company/schema.ts` (new) | Added Zod schema and validation rules for editing company profile, bank, UPI, and invoice fields. |
| `src/features/company/types.ts` (new) | Defined TypeScript interfaces for company profile records and update payloads. |
| `src/components/layout/app-sidebar.tsx` | Added Company Profile link under the Master Data navigation group. |
| `src/features/pi-status/components/proforma-pdf-data.ts` | Updated buildProformaPdfData to accept companyDetail, dynamically mapping name, brand, address, phone, email, website, CIN, MSME, GSTIN, PAN, ISO certification, bank account, and UPI details with fallback to default company constants. |
| `src/features/pi-status/components/invoice-preview-modal.tsx` | Injected useMyCompany hook and passed live company data to ProformaInvoiceSheet. |
| `src/features/pi-status/components/print-options-modal.tsx` | Injected useMyCompany hook into Proforma Invoice PDF generation and Work Order print company fallback. |
| `src/lib/format-date.ts` | Refined date parsing regex to match exact YYYY-MM-DD date strings so ISO timestamps with time components preserve hours, minutes, and time zone information. |

## Why
Company details such as registered address, bank accounts, UPI IDs, and statutory registrations (PAN, CIN, MSME, ISO) were previously hardcoded in client-side PDF templates. There was no user interface to view or update company information. Adding a dedicated Company Profile page allows administrators to maintain their operational and financial data in one place and ensures generated invoices accurately reflect the current company configuration.

## How
**Company Profile interface**: Created `/admin/company` adhering to the standard master data UI pattern. The page displays balanced cards for Company Information, Contact Information, Address, Bank Details, and Invoice Details. Badges are applied to sensitive or statutory values such as PAN, CIN, MSME, and Account Number.

**Dynamic UPI QR code generation**: Used `qrcode.react` to render a standard `upi://pay?pa=...` payment QR code automatically whenever a valid UPI ID is configured, removing the need for manual QR image uploads.

**Dynamic Proforma Invoice generation**: Updated `buildProformaPdfData` in the PI status module to populate PDF data structures from `useMyCompany()`. If any fields are not yet set in the company profile, it falls back cleanly to the existing default values to guarantee invoices always render properly.

**Timestamp formatting fix**: Updated the regex in `format-date.ts` from `/^(\d{4})-(\d{2})-(\d{2})/` to `/^(\d{4})-(\d{2})-(\d{2})$/`. This ensures date-only strings avoid time zone shifts while full ISO strings (like `createdAt` and `updatedAt`) retain their full timestamp when displayed.

## Decisions made
**Graceful fallback to default invoice constants.** Existing proforma invoices must remain printable even if a company has not filled out all invoice metadata. The mapper checks company profile fields first and falls back to default constants when undefined or empty.

**Live QR code rendering instead of image file upload.** Rather than requiring users to generate and upload image assets for UPI payments, the UI and payment payload construct the standard UPI URI and render the QR code client-side from the stored UPI ID and name.

## Notes
- Verified production build live with `npm run build` (`vite build && tsc -b`), passing with exit code 0 and zero TypeScript errors.
- Confirmed responsive card layout stacks cleanly on smaller screens without horizontal overflow.

## Test plan
**Automated tests (all passing):**
- [x] `npm run build`: Vite production bundle and TypeScript build completed with exit code 0.

**Manual checks (local):**
- [x] Navigated to Master Data -> Company Profile from the sidebar and verified all cards render with proper spacing and alignment.
- [x] Verified UPI QR code updates in real time based on the entered UPI ID.
- [x] Opened Edit Profile dialog, modified bank and statutory fields, saved changes, and verified optimistic/invalidated cache refresh.
- [x] Verified Proforma Invoice preview and PDF export correctly pull company name, brand, bank details, and statutory fields from the updated company profile.
- [x] Verified created and updated timestamps display accurate date and time values.
