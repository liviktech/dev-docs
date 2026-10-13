# 2026-10-09 - ui-consistency-and-pi-details (backend)

## Summary

The Proforma Invoice screens had no way to save changes to a PI that already exists: the API could only create one. This PR adds a `PUT /proforma-invoice/:id` route so an existing PI can be updated. The create function now also works as an update when it is given the code of a PI that already exists.

I did not write this change. It came in with a teammate's commit (`42fb385`, "fix: bug fixes and ui modifications") that is on this branch, and this document is based on reading that diff. I did not run it. See "Test plan".

## What changed

| File | Change |
|---|---|
| `src/modules/proforma-invoice/proforma-invoice.routes.ts` | New `PUT /:id` route. It needs the `proforma_invoice` EDIT permission and uses the same validation as create. |
| `src/modules/proforma-invoice/proforma-invoice.controller.ts` | New `updateProformaInvoice` handler. It calls the create service with the PI code from the URL and returns "Proforma invoice updated successfully". |
| `src/modules/proforma-invoice/proforma-invoice.repository.ts` | `createProformaInvoice` now checks whether the given PI code already exists. If it does, it deletes the PI's payment terms, taxes, charges and items, updates the header, and re-creates the child rows. If it does not, it generates a new code and inserts, as before. |

## Why

The PI Status table in the frontend has an Edit action for a PI, which needs a route to save the changes. I did not confirm from the frontend code that it calls this route.

## How

**One function for both**: create and update share `createProformaInvoice`. A PI code in the request (as `piCode`, `pi_code` or `id`) that matches an existing row for the company makes it an update. Otherwise a new code is generated, so create behaves as before.

**Children are rebuilt**: on update the old payment terms, taxes, charges and items are deleted, and the rest of the function inserts them again from the request and the selected bookings.

## Notes

- **Not in a transaction.** The service calls `createProformaInvoice` without a database client, so each query runs on its own connection from the pool. If the re-insert fails after the delete, the PI could be left without its items. The approve and delete flows in the same service do use `withTransaction`. This is worth fixing before relying on it.
- On update the status is not changed (the `UPDATE` does not set `status`), so editing a PI keeps its current status.
- The update route reuses the create validation, so a body that is valid for create is needed even when only one field changes.

## Next steps

- Run the update and the delete and re-insert inside `withTransaction`, as the approve and delete flows do.
- Add a test that updates a PI and checks its items, charges and taxes.

## Test plan

**Automated tests (no test files for this module):**
- [ ] Not run. There are no tests for the proforma invoice module, and I did not run the type check on the backend after this commit.

**Manual checks (local):**
- [ ] `PUT /proforma-invoice/:id` with a valid body updates the header and rebuilds the items. Not checked.
- [ ] A user without EDIT permission gets a 403. Not checked.
- [ ] Creating a PI still generates a new code. Not checked.
