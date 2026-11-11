# 2026-10-12 - work-order-print-and-scan (backend)

## Summary

The Work Order PDF needs to show a produced quantity for each size row, but the work order API did not say which booking size each piece was cut from, or which stages it had finished. This PR adds those two details to each piece in the work order response.

I did not run this against the live API. I checked the data in the database with a read-only query, but I did not call the endpoint after the change. See "Test plan".

## What changed

| File | Change |
|---|---|
| `src/modules/work-order/work-order.types.ts` | `WorkOrderPiece` has two new fields: `orderBookingSizeId` (the booking size row the piece was cut from) and `completedStageCodes` (the work center codes of the stages the piece has finished). |
| `src/modules/work-order/work-order.service.ts` | `toWorkOrderPiece` fills the two new fields. `orderBookingSizeId` comes from the work order item row. `completedStageCodes` lists the stages whose status is completed or skipped. |

## Why

The frontend counts finished pieces for each size row to fill Produced Qty on the printed work order. To do that it has to know which size row each piece belongs to. A piece's size was only available as width, height and counts, which can match more than one row.

## How

**Size link**: each piece is a work order item that already stores `order_booking_size_id`. The service now returns it.

**Finished stages**: the list uses the same "done" rule as the rest of the service (completed or skipped), so skipped Dispatch and HOLE rows from the earlier stage change count as done.

**No query changes**: both values come from rows the service already loads, so there is no new query.

## Decisions made

**`completedStageCodes` was added but the frontend no longer uses it.** The first version of Produced Qty counted each stage separately. It was then changed to one column that counts finished pieces, which only needs `orderBookingSizeId` and the existing piece status. The list was left in because it is cheap and harmless.

## Bugs found while testing (already there before, not fixed here)

**1. Work center codes differ between companies.** One company has codes like `CUT12141011` and `TMP12141011`, others use `CUT` and `TMP`. The frontend matches stage codes by prefix because of this, so a code that does not start with the expected letters would not be recognised.

## Notes

- A read-only check of completed work orders in the database showed that every piece has a size link and finished stage rows, for example `GRN` on `WO-20261008-105`.
- The backend needs a restart. Without it, the frontend does not receive the new field and Produced Qty shows 0.
- A teammate's change in `src/modules/order-booking/order-booking.repository.ts` is not part of this work, and this document does not cover it.
- This builds on the backend change in the 2026-10-10 document, which removes Dispatch and HOLE from the stages.

## Next steps

- Remove `completedStageCodes` if nothing else needs it.
- Add a test that checks a piece response carries its size id and finished stages.

## Test plan

**Automated tests (no test files for this module):**
- [x] `npx tsc --noEmit -p .`: passed after the change.

**Manual checks (local):**
- [ ] `GET` a completed work order and check that each piece has `orderBookingSizeId` and `completedStageCodes`. Not checked, the API was not called.
- [ ] Complete one piece and check that its finished stages are listed. Not checked.
