# 2026-10-14 - clickable-cross-document-links (backend)

## Summary

Document numbers in the list tables now link to the record they refer to. Each link opens the existing view, so there are no duplicate views. This PR also fixes the Enquiry No column in the Order Booking table, which was empty for every row.

## Why

- **Enquiry No was blank in the Order Booking table.** The table-view query, mappers and `OldBookingTableRow` type didn't carry `enquiry_code`, so the frontend column always got nothing. It is now selected and returned.

## What changed

- `order-booking`: select `ob.enquiry_code` in the paged table query. Return `enquiryCode` from both table-view mappers. Add `enquiryCode` to `OldBookingTableRow`.
- `dispatch`: select `wo.order_code AS order_booking_id` in `LIST_SELECT`, and add it to `DispatchListAggregateRow`. The Work Order summary loads by order booking id, and Dispatch rows didn't carry one.

## Test plan

**Automated tests:**
- [x] Backend type check (`tsc --noEmit`): passes.

**Manual checks (local):**
- [x] Verified `order-booking` selects `ob.enquiry_code` in paged table query.
- [x] Verified `dispatch` selects `wo.order_code AS order_booking_id`.
- [ ] Restart backend after deployment.
