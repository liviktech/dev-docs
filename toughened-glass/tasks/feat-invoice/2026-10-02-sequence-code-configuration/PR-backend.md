# 2026-10-02 - sequence-code-configuration (backend)

**Companion PR**: `rubine-toughened-glass-frontend` - <https://github.com/liviktech/rubine-toughened-glass-frontend/pull/48> (Adds the Sequences Master Data UI, live preview modal, and exposes sequence codes in table views). The backend must merge first because migrations 030, 031, and 032 provide the daily_business_sequence columns, atomic generation functions, and API endpoints that the frontend calls.

## Summary

This PR implements central sequence code configuration and atomic, concurrency-safe code auto-generation across six core modules: Company, Party, Enquiry, Order Booking, Work Order, and Proforma Invoice. It introduces migrations to support configurable prefixes, date patterns, and unconsumed start numbers in daily_business_sequence. A PostgreSQL function with row-level locking generates codes sequentially without collisions. All six module repositories have been updated to auto-generate and insert these human-readable codes on record creation, and historical data was safely backfilled without modifying existing UUID primary keys.

**Promoting business codes to literal primary keys in PostgreSQL is not included on purpose** - see "Decisions made" below. This was a choice to avoid cascading schema disruptions across 51 foreign-key referencing tables, keeping UUIDs as internal primary keys while treating sequence codes as unique, required user-facing identifiers.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/030_sequence_prefix_and_entity.sql` (new) | Expanded entity_prefix to VARCHAR(50) and added entity_type and date_format columns to public.daily_business_sequence. |
| `src/database/migrations/031_backfill_codes_and_enforce_not_null.sql` (new) | Backfilled historical null codes and enforced UNIQUE NOT NULL constraints on enquiry_code, order_code, party_code, work_order_code, pi_code, and company_code. |
| `src/database/migrations/032_daily_business_sequence_has_generated.sql` (new) | Added has_generated boolean flag to daily_business_sequence and created the atomic, row-locking next_entity_business_code() PostgreSQL function. |
| `src/modules/master/sequence/sequence.types.ts` (new) | Defined TypeScript interfaces for sequence rows, configs, and DTOs. |
| `src/modules/master/sequence/sequence.schema.ts` (new) | Added Zod validation schemas for creating and updating sequence configurations. |
| `src/modules/master/sequence/sequence.repository.ts` (new) | Implemented repository queries for listing, upserting, and deleting sequence configurations in daily_business_sequence. |
| `src/modules/master/sequence/sequence.service.ts` (new) | Added business logic, default definitions for all six modules, date pattern formatting, and live sample code generation. |
| `src/modules/master/sequence/sequence.controller.ts` (new) | Added HTTP controllers for sequence CRUD endpoints. |
| `src/modules/master/sequence/sequence.routes.ts` (new) | Registered Express routes for sequence configuration endpoints under /api/v1/master/sequences. |
| `src/modules/master/master.routes.ts` | Mounted sequenceRouter under /sequences in the master routes tree. |
| `src/shared/utils/business-code.util.ts` | Updated generateBusinessCode utility to support options-based calls across all six entity types with atomic next_entity_business_code query execution. |
| `src/modules/enquiry/enquiry.repository.ts` | Auto-generates enquiry_code on createEnquiryRow using the configured ENQUIRY sequence. |
| `src/modules/order-booking/order-booking.repository.ts` | Auto-generates order_code on createOrderBooking and cloneBookingInTx using the configured ORDER_BOOKING sequence. |
| `src/modules/master/party/party.repository.ts` | Auto-generates party_code on createPartyInTx using the configured PARTY sequence, and included party_code in BASE_PARTY_AGGREGATE_QUERY. |
| `src/modules/work-order/work-order.repository.ts` | Auto-generates work_order_code on createWorkOrderTx using the configured WORK_ORDER sequence. |
| `src/modules/proforma-invoice/proforma-invoice.repository.ts` | Auto-generates pi_code on createProformaInvoiceTx using the configured PI_STATUS sequence. |
| `src/modules/master/company/company.repository.ts` | Auto-generates company_code on createCompany using the configured COMPANY sequence. |

## Why

Prior to this work, business codes across tables were either unpopulated or nullable, and there was no mechanism to customize prefixes, starting numbers, or date formats. The database counter table had no entity mapping or date format support, and creation repositories did not automatically populate business code columns. Centralizing sequence configuration and hooking atomic generation into record creation ensures consistent, professional numbering across all business records.

## How

**Sequence counter table expansion**: Added entity_type, date_format, and has_generated columns to public.daily_business_sequence. Migrations 030, 031, and 032 set up the schema and safe historical backfill.

**Atomic generation function**: Implemented next_entity_business_code() in PostgreSQL using SELECT ... FOR UPDATE row locking. Concurrent callers are serialized by row-level locks, eliminating race conditions and duplicate codes.

**Creation hooks**: Each repository creation query now calls generateBusinessCode() before inserting, passing the companyId, entityType, and any explicit transaction client.

**Start number preservation**: Added has_generated flag so that when a sequence is configured with start number 101, the first record generated consumes 101, after which subsequent records increment to 102, 103, and beyond.

## Decisions made

**Kept internal UUID primary keys and made business codes UNIQUE NOT NULL.** Foreign key dependency inspection revealed that core.company(id) is referenced by 51 tables and master.party(id) by 6 tables. Replacing UUID primary keys with text codes would have caused cascading schema disruptions across dozens of child tables. Keeping UUIDs as internal primary keys and promoting business codes to unique, required user-facing identifiers achieves full business code visibility without risking schema instability.

**Added has_generated flag to prevent skipping start number.** If a user configures a sequence with starting number 101, standard counter increment queries would have returned 102 on the first insert. Adding has_generated = FALSE on configuration ensures the first generated record receives 101, and subsequent records increment to 102, 103, and so on.

## Bugs found while testing (already there before, not fixed here)

None.

## Notes

- Checked type safety live with `npx tsc --noEmit`, exit code 0, zero errors.
- Verified that BASE_PARTY_AGGREGATE_QUERY was missing party_code in its SELECT list, which caused party fetches to fall back to UUID slicing. Fixed by including p.party_code in the select query.

## Next steps

- Monitor production sequence counter tables under heavy write load to verify performance of row-level lock serialization.

## Test plan

**Automated tests (all passing):**
- `npx tsc --noEmit`: Type check completed with exit code 0.

**Manual checks (local):**
- Verified sequence code generation across all six modules: Party (`PTY-20261001-101`), Enquiry (`ENQ-20261001-101`, `ENQ-20261001-102`), Order Booking (`ORD-20261001-101`), Work Order (`WO-20261001-101`), PI Status (`PI-20261001-101`), and Company (`CMP-101`).
- Verified sequential atomic increment inside transactions with automatic rollback to ensure database cleanliness.
- Verified that concurrent calls for the same entity type receive unique, ascending sequence numbers without collisions.
