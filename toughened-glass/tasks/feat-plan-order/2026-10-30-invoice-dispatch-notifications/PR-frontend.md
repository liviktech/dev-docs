# 2026-10-30 - invoice-dispatch-notifications (frontend)

## Summary

The superadmin Companies page had no place for the notification service API key, which is now stored per company instead of in `.env`. This PR adds an API Key field to the Companies table and to the add and edit dialog, and fixes the edit dialog so Save works.

The new final invoice and dispatch events need no frontend change. They are meant to show up on their own in the admin Notifications events tab, because it lists whatever `GET /notifications/events` returns. That was not checked in the running app.

## What changed

| File | Change |
|---|---|
| `src/features/superadmin/types.ts` | `apiKey` added to `Company` (`string \| null`), `CreateCompanyInput` and `UpdateCompanyInput`. |
| `src/features/superadmin/schema.ts` | `apiKey` (trimmed, up to 500 characters, optional). New `updateCompanySchema` picking only name, GST number, admin mobile and API key. |
| `src/features/superadmin/company-dialog.tsx` | API Key input (monospace, `autoComplete="off"`). Defaults and reset include it. The form uses the update schema when editing. Update sends `apiKey \|\| null`, create sends `apiKey \|\| undefined`. |
| `src/features/superadmin/companies-page.tsx` | New "API Key" column, left aligned, showing the last four characters masked as `••••abcd`, or a dash when there is no key. |

## Why

The backend now reads the key from `core.company.api_key` for each company, so the superadmin needs a way to enter and change it. Edit Save was broken because the form validated against the create schema, which requires the hidden admin user fields, so Save did nothing.

## How

**Masked in the table**: only the last four characters are shown, so the key is not readable on screen. The dialog input shows the full value so it can be changed.

**Two schemas**: editing uses `updateCompanySchema`, so the admin user fields that only exist on create are not required. The resolver switches on whether the dialog is editing.

**Clearing the key**: saving an empty key on edit sends `null`, which clears it on the backend. Leaving the key out of a request would leave it unchanged, so the dialog always sends the value.

## Notes

- The key is shown in full inside the edit dialog and returned in full by the companies list endpoint, so only superadmins should be able to reach it. That was not changed here.
- Depends on the backend PR for the `api_key` column and the `apiKey` field in the companies endpoints. Run `npm run migrate` on the backend first.

## Test plan

**Automated tests:**
- [x] `npx tsc --noEmit -p tsconfig.app.json`: clean. It first reported a type error on the resolver in `company-dialog.tsx` (the update and create schemas have different field types), fixed by typing the switching resolver as `Resolver<CreateCompanyFormValues>`.
- [x] `npx eslint src/features/superadmin`: no errors. One warning in `companies-page.tsx` (`useMemo` missing the `handleToggleStatus` dependency), not checked whether it was there before this change.
- [ ] No frontend test covers the Companies dialog, none added.

**Manual checks (local):**
- [ ] Add a company with an API key, edit it, clear it - not run in the browser, so the table column, the masked display and the Save fix are checked by type check only.
- [ ] Final invoice and dispatch events appearing in the Notifications events tab - not run in the browser.
