# 2026-10-01 - order-booking-manual-charges-and-performance (backend)

## Summary

Implements manual charges support, resolves critical P0 backend performance and data truncation issues, and polishes the Order Booking UI.

---

## What changed



## Test plan

**Automated tests:**
- [x] Backend type check (`tsc --noEmit`) passed with 0 errors.
- [x] Verified `GET /full` performs zero write queries on confirmed orders.
- [x] Verified all size and item mutations update charges correctly.

**Manual checks (local):**
- [x] Verified `LIMIT 20` truncation removed from `listOldBookings`.
- [x] Verified atomic transaction wrapping on status update to `CONFIRMED`.
