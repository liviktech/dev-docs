# 2026-10-29 - location-paging-visit-fixes (backend)

## Summary

The Storage Locations tab loaded every location in one plain list and did the Active, Inactive and All filtering and the search in the browser. The Sales Visits search did not look at the visiting purpose, and the visit picker on the Enquiry form listed every visit, including visits already used by an enquiry. This PR adds a paged Storage Locations list that returns the rows, the tab counts and the card totals in one call, makes the Sales Visits search match the visiting purpose, and adds a filter so the Enquiry form only gets visits that can still be used.

The backend compiles (`tsc --noEmit` passes with no errors). I did **not call these endpoints against a running server**, so the paged location list and the visit filters were not checked live.

**Sorting on the Storage Locations list is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/modules/master/inventory-location/inventory-location.repository.ts` | New `listInventoryLocationPage`. It counts Active and Inactive locations for the current search, counts the company wide totals for the cards, and reads one page of rows (only code, name, type and active flag) with the status and search applied. |
| `src/modules/master/inventory-location/inventory-location.service.ts` | New `listInventoryLocationPage` that turns the rows into `items` (id, code, name, locationType, isActive) and returns them with `totalCount`, `statusCounts`, `kpis`, `page` and `limit`. |
| `src/modules/master/inventory-location/inventory-location.controller.ts` | `GET /master/inventory-locations` returns the paged result when `page` is sent (with `limit`, `status` and `search`). Without `page` it returns the plain list exactly as before. |
| `src/modules/sales-visit/sales-visit.repository.ts` | The search also matches the visiting purpose name. The purpose table is joined into the totals and the order type count queries so the search works there too. New `forEnquiry` filter keeps only New Enquiry visits that no enquiry uses yet, with `includeVisitId` to keep one visit in the list. |
| `src/modules/sales-visit/sales-visit.schema.ts` | The list query accepts `forEnquiry` and `includeVisitId`. |

## Why

The Enquiry list already pages on the server and sends only the fields it shows. The Storage Locations tab should work the same way, with the tabs, the search and the card totals all coming from one call. The Sales Visits changes come from review: people searched for a purpose and got nothing, and a visit that already had an enquiry (or an order) could be picked again on a new Enquiry.

## How

**One call for the whole tab**: The paged call runs a count query for the current search, a count for the cards (reused when there is no search), and one page query. Tab counts follow the search. The three cards stay company wide so they do not move while someone types.

**Same URL, two shapes**: The old list is used by many dropdowns (finished glass box, receive stock, purchase rows). Adding a separate path would have meant more routing work, so the page parameters switch the response shape on the existing path. No page parameter means the old array.

**Visits that can still be used**: `forEnquiry=true` keeps a visit only when its purpose is New Enquiry and no enquiry has its code. A visit with an order always has an enquiry, so it drops out too. When an enquiry is edited, the frontend sends that enquiry's own visit as `includeVisitId` so it stays selectable.

## Decisions made

**No sorting on the location list.** The rows are ordered by name then code. The tab never had column sorting, so adding it was not needed.

**The New Enquiry rule uses the purpose code `NEW_ENQUIRY`.** It is the code seeded in migration 088. If someone renames the purpose in the database the filter will stop matching, which would empty the Enquiry visit list.

## Notes

- Check the new location call by hand: `GET /master/inventory-locations?page=1&limit=10&status=ACTIVE&search=rack`.
- No migration is needed for this PR.

## Test plan

**Automated tests (no new tests added):**
- [x] `tsc --noEmit` in the backend: passes with no errors.

**Manual checks (local):**
- [ ] Paged location list with each status and a search - not run, no server was started. The frontend tab depends on it.
- [ ] Sales Visits search by a purpose name - not run for the same reason.
- [ ] Enquiry visit picker with `forEnquiry=true` after an enquiry uses a visit - not run for the same reason.
