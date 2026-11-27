# 2026-10-14 - clickable-cross-document-links (frontend)

## Summary

Document numbers in the list tables now link to the record they refer to. Each link opens the existing view, so there are no duplicate views. This PR also fixes the Enquiry No column in the Order Booking table, which was empty for every row.

## What changed

### New cross-document navigation links

| Table | Column | Opens |
|---|---|---|
| Order Booking | Enquiry No | Enquiry form (read-only) |
| Order Booking | PI No | PI details drawer |
| PI Status | Order No | Booking Details page (`/orders/$bookingId`) |
| PI Status | Work Order | Work Order summary modal |
| Work Orders | Booking No | Booking Details page |
| Work Orders | PI No | PI details drawer |
| Final Invoices | Work Order | Work Order summary modal |
| Final Invoices | PI | PI details drawer |
| Dispatch | Invoice No | Final Invoice details modal |
| Dispatch | Work Order | Work Order summary modal |

### Frontend changes

- Link buttons use the same cyan chip style as the existing PI Number and Order No buttons. Clicks call `stopPropagation`, so row click actions are unaffected.
- New `PIStatusDrawerByCode` opens the existing PI drawer from a PI code alone (Final Invoice and Dispatch rows only carry the code).
- Dispatch type and API mapper gain `orderBookingId`.
- The PI Status Order Booking modal is replaced by navigation to the Booking Details page. The old modal, its import and its state are commented out, not deleted.
- Fixed a wiring bug in the PI Status page: `onSelectWorkOrder` pointed at the PI drawer's setter, so the Work Order modal could never open.

## Notes

- The Work Order summary is view-only when opened from PI Status, Final Invoices or Dispatch (`canApprove={false}`). Approving stays on the Work Orders page.
- The Work Orders page finds the PI for its PI No link in its list of confirmed PIs. If a PI isn't in that list, the click does nothing.
- Rows with no linked record show plain text or ", " instead of a link.

## Test plan

**Automated tests:**
- [x] Frontend type check (`tsc --noEmit`): passes.

**Manual checks (local):**
- [ ] Verify each table link opens the right record modal or drawer in the running application.
