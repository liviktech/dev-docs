# 2026-10-06 - company-profile-invoice-details (backend)

## Summary
This PR adds database support and API endpoints for company banking, UPI payment, and statutory invoice metadata in `core.company`. It introduces two database migrations adding columns for bank details, UPI ID and name, company email, and invoice statutory credentials including brand name, PAN, CIN, ISO certification, MSME, and website. The company master module types, validation schemas, repository queries, and service mappers have been updated to allow viewing and editing these fields.

## What changed
| File | Change |
|---|---|
| `src/database/migrations/048_company_bank_upi_email.sql` (new) | Added bank_name, bank_account_no, bank_ifsc_code, bank_branch, upi_id, upi_name, and email columns to core.company. |
| `src/database/migrations/049_company_invoice_fields.sql` (new) | Added brand_name, pan, cin, iso_certification, msme, and website columns to core.company. |
| `src/modules/master/company/company.types.ts` | Updated CompanyRow and CompanyDetail interfaces to include all new bank, UPI, email, and statutory invoice fields. |
| `src/modules/master/company/company.schema.ts` | Updated updateCompanySchema to validate optional email, bank fields, UPI identifiers, and statutory fields with case normalization. |
| `src/modules/master/company/company.repository.ts` | Expanded BASE_COMPANY_SELECT and updateCompany SQL query to select and update all new company columns dynamically. |
| `src/modules/master/company/company.service.ts` | Updated mapRowToDetail to map the new database columns into camelCase company response objects. |
| `src/modules/master/company/company.controller.ts` | Added handling and response serialization for the updated company profile fields. |
| `src/modules/master/company/company.routes.ts` | Updated company route validation for company update payloads. |

## Why
Previously, the company record only stored basic information such as company name, code, GSTIN, and phone. Bank account numbers, IFSC codes, UPI details, and statutory numbers (PAN, CIN, MSME, ISO certification) were hardcoded across frontend invoice templates. Storing these in `core.company` lets administrators manage their bank accounts and legal details directly from the application and allows downstream invoicing to read them dynamically.

## How
**Database schema expansion**: Added migrations 048 and 049 with `ALTER TABLE core.company ADD COLUMN IF NOT EXISTS` statements for each required field, ensuring safe idempotency during deployments.

**Repository query updates**: Updated `BASE_COMPANY_SELECT` to include all new columns and extended `updateCompany` to dynamically construct parameterized SQL SET clauses for whichever optional fields are provided in the patch payload.

**Zod input validation**: Updated `updateCompanySchema` to allow optional updates with string trimming, uppercase transforms for PAN and IFSC codes, and standard email validation.

## Decisions made
**Kept bank and invoice fields optional in the company schema.** Existing companies may not have bank accounts or MSME registration configured immediately upon creation, so making these columns nullable avoids breaking onboarding workflows while still allowing full editing when available.

## Notes
- Checked type safety live with `npx tsc --noEmit` and build with `npm run build`, exiting cleanly with code 0.
- Migrations 048 and 049 were applied and verified against the running PostgreSQL database.

## Test plan
**Automated tests (all passing):**
- [x] `npx tsc --noEmit`: TypeScript compilation check completed with exit code 0.
- [x] `npm run build`: Production build succeeded with exit code 0.

**Manual checks (local):**
- [x] Tested GET `/api/v1/master/company/me` and confirmed all new fields (bank, UPI, email, PAN, CIN, MSME, ISO, brand name, website) are returned correctly in the response JSON.
- [x] Tested PATCH `/api/v1/master/company/:id` with bank and invoice payloads, confirming database persistence in `core.company`.
