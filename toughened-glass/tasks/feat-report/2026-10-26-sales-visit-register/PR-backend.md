# 2026-10-26 - sales-visit-register (backend)

## Summary

Sales reps had no place to record the customer visits they make. This PR adds a Sales Visit module to the backend: a visit table, an API to list, create, edit and delete visits, and a link from an enquiry back to the visit it came from. Visiting purposes and enquiry sources are now small master tables, so the dropdowns on the forms read from the database instead of a fixed list. The Sales Order Number of a visit is worked out through the enquiry (visit, then enquiry, then order booking), so there is no new link table.

The code compiles (`tsc --noEmit` passes). The new migrations and the seed file were **not run** against a database, and the API was **not called** from a client or a REST tool, so nothing here is checked live yet.

**The Sales Rep Performance report data is not included on purpose** - see "Notes" below. This PR only stores the visit data that report will need. It was a scope choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/087_sales_visit.sql` (new) | Creates `sales.sales_visit` (primary key `company_id` + `visit_code`) with sales rep, customer, visit date, purpose, customer location, GPS latitude, longitude and accuracy, order type (`IMMEDIATE` or `LATER`), expected order date, visit expense and remarks. Adds checks (expected date needed for `LATER`, expense not negative, GPS ranges) and indexes. Adds a nullable `visit_code` column on `sales.enquiry` with a foreign key to the visit. |
| `src/database/migrations/088_sales_visit_purpose.sql` (new) | Creates `master.visit_purpose` and seeds seven default purposes for every existing company. Adds a foreign key from a visit to its purpose. |
| `src/database/migrations/089_enquiry_source.sql` (new) | Creates `master.enquiry_source`, seeds Phone, Email, WhatsApp and Visit for every company, and also copies any other source already typed on existing enquiries so old data still shows. |
| `src/database/migrations/090_remove_walkin_and_other.sql` (new) | Removes the Walk-in enquiry source and the Other visiting purpose from the dropdown tables. A purpose already used by a saved visit is only switched off, not deleted. |
| `src/database/seeds/028_sales_visit_rbac.sql` (new) | Registers the `sales_visit` module with a View right and a Control right. |
| `src/modules/sales-visit/sales-visit.routes.ts` (new) | Routes under `/sales-visits`. View right for reads, Control right for create, edit, delete and for adding a purpose. |
| `src/modules/sales-visit/sales-visit.controller.ts` (new) | Request handlers for list, get, create, update, delete, customer location, list purposes and create purpose. |
| `src/modules/sales-visit/sales-visit.service.ts` (new) | Validates the sales rep, customer and purpose, fills the customer location from the party address when none is sent, and blocks deleting a visit that has enquiries linked to it. |
| `src/modules/sales-visit/sales-visit.repository.ts` (new) | All SQL: filtered, sorted and paged list, order type counts, expense total, enquiry and order lookups, party address lookup, purpose list (adds the defaults for a company that has none) and purpose insert. |
| `src/modules/sales-visit/sales-visit.schema.ts` (new) | Zod rules for the body and the list query. `expectedOrderDate` is required when the order type is `LATER`, and GPS latitude and longitude must come together. |
| `src/modules/sales-visit/sales-visit.types.ts` (new) | Row and response types. |
| `src/api/route-mapper.ts` | Mounts the new router at `/sales-visits`. |
| `src/modules/enquiry/enquiry.schema.ts` | Adds an optional `visitId` to create and update, and a schema for adding an enquiry source. |
| `src/modules/enquiry/enquiry.repository.ts` | Saves and returns `visit_code` as `visitId`. Adds the source list and source insert queries (with default sources for a company that has none). |
| `src/modules/enquiry/enquiry.service.ts` | Checks that a given visit exists before saving. Adds list source and create source (a duplicate name gives a conflict error). |
| `src/modules/enquiry/enquiry.controller.ts`, `enquiry.routes.ts`, `enquiry.types.ts` | New `GET` and `POST /enquiries/sources` routes, placed before `/:id`, and the `visitId` field on the types. |
| `src/modules/master/sequence/sequence.schema.ts`, `sequence.service.ts`, `sequence.types.ts` | Adds a `SALES_VISIT` entity (default prefix `VST`) so the visit code uses the normal Sequences setup. |
| `src/shared/utils/business-code.util.ts` | Adds the same `SALES_VISIT` entity default so code generation works. |

## Why

The client asked for a Sales Executive Visit Register, with a Performance report to follow. The register needs a customer, a purpose, a location, an expected order timing and an expense for every visit. The easy part was the table and the CRUD API. The real work was deciding how a visit relates to an order, because the client's notes left that open.

## How

**Visit to order link**: A visit does not point at an order directly. An enquiry can point back to a visit (`sales.enquiry.visit_code`), and an order booking already points to an enquiry. The list query follows that chain to build the Sales Order Number column, so one visit can show several enquiries and several orders. The existing enquiry and order flow was not changed.

**Customer location**: The registered location is read from the party's default address (then other addresses) and joined into one line. It is saved on the visit as text, so later edits to the party address do not rewrite old visits. The caller can send a corrected value.

**GPS**: The API only stores the position the browser sends (latitude, longitude, accuracy) and the time it was captured. Nothing tracks the rep continuously.

**Master tables for dropdowns**: Purposes and enquiry sources are stored by code, with a name, a sort order and an active flag. Adding a new one makes a code from the name (upper case, underscores). A company created after the migration gets the defaults the first time the list is asked for. `sales.enquiry.enquiry_source` still stores plain text, so existing enquiries were not touched.

**Rights**: The Sales Visit routes use `requirePermission('sales_visit', 'VIEW')` and `'CONTROL'`. Admin and super admin skip the check as everywhere else.

## Decisions made

**Order link goes through the enquiry.** The other choice was a direct visit to order link table. I did not pick it because the client could not say if one visit may have many orders, and the enquiry chain already allows that without a new table. It also means an order booked with no enquiry has no visit.

**Deleting a visit with enquiries is blocked.** The foreign key from enquiry to visit is `RESTRICT`, and the service gives a clear message first. Setting the link to null would have lost the history.

**A used visiting purpose is switched off, not deleted.** Migration 090 removes Other only if no visit uses it, otherwise it just hides it from the dropdown, so old visits keep a valid purpose.

**A separate migration for purposes (088) instead of editing 087.** This works whether or not 087 was already applied somewhere.

**Walk-in is deleted from the sources table.** Enquiries that already have Walk-in as their source keep that text. The enquiry form shows an empty source for those when edited, because the option no longer exists.

## Notes

- Run `npm run migrate` and `npm run seed` in the backend. Migrations 087 to 090 must run in order. After the seed, give the "Sales Visits" right to non-admin roles in Roles and Access.
- The Sales Rep Performance numbers (conversion rate, orders from visits, orders without visits) are not built. The client still has to confirm how an order is credited to a visit and which period conversion is counted in.
- Live GPS tracking is not built. It is a separate feature from saving one position per visit and needs its own scope decision.
- No screen yet to rename, switch off or delete visiting purposes and enquiry sources. Only listing and adding are supported.

## Next steps

- Get the client's answers on order attribution and the conversion period, then add the performance report API.
- Add a Master Data screen for visiting purposes and enquiry sources if the client wants to manage them.

## Test plan

**Automated tests (no new tests added):**
- [x] `npx tsc --noEmit -p .` in the backend: passes with no errors.
- [ ] No unit or integration test was added for the new module. The existing files in `tests/` do not cover it.

**Manual checks (local):**
- [ ] Run migrations 087 to 090 on a database - not done. The migration SQL was written but never executed.
- [ ] Call `POST /sales-visits`, `GET /sales-visits`, `PUT` and `DELETE` - not done. No API call was made.
- [ ] Create an enquiry with a `visitId`, convert it to an order, and check the visit lists the order number - not done. Covered only by reading the query.
- [ ] Check `GET /enquiries/sources` and `GET /sales-visits/purposes` return the defaults - not done.
