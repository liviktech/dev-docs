# 2026-09-29 - enquiry-workflow-enhancements

## Summary

This PR builds full CRUD, manager approval workflows, and one-click order booking conversion for the Customer Enquiry module across both frontend and backend repositories. Previously, the Enquiry module was read-only with GET endpoints only, and the frontend showed a basic table with no creation, edit, approval, or order conversion capabilities. With these changes, staff can record customer enquiries, managers can approve open enquiries, users can convert approved enquiries directly into Order Bookings with auto-filled details, and non-OPEN enquiries are locked from modification and photo uploads.

**Line item specification at enquiry creation is not included on purpose** - see "Decisions made" below. This was a choice, not a miss, as size capture belongs to Order Booking.

## What changed

### rubine-toughened-glass-frontend
| File | Change |
|---|---|
| `src/components/layout/app-sidebar.tsx` | Restored Customer Enquiries navigation link under Sales in the main sidebar. |
| `src/features/ai-extraction/enquiry-photos-button.tsx` | Added readOnly prop support and enforced uniform fixed 96px width for slips/photos buttons across table rows. |
| `src/features/ai-extraction/photo-review-panel.tsx` | Implemented readOnly mode with a lock banner, disabled upload and camera tiles, and blocked file processing for locked enquiries. |
| `src/features/ai-extraction/read-from-photo-button.tsx` | Updated screen type and record ID propagation for AI photo extraction. |
| `src/features/enquiries/api.ts` | Added API client functions for creating, updating, approving, and updating status of enquiries. |
| `src/features/enquiries/enquiries-page.tsx` | Redesigned Customer Enquiries page to match Order Booking UI layout. Added header KPI cards with uppercase descriptions, segmented status filter tabs with counts, date pickers, party dropdown, right-aligned SearchBar, locked photo access for non-OPEN enquiries, uniform 96px status badges, and direct DataTable rendering. |
| `src/features/enquiries/enquiry-form-modal.tsx` (new) | Added modal component for creating, editing, and viewing enquiries with party selection, project filtering, sales rep assignment, customer reference, and non-OPEN lock banner. |
| `src/features/enquiries/queries.ts` | Added TanStack Query options for fetching enquiry lists, detail records, and sales reps. |
| `src/features/order-booking/api.ts` | Updated order booking API contracts and payload types. |
| `src/features/order-booking/booking-header-form.tsx` | Added auto-filling of customer, project, and reference details when navigating from an approved enquiry via enquiryId route search parameter. |
| `src/features/order-booking/booking-page.tsx` | Handled pre-selected enquiry route query parameters during booking creation. |
| `src/routes/orders_.new.tsx` | Wired enquiryId search params to route schema for one-click enquiry-to-order navigation. |

### rubine-toughened-glass-backend
| File | Change |
|---|---|
| `src/database/migrations/021_enquiry_enhancements.sql` (new) | Created migration adding customer_reference, sales_rep_id, and APPROVED status constraint to public.enquiry table. |
| `src/modules/enquiry/enquiry.controller.ts` | Implemented express route controllers for enquiry creation, updates, manager approvals, and status transitions. |
| `src/modules/enquiry/enquiry.repository.ts` | Added SQL repository methods for inserting, updating, fetching, and approving enquiries with joined photo counts. |
| `src/modules/enquiry/enquiry.routes.ts` | Configured protected API endpoints for POST /api/enquiries, PATCH /api/enquiries/:id, and PATCH /api/enquiries/:id/approve with role-based auth. |
| `src/modules/enquiry/enquiry.schema.ts` | Added Zod validation schemas for enquiry creation, updates, and approval payloads. |
| `src/modules/enquiry/enquiry.service.ts` | Added business logic and status validation preventing updates to non-OPEN enquiries. |
| `src/modules/enquiry/enquiry.types.ts` | Defined TypeScript types for Enquiry database rows, DTOs, and query filters. |
| `src/modules/order-booking/order-booking.service.ts` | Updated order booking creation logic to automatically set linked enquiry status to CONVERTED upon booking creation. |
| `src/modules/ai-extraction/ai-extraction.service.ts` | Added status check rejecting photo uploads for enquiries that are not in OPEN status. |

