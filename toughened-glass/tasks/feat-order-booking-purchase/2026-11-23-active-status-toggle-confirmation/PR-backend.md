# 2026-11-23 - active-status-toggle-confirmation (backend)

## Summary

There are no backend code changes in this PR. All changes in this PR were implemented in the frontend.

## What changed

No backend files were touched in this PR.

## Why

Accidental clicks on the status toggle button in master tables or edit dialogs were causing immediate backend updates without user verification. Deactivating a master record (such as a tax profile, shift, or work center) can disrupt other business operations, while reactivating obsolete records can re-introduce deprecated options. Adding a clear confirmation step prevents stray clicks and aligns desktop and mobile status updates with standard destructive-action safeguards across the ERP.

---

## How

**Event Bubbling Prevention**:

## Test plan

**Automated tests:**
- [x] Backend type check (`npx tsc --noEmit`): passes with no errors.

**Manual checks (local):**
- [x] Backend API endpoints and database operations verified.
