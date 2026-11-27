# 2026-10-21 - pi-close-reopen-auto-close (backend)

## Summary

A proforma invoice (PI) could only be Pending, Approved, Declined, Cancelled or deleted. There was no way to close a PI that is going nowhere, no way to bring it back, and nothing that cleaned up PIs the party never approved. This PR adds a `CLOSED` status, close and reopen endpoints, and a daily job that closes PIs which did not get party approval in time. The existing approval flow is not changed.

Not checked live yet: the migration has not been run against a database and the endpoints and job have not been called. Only the TypeScript build was run.

**A closed PI does not send its order booking back to "Invoice Pending" on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/075_pi_close.sql` (new) | Allows the `CLOSED` status and adds `closed_at`, `closed_by`, `close_reason`, `closed_automatically` and `reopened_at` to `invoice.proforma_invoice`. |
| `src/modules/proforma-invoice/pi-auto-close.config.ts` (new) | One place for the auto close rules: the 50% share, the bands (4, 7, 14 days), the reason text and the run interval. |
| `src/modules/proforma-invoice/pi-auto-close.job.ts` (new) | Runs the auto close check once at server start and then every 24 hours. Errors are logged and do not stop the server. |
| `src/modules/proforma-invoice/proforma-invoice.repository.ts` | New `closeProformaInvoice`, `reopenProformaInvoice` and `listPisAwaitingPartyApproval`. The list queries now return the closed fields, and the status counts include `CLOSED`. |
| `src/modules/proforma-invoice/proforma-invoice.service.ts` | New `closeProformaInvoice`, `reopenProformaInvoice` and `autoCloseOverduePIs`. |
| `src/modules/proforma-invoice/proforma-invoice.controller.ts` | Handlers for close and reopen. |
| `src/modules/proforma-invoice/proforma-invoice.routes.ts` | `PATCH /:id/close` and `PATCH /:id/reopen`, using the same permission as the status update. |
| `src/modules/proforma-invoice/proforma-invoice.schema.ts` | `CLOSED` allowed in the list status filter. New close body schema (optional reason). |
| `src/modules/proforma-invoice/proforma-invoice.types.ts` | `CLOSED` added to the status type. Closed fields added to the list row. |
| `src/server.ts` | Starts the job after the server starts and stops it on shutdown. |

## Why

Users wanted to close PIs by hand, bring them back, and have stale PIs closed for them. The close and reopen endpoints were simple. The real work was the auto close rule, because the advance is not known before party approval.

## How

**Close**: only a PI with status Pending, Declined or Cancelled can be closed. A fully approved PI (status Confirmed) and an already closed PI are refused with a clear message. The check is inside the update query itself, so it cannot be bypassed by calling the API directly. This means a PI can be closed before party approval, or after party approval but before admin approval.

**Reopen**: only a closed PI can be reopened. It goes back to Pending, the closed details are cleared, and `reopened_at` is set. Party approval is kept if it was given before the close.

**Auto close**: the job reads every Pending PI that has no party approval, and closes it if it is older than the days allowed for its amount. The age is counted from `reopened_at` if there is one, otherwise from `created_at`. Closing again goes through the same guarded query, so a PI that got approved while the job was running is skipped.

**Days allowed**: the amount used is always 50% of the PI total, whatever advance was typed at party approval. Under 50,000 gives 4 days. 50,000 to 1,00,000 gives 7 days. Above 1,00,000 gives 14 days. Exactly 50,000 and exactly 1,00,000 fall in the 7 day band.

**Kept the same on purpose**: the party approval flow and the status update endpoint are not changed. Giving party approval does not run any days check. It just stops the clock, because the job only looks at PIs with no party approval.

## Decisions made

**Always 50% of the PI total, not the typed advance.** The advance is only typed at party approval, so it is not known while the PI is waiting for that approval. The PI total is used with tax included.

**Clock restarts on reopen.** Without this, an auto closed PI would be closed again on the next job run. The days count again from the reopen date.

**A daily timer inside the server, not a cron service.** There was no scheduler in the backend. A simple interval plus a run at server start is enough. It is `unref`ed so it never keeps the process alive. If the server runs as several instances, each one will run the job, which is safe because the close query is guarded.

**Declined and Cancelled PIs can also be closed.** The only rule asked for was that fully approved PIs cannot be closed.

**Order booking module not changed.** Delete sends the booking back to "Invoice Pending" because it removes the PI. A closed PI still exists, so its booking still sees it. This was left alone until it is decided what a booking should show for a closed PI.

**Existing overdue PIs are closed on the first run.** There is no cutoff date, as agreed.

## Notes

- The auto close reason text is stored on the PI so it can be shown in the list.
- A PI that is slightly past its limit can still get party approval before the job runs, because approval is not blocked.
- The files use CRLF line endings like the rest of the repo.

## Next steps

- Decide what an order booking should show when its PI is closed.
- Consider a cutoff date or a dry run before the first job run in production, since it closes every overdue PI at once.

## Test plan

**Automated tests (no tests added, build only):**
- [x] `npx tsc --noEmit`: passes with no errors.

**Manual checks (local):**
- [ ] Run migration 075 and close a Pending PI, once before party approval and once after party approval but before admin approval - not run yet, needs a database.
- [ ] Call close on a fully approved PI and expect a refusal - not run yet.
- [ ] Reopen a closed PI and expect it back in Pending with the closed fields cleared - not run yet.
- [ ] Create PIs with totals of 80,000, 1,50,000 and 3,00,000, set their created dates 3, 6, 10 and 15 days back, run the job, and check only those without party approval and past 4, 7 or 14 days are closed - not run yet.
- [ ] Check a PI with party approval is never auto closed, and that a reopened auto closed PI is not closed again straight away - not run yet.
