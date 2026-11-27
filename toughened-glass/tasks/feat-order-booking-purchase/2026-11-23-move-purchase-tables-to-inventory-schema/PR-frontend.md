# 2026-11-23 - move-purchase-tables-to-inventory-schema (frontend)

## Summary

There are no frontend code changes in this PR. This PR moved 15 purchase and procurement tables from the public schema to the inventory schema in the backend and database. Frontend screens, API contracts, and behaviour are unaffected.

## What changed

No frontend files were touched in this PR.

## Test plan

**Automated tests:**
- [x] Frontend type check (`npx tsc -b`): passes with no errors.

**Manual checks (local):**
- [x] Verified procurement and purchase screens continue to load data seamlessly when backend is updated.
