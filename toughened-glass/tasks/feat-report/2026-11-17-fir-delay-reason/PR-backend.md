# 2026-11-17 - fir-delay-reason (backend)

## Summary

A FIR (complaint) could be raised at any time after a sale, and nothing showed the invoice or dispatch dates it related to. This PR lets the FIR API read the Invoice Date and Dispatch Date of the linked invoice, and makes the server refuse a FIR that is raised more than 48 hours after the reference date unless a reason for the delay is given. The reason is saved with the FIR and sent back when the FIR is read.

The 48 hour check was tested live against real invoices (on time, late, and late with an empty or whitespace only reason). Saving a late FIR with a valid reason was not run, because the new column is not in the shared database yet (see "Notes").

## What changed

| File | Change |
|---|---|
| `src/database/migrations/143_fir_delay_reason.sql` (new) | Adds a nullable `delay_reason` text column to `sales.fir`. |
| `src/modules/fir/fir.types.ts` | `FirRow` and `Fir` get `delay_reason` / `delayReason`, plus `invoiceDate` and `dispatchDate` on `Fir`. New `FirReferenceDates`, `FirReferenceLinks` and the `FIR_DELAY_LIMIT_HOURS` constant (48). |
| `src/modules/fir/fir.schema.ts` | The create body accepts an optional `delayReason` (up to 2000 characters). New `referenceDatesQuerySchema` for the new endpoint. |
| `src/modules/fir/fir.repository.ts` | The FIR select and insert now carry `delay_reason`. New `findFirReferenceRow`, which finds the invoice and its confirmed dispatch and returns both dates plus the seconds elapsed since the reference moment. |
| `src/modules/fir/fir.service.ts` | New `getReferenceDates` (works out the dates and whether the complaint is late). `buildWrite` now refuses a late FIR with no reason. `getFir` now returns the invoice date, dispatch date and delay reason. |
| `src/modules/fir/fir.controller.ts` | New `getReferenceDates` handler. |
| `src/modules/fir/fir.routes.ts` | New `GET /firs/reference-dates` route (FIR view permission), placed before `/:id` so it is not read as a FIR id. |

## Why

The complaint team needs to see when the goods were invoiced and dispatched while raising or reviewing a FIR, and a complaint that comes in late needs a recorded reason. The check must live on the server, not only in the form, so a late FIR cannot be saved by calling the API directly.

The dates were easy because both already exist (`invoice.final_invoice.invoice_date` and `invoice.dispatch.approved_at`). The real work was choosing the date the 48 hours is counted from, and handling the time zone.

## How

**Which date starts the clock**: A FIR has no complaint type, so one rule is used for all of them. The clock runs from the Dispatch date, because that is when the goods left and the customer could start finding faults. The Dispatch date is the time the invoice's dispatch was confirmed (`invoice.dispatch.approved_at` where status is CONFIRMED), the same source the Order Status report uses. If the invoice has no confirmed dispatch yet, the Invoice Date is used instead. The two dates are never compared to pick the older one.

**Which invoice is used**: The linked Final Invoice if there is one. Otherwise the latest live invoice (not cancelled or declined) of the linked work order. Otherwise the latest live invoice of the linked PI. If the FIR has none of these, there are no dates and no reason is needed.

**Time zone**: The dispatch moment is a timestamp with time zone, so it is compared as it is. The Invoice Date is only a date, so it counts from the start of that day in India time (Asia/Kolkata). The dispatch date shown to the user is also the India time date. The "now" and the elapsed time are read in the same SQL statement as the dates, so they cannot drift apart.

**The 48 hour rule**: Late means strictly more than 48 hours. Exactly 48 hours is still allowed.

**The server check**: `buildWrite` works out the dates for the FIR being saved. If it is late and the reason is empty or only spaces, it throws a 400 with code `FIR_DELAY_REASON_REQUIRED`. If it is not late, any reason sent is ignored and `NULL` is saved, so a reason is only stored when one was needed.

**Left the same on purpose**: An existing FIR still only changes its status and internal remarks. The update endpoint and its schema did not change.

## Decisions made

**One reference date rule for every FIR.** The brief says to use Invoice Date or Dispatch Date by complaint type, but a FIR has no type field. I did not add one. I used Dispatch Date with an Invoice Date fallback for all FIRs. If billing complaints should run from the Invoice Date, that needs a decision and probably a type on the FIR.

**The dates are read live, not saved on the FIR.** Only the reason is stored. The invoice date and dispatch date are read from the invoice and dispatch records each time the FIR is read, so they always match those records. The cost is that if a dispatch is later changed, the dates shown on an old FIR change too. The late or not late result is not re-worked for old FIRs, only the saved reason is shown.

**The complaint time is the moment of saving, not the FIR date field.** The FIR Date field is a plain date the user picks, so it cannot show hours. The check uses the server time when the FIR is saved.

## Notes

- Migration `143_fir_delay_reason.sql` has to be run before this works. The FIR select now reads `delay_reason`, so until `npm run migrate` is run the FIR list and the New FIR screen will fail. The database used here is shared and remote, so the migration was not run as part of this work. Running `npm run migrate` also applies any other pending migrations.
- Checked live (read only, against the shared database): `getReferenceDates` returned the dispatch date where a dispatch is confirmed and the invoice date otherwise. It marked invoices about 17, 20 and 41 hours old as on time, and ones about 90 to 114 hours old as late. With no linked document it returned no dates and not late.

## Test plan

**Automated tests (none added):**
- [ ] No automated test was added for the FIR module in this PR. The backend has no existing FIR test file to extend.
- [x] `npx tsc --noEmit`: passes with no errors.

**Manual checks (local, against the shared database, scripts run with tsx):**
- [x] `getReferenceDates` on the six newest invoices: dates and on time or late result were correct for each (see Notes).
- [x] `getReferenceDates` with no linked document: no dates, not late.
- [x] `createFir` on an invoice more than 3 days old with no reason: refused with the delay reason message.
- [x] `createFir` on the same invoice with a reason of only spaces: refused with the same message.
- [ ] `createFir` on a late invoice with a valid reason, then reading it back with `getFir` - not run, the `delay_reason` column does not exist in the shared database until the migration is run.
- [ ] A FIR raised exactly at 48 hours - not run, the exact boundary cannot be set up from real data. The rule is a strict "more than 48 hours" comparison in `getReferenceDates`.
- [ ] `GET /firs/reference-dates` over HTTP with a token - not run, only the service function was called directly.
