# 2026-10-08 - final-invoice-ui-fixes (backend)

## Summary

Picking a party on the Final Invoice page made `GET /api/v1/final-invoices?partyId=PTY-001` return a 500, so the party dropdown never worked. This PR fixes the list query in the final invoice repository so the party filter works, and lets the search box also find invoices by their FI number. Nothing else in the backend changed.

## What changed

| File | Change |
|---|---|
| `src/modules/final-invoice/final-invoice.repository.ts` | The party filter now matches the PI party code, the master party code or the party name, with explicit `::text` casts, and ignores the value `ALL`. The search box now also matches `final_invoice_code` and `final_invoice_no`. |

## Why

The browser console showed a 500 for the party filter. The page sends the party code (for example `PTY-001`) but the old query only compared it to `pi.party_code`. While fixing it, one of my edits compared against `p.id`, and `master.party` has no `id` column, which also gave a 500. That condition was removed. Finding the cause took a direct look at the table columns with `npm run db:query`.

## How

**Party filter**: One parameter is checked against `pi.party_code`, `p.party_code` and `p.name` (ILIKE), each with a `::text` cast so Postgres does not complain about mixed column types. The value `ALL` skips the filter.

**Search**: The existing search already covered work order code, party name and PI code. It now also covers the FI code and FI number.

## Decisions made

**The `p.id::text` match was dropped.** It looked useful for filtering by party id but the column does not exist, so only code and name are matched.

## Notes

- The change was only type checked. I did not rerun the request after the last edit, so restart the backend and pick a party to confirm.

## Test plan

**Automated tests (no test files for this module, type check only):**
- [x] `npx tsc --noEmit`: no errors.

**Manual checks (local):**
- [x] Checked the column types with `npm run db:query` (`master.party` has `party_code`, `name`, no `id`).
- [ ] Party filter request after the final fix - not run by me, please confirm after restarting the backend.
