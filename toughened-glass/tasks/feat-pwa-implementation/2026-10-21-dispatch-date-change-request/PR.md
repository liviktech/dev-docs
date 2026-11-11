# 2026-10-21 - dispatch-date-change-request

## Summary

An approved dispatch had no way to move its delivery date. If the date had to change, someone edited the invoice by hand and the admin never saw why. This PR adds a request flow on the existing Dispatch screen. An executive asks for a new delivery date and types the reason. The dispatch moves to a new **Postponed** tab. The admin (or a manager) sees the request from a small **Date Requests** button at the top right of the Dispatch screen, reads the details, and accepts or rejects it. Accepting puts the dispatch back to **Approved** with the new date. Rejecting (with a reason) puts it back with the old date.

This replaces an earlier idea of a separate Dispatch Planning tab, which was built and then removed before it reached any database. Nothing from it is left in either repo.

**Mobile dispatch screens (`/m/dispatch`) are not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

### rubine-toughened-glass-frontend
| File | Change |
|---|---|
| `src/features/dispatch/types.ts` | `POSTPONED` added to the dispatch status. New fields for the delivery date and the open request (requested date, reason). New `DateChangeRequest` type for the admin list. |
| `src/features/dispatch/api.ts` | Maps the new fields. New calls: request a date change, list open requests, approve a request, reject a request. |
| `src/features/dispatch/queries.ts` | New query for open requests (only fetched for people who can see the button). New mutations to request and to decide. All of them refresh the dispatch list. |
| `src/features/dispatch/dispatch-page.tsx` | New **Postponed** tab. **Date Requests** button with a count badge in the page header, small size, shown only for admin and manager roles who hold the Approve right. Wires up the two new modals. |
| `src/features/dispatch/components/dispatch-table.tsx` | New **Delivery Date** column (shows the asked-for date in amber while a request is waiting). Status label shows Postponed. New row menu item **Request Delivery Date Change** on approved dispatches. Approve Dispatch is now only offered on pending (draft) rows. |
| `src/features/dispatch/components/request-date-change-modal.tsx` (new) | The executive's form: new date and one free-text reason box. Cancel and Send to Admin buttons use the same style as the Approve Dispatch modal. |
| `src/features/dispatch/components/date-requests-modal.tsx` (new) | The admin's list. Each request shows dispatch, work order, party, old date to new date, the reason, who asked and when. Accept, or Reject with a typed reason. |
| `src/features/dispatch/components/approve-dispatch-modal.tsx`, `gate-pass-modal.tsx` | no real change (only used as the style reference) |

### rubine-toughened-glass-backend
| File | Change |
|---|---|
| `src/database/migrations/073_dispatch_date_change_request.sql` (new) | Adds `POSTPONED` to the dispatch status check. New table `invoice.dispatch_date_request` (one row per request, kept as history, with the decision, who decided and when). A unique index allows only one open request per dispatch. |
| `src/database/migrations/074_dispatch_date_request_free_text_reason.sql` (new) | The reason is now typed, so the reason code column is dropped and the reason text is required. |
| `src/modules/dispatch/dispatch.schema.ts` | `POSTPONED` added to the list filter. New schemas for the request body (`newDate`, `reason` of at least 3 characters), the request id param, and the reject reason. |
| `src/modules/dispatch/dispatch.types.ts` | New status and the delivery date and request fields on the dispatch rows. |
| `src/modules/dispatch/dispatch.repository.ts` | List and detail queries now join the open request and the invoice delivery date. New queries to lock a dispatch, insert and decide a request, move the invoice delivery date, and list open requests. Fixed the "Pending" KPI count so postponed dispatches are not counted as pending. |
| `src/modules/dispatch/dispatch.service.ts` | `requestDeliveryDateChange`, `listDateRequests`, `approveDateRequest`, `rejectDateRequest`. All inside transactions. |
| `src/modules/dispatch/dispatch.controller.ts`, `dispatch.routes.ts` | Four new endpoints (see "How"). |

## Why

Delivery dates do change (material, machine, customer, vehicle), and the admin needs to know why and agree to it. Before this, there was no record and no approval step. The flow follows what was asked for: the executive requests with a reason, the dispatch sits in its own Postponed tab, the admin decides from a modal on the Dispatch screen, and an accepted request sends the dispatch back to Approved.

## How

**Request**: `POST /dispatches/:id/date-request` with `{ newDate, reason }`. Needs the Dispatch "Update Status" right. Only an approved dispatch can be moved. The new date cannot be in the past or the same as the current one. The dispatch goes to `POSTPONED` and one open request row is saved.

