# 2026-10-01 - order-booking-manual-charges-and-performance (frontend)

## Summary

Polishes the Order Booking UI, adds project selection logic, improves KPI card typography, fixes invoice column artifacts, and adds manual charges preview.

## What changed

- **Project Selection Logic**: Automatically disables the Projects dropdown when the selected Party has no associated projects.
- **KPI Card Typography**: Reduced metric number font size to improve card hierarchy and avoid layout clipping.
- **Invoice Column Fix**: Fixed PI status display in the invoice column (removed blank underline artifact).
- **Manual Charges UI**: Added manual charge row inputs and live charge previews in Add Product modal.

---

## Test plan

**Automated tests:**
- [x] Frontend build (`npm run build`) passed with 0 errors.

**Manual checks (local):**
- [x] Projects dropdown auto-disables when selected Party has no associated projects.
- [x] Metric typography and KPI card layout verified.
- [x] Manual charge row inputs and live previews verified in modal.
