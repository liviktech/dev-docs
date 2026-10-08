# 2026-10-07 - enquiry-datepicker-refinements (backend)

## Summary
This PR widens the enquiry source field in the database and updates validation in the enquiry schema to support custom lead source names. Previously, enquiry sources were constrained to an enum with only PHONE and EMAIL, and the database column was limited in size. This change alters sales.enquiry.enquiry_source to VARCHAR(100) and updates Zod validation to allow arbitrary trimmed strings up to 100 characters.

## What changed
| File | Change |
|---|---|
| `src/database/migrations/050_enquiry_source_length.sql` (new) | Alters column enquiry_source in sales.enquiry to VARCHAR(100) to support custom and longer lead source names. |
| `src/modules/enquiry/enquiry.schema.ts` | Replaced strict PHONE and EMAIL enum validation on enquirySource with an optional trimmed string up to 100 characters. |

## Why
Sales teams receive enquiries from diverse channels such as WhatsApp, walk-in visits, social media, and customer referrals. Restricting the field to only PHONE and EMAIL prevented capturing accurate source channels, and saving custom sources caused schema validation errors.

## How
**Database column expansion**: Added migration 050_enquiry_source_length.sql altering sales.enquiry.enquiry_source to VARCHAR(100), ensuring existing data is preserved while accommodating longer source names.
**Validation schema update**: Updated createEnquirySchema in enquiry.schema.ts so enquirySource accepts an optional trimmed string up to 100 characters.

## Decisions made
**Used free-form string validation instead of a rigid database enum.** Lead sources vary across businesses and marketing campaigns, so allowing strings up to 100 characters gives the frontend flexibility to introduce custom sources without requiring database schema migrations each time.

## Notes
- Verified TypeScript compilation and build live with npm run build (tsc), passing cleanly with exit code 0.

## Test plan
**Automated tests (all passing):**
- [x] `npm run build`: TypeScript build completed with exit code 0.

**Manual checks (local):**
- [x] Verified Zod schema validation passes with custom strings like WHATSAPP and Walk-in up to 100 characters.