**Admin list**: `GET /dispatches/date-requests` returns the open requests with the dispatch, work order, party, old and new date, reason and requester. Needs the Dispatch "Approve" right.

**Decide**: `PATCH /dispatches/date-requests/:requestId/approve` and `/reject` (reject needs a reason). Needs the Dispatch "Approve" right. Approve writes the new date to the final invoice delivery date, so there is still one source of truth for the date. Both put the dispatch back to `CONFIRMED`.

**Route order**: the `date-requests` routes are registered before `/:id` so they are not read as a dispatch id.

**Who sees the button**: the frontend shows Date Requests only to SUPER_ADMIN, ADMIN, MANAGER and any role ending in `_MANAGER`, and only if they hold the Approve right. The backend checks the Approve right only, not the role name.

## Decisions made

**Reason is free text, not a pick list.** A first version had reason chips (production delay, material shortage and so on) plus a remarks box. It was changed to one typed reason, as asked. The reason code column was dropped in a second migration instead of editing the first, because the first one had already been applied on the dev database.

**Reuse the existing dispatch status and tab, not a new screen or module.** The first idea was a separate Dispatch Planning module with its own page. It was removed. This keeps everything on the Dispatch screen with no new rights.

**No new rights.** Requesting uses the existing Dispatch "Update Status" right and deciding uses the existing "Approve" right. The Date Requests button is additionally limited to admin and manager roles on the screen only.

**Rejecting asks for a reason; accepting does not.** The executive needs to know why it was refused. A plain accept needs no explanation.

**Mobile dispatch screens are not changed.** Only the web Dispatch screen has the request flow for now. Mobile dispatch users will see the `POSTPONED` status come through but have no button to request or decide.

## Bugs found while testing (already there before, not fixed here)

**1. Lint errors in two dispatch modals.** `approve-dispatch-modal.tsx` sets state inside an effect (`react-hooks/set-state-in-effect`, one error) and `gate-pass-modal.tsx` has an unused eslint-disable comment (one warning). Both files are untouched by this PR. They affect anyone running `eslint .` on the dispatch folder.

**2. The `CLAUDE.md` in the backend says the app is single company with no `company_id`.** The code and migrations use `company_id` everywhere. New code here follows the code. The note is out of date.

## Notes

- Both migrations (073 and 074) were applied on the dev database. The 074 file runs after 073 whether or not 073 was applied yet.
- A first attempt at the request endpoint returned a 500. The cause was an extra `$7` placeholder left in the insert after the reason code column was removed. It is fixed in `insertDateRequest`. I did not get a confirmed re-test of the button after the fix, so please click Send to Admin once to be sure.
- The dispatch used while checking had no delivery date on its invoice (`null`). The request still works for that case (the old date shows as "Not set"), but accepting it is the first time that invoice gets a delivery date.
- While writing this I found the Date Requests button rule missed the plain `MANAGER` role (it only matched names ending in `_MANAGER`). It is fixed in `dispatch-page.tsx`.
- This branch also contains a separate edit approval (PI change request) feature on `feat/edit-approval`. It is documented in its own PR file and shares no code with this one.

## Next steps

- Add the request and decide flow to the mobile dispatch screens if mobile users need it.
- Decide whether the backend should also limit the decide endpoints by role name, to match the button.
- Fix the two lint findings in the dispatch modals in a small separate PR.

## Test plan

**Automated tests (no test framework in either repo):**
- [x] Backend `npx tsc --noEmit`: passes with no errors.
- [x] Frontend `npx tsc -b`: passes with no errors.
- [x] Frontend `npx eslint src/features/dispatch`: the new files are clean. Two findings remain in the untouched modals listed above.

**Manual checks (local):**
- [x] Database: after `npm run migrate`, `invoice.dispatch_date_request` exists with the expected columns (no reason code, reason text required), and `chk_dispatch_status` allows `DRAFT`, `CONFIRMED` and `POSTPONED`.
- [x] Database: the dispatch lock query runs against a real approved dispatch and returns it.
- [ ] Executive clicks Request Delivery Date Change on an approved dispatch, enters a date and a reason, sends. Dispatch should appear in the Postponed tab. Not confirmed after the placeholder fix.
- [ ] Admin opens Date Requests, sees the details, accepts. Dispatch should return to Approved with the new delivery date on the invoice. Not run.
- [ ] Admin rejects with a reason. Dispatch should return to Approved with the old date. Not run.
- [ ] A user without the Approve right and a non-manager role does not see the Date Requests button. Not run.
- [ ] Sending a second request while one is open is refused. Not run.