## Why

The Enquiry module previously operated as a read-only list with backend GET endpoints only. Sales personnel could not log new customer enquiries, managers could not approve them, and there was no seamless way to transition an approved enquiry into an Order Booking. This update completes the end to end enquiry workflow, enforces strict lifecycle locking once approved or converted, and unifies the Enquiry UI design with Order Booking.

## How

**Backend CRUD and Approval Endpoints**: Created POST /api/enquiries for recording header details (party, project, enquiry date, customer reference, sales rep, remarks) with status defaulting to OPEN. Added PATCH /api/enquiries/:id/approve gated to SUPER_ADMIN/ADMIN roles to transition status from OPEN to APPROVED.

**Automatic Conversion Tracking**: Updated order-booking.service.ts so that creating an Order Booking with bookingType='ENQUIRY' and a linked enquiry_id automatically marks that enquiry status as CONVERTED.

**One-Click Order Creation**: Configured the frontend to pass the enquiry ID via URL query parameter (/orders/new?enquiryId=...) when clicking Create Order on an APPROVED row. BookingHeaderForm detects the parameter on mount and auto-fills customer, project, and reference details.

**Strict Lifecycle Locking**: Enforced status guards in backend services (enquiry.service.ts and ai-extraction.service.ts) and frontend components (enquiry-form-modal.tsx, photo-review-panel.tsx, enquiries-page.tsx) so that non-OPEN enquiries cannot be edited and cannot accept new photo uploads.

**UI Alignment**: Restructured enquiries-page.tsx with KPI summary cards, FilterTabs for status counts, date range pickers, party filter, SearchBar, uniform 96px status badges, and 96px slips/photos buttons, matching Order Booking table layout.

## Decisions made

**Enquiry creation remains header-only.** We intentionally did not add product size capture to enquiry creation because glass size and item breakdown belong strictly to the Order Booking stage.

**Approval action gated to SUPER_ADMIN and ADMIN.** Role gating was assigned to SUPER_ADMIN/ADMIN for now to prevent blocking on future 9-role authorization split work.

**Disabled photo upload tiles instead of hiding them.** In read-only mode for non-OPEN enquiries, photo upload tiles are displayed in a disabled state with a top lock banner rather than hidden completely, matching the Enquiry View modal lock presentation.

## Notes

- Migration `021_enquiry_enhancements.sql` was executed locally against PostgreSQL and verified.
- Both frontend and backend compile cleanly with zero TypeScript errors.

## Next steps

- Revisit role-based approval permissions once the full 9-role permission model lands.

## Test plan

**Automated tests (All passing):**
- [x] `rubine-toughened-glass-backend`: `npx tsc --noEmit` compiled with 0 errors.
- [x] `rubine-toughened-glass-frontend`: `npm run build` compiled with 0 errors.

**Manual checks (local):**
- [x] Created a new customer enquiry with party, project, date, customer reference, sales rep, and remarks, verifying creation status is OPEN.
- [x] Uploaded photo slips to an OPEN enquiry, verifying photos attach correctly.
- [x] Approved an OPEN enquiry as ADMIN, verifying status changes to APPROVED and Edit action locks.
- [x] Clicked Create Order on an APPROVED enquiry, verifying navigation to `/orders/new?enquiryId=...` and auto-filling of header fields.
- [x] Completed order booking creation, verifying linked enquiry status automatically updated to CONVERTED.
- [x] Opened Slips / Photos dialog on APPROVED and CONVERTED enquiries, verifying lock banner appears, photo upload tiles are disabled, and file uploads are blocked.
- [x] Tested search bar, status tabs, date filters, and party filter on the redesigned Enquiry page, verifying real-time table filtering.
