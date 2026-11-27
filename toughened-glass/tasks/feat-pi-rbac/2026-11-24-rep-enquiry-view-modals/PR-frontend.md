# 2026-11-24 - rep-enquiry-view-modals (frontend)

## Summary

Two things were missing. A sales rep who opened New Enquiry had to pick themselves in Assigned Sales Rep by hand, and the master data view modals did not show who created a record, and each one laid out its fields and dates in a different way. This PR fills the rep in automatically on a new enquiry and locks the field, and brings the view modals of the Plant & Operations, Billing & Finance and Settings masters (plus the stock masters, Glass Thickness and Items) to one layout: Name on the left, Created By on the right. It also fixes the three `tsc -b` errors that were failing `npm run build`.

I checked that `npm run build` passes (Vite and `tsc -b`). I did not open the screens in a browser, so the auto fill and the modal layouts have not been seen live.

**Approved At is not included on purpose** - see "Notes" below. This was a choice, not a miss.

**The Dispatch Details "Dispatch Created" field is not included on purpose** - see "Notes" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/enquiries/enquiry-form-modal.tsx` | On a new enquiry, a non-admin user's own Sales Rep record is found and filled into Assigned Sales Rep. The field becomes view-only and the "+ Add New Sales Rep" shortcut is hidden for them. Edit mode, admins and users with no matching rep are unchanged. |
| `src/features/enquiries/api.ts` | The `SalesRep` type now carries `code` and `email` (the endpoint already returned them). |
| `src/features/stock-masters/stock-master-page.tsx` | View modal (Logo and the other stock masters): Name on the left, Created By on the right. Created At and Last Updated removed. Unused date import removed. |
| `src/features/glass-thickness/glass-thickness-page.tsx` | View modal: Created By added before Created At. The two date fields are kept here. |
| `src/routes/admin.work-centers.tsx`, `admin.shifts.tsx`, `admin.tax-profiles.tsx`, `admin.charges.tsx`, `admin.purchase-charges.tsx`, `admin.payment-terms.tsx`, `admin.invoice-terms.tsx` | View modal: Name (or Term Text) on the left, Created By on the right. Created At and Last Updated removed. Unused date import removed. Shifts keeps Start Time and End Time. |
| `src/routes/admin.order-charges.tsx` | Same as above, and the Value field now sits to the right of Type instead of taking a full row. |
| `src/routes/admin.sales-reps.tsx` | View modal: Created By and Created At replace Registered On and Last Updated. |
| `src/routes/admin.sequences.tsx` | View modal: Created By and Created At are now the last two fields. The old Created At and Last Updated rows are gone. |
| `src/features/items/item-view-modal.tsx` | View modal: Created By and Created At added at the end, after Status. |
| `src/features/products/product-dialog.tsx` | Three `<PascalText text=...>` calls changed to `value=...`. The component only has a `value` prop, so `tsc -b` was failing. |
| `src/features/pwa/master/heavy-adapters.tsx` | `PartySavePayload` now uses `NonNullable` around `PartyDialog`'s optional `onSave`. This was the other `tsc -b` error. |

## Why

Reps were creating enquiries under the wrong rep, or no rep, because the field started empty and was open. The view modals were asked to show who created each record, the same way the list tables already do (see the created-by-approved-by PR). The two `tsc -b` errors were already on the branch and blocked `npm run build`.

The easy part was the layout work. The real work was the rep lookup, because nothing in the app links a login to a Sales Rep record.

## How

**Finding the rep**: the form loads `/enquiries/sales-reps` (it already returns code, name and email) and picks the rep whose name or code matches the logged-in user's username or full name. The match ignores case, spaces and punctuation, so the username `arun.kumar` matches the rep "Arun Kumar". It only runs for a new enquiry and for non-admin users.

**Locking the field**: a small effect sets the rep after the form resets itself, so the rep always wins on a new enquiry. The select is disabled when a rep was found.

**Created By in the view modals**: the detail record types do not carry `createdByName`, so each modal reads it from the list row that was clicked (the list rows already have it). While the detail is still loading it shows "Loading...".

**Kept the same on purpose**: the save payload, the edit flow and every list table are untouched.

## Decisions made

**Match on username or full name, not email.** The first version matched on email. That was changed because the logins in the dev data mostly have no email that matches the rep, while names line up. An exact match was not enough either, so it was loosened to ignore case and punctuation.

**No new "sales rep right".** Any non-admin user whose name matches a Sales Rep record is treated as that rep. Adding a right to the permission list was not picked, to keep this small.

**Created By read from the list row.** Adding `createdByName` to every detail type was not picked, because the detail endpoints are not known to return it.

## Bugs found while testing (already there before, not fixed here)

**1. Duplicate usernames break login.** The login lookup (`findUserByUsername` in the backend) matches the username without case and without a company filter, then takes the first row. The dev database had `Bavin` (OPERATOR) and `bavin` (ADMIN) in different companies, so `bavin` with the right password was refused as "Invalid username or password". `mohan` and `vignesh` are also duplicated. It affects anyone whose username is shared across companies. This is backend code, so it is not fixed in this frontend PR.

## Notes

- **Approved At is not shown** on Sequences or Items because those records have no approval data (no approved by or approved at on them). Showing it needs a real approval step in the backend first.
- **The "Dispatch Created" field on the Dispatch Details modal** was added during this work but is not in the final files, because the file was changed on disk afterwards. It is not part of this PR.
- The rep is only locked in the screen. The backend does not force the assigned rep on create, so a direct API call can still set another rep.
- The mobile enquiry form (`src/features/pwa/enquiry/mobile-enquiry-form.tsx`) was not changed. It still lets the rep pick anyone.
- The other master modals (Designations, Company Profile, Admin Configuration, Notifications, Archive) were not looked at.
- In my local dev database I deleted the duplicate `Bavin` (OPERATOR) login to be able to sign in. That was data only and is not part of the code.

## Next steps

- Make the backend set the assigned rep on create for sales rep logins.
- Apply the same auto fill to the mobile enquiry form.
- Fix the duplicate username login lookup in the backend, and clean the duplicate logins (`mohan`, `vignesh`).
- Decide whether Items and Sequences need an approval step before adding Approved At.

## Test plan

**Automated tests (build passing, no unit tests run):**
- [x] `npm run build`: Vite build passes and `tsc -b` passes with no errors. Only the usual "chunk larger than 500 kB" warning shows.
- [ ] No unit test covers the rep auto fill or the view modals. None was added in this PR.

**Manual checks (local):**
- [x] Read the dev database to confirm usernames and Sales Rep names. `arun.kumar` has the full name "Arun Kumar", and the rep is "Arun Kumar", so the loosened match applies. `bavin2` has the full name "Bavin" and the rep is "Bavin".
- [ ] New Enquiry as `arun.kumar` or `bavin2`: the rep is filled and the field is locked. Not opened in a browser, so not seen.
- [ ] New Enquiry as an admin: the field stays editable and empty. Not opened in a browser, so not seen.
- [ ] Open each view modal listed above and check Name on the left and Created By on the right. Not opened in a browser, so not seen. Type checks cover that the fields exist.
