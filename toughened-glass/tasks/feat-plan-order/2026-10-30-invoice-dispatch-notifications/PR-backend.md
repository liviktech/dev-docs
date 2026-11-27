# 2026-10-30 - invoice-dispatch-notifications (backend)

## Summary

The notification module read one notification service API key from `.env`, so every company shared it. Creating a PI queued no mail on some databases, and a final invoice or an approved dispatch sent nothing at all. This PR stores the key per company, repairs the PI notification setup, and adds mail for the final invoice and the dispatch.

- **Per-company key.** `core.company.api_key` is set from the superadmin Companies page. The notification provider reads it from the database on every call, and `NOTIFICATION_SERVICE_API_KEY` is gone from `.env`.
- **PI repair.** An older `073` had already been applied on some databases before it was extended, so they had no `pi_created` trigger and no reminder tables (`relation "notification.reminder_slab" does not exist`). Migration `104` brings them level.
- **Final invoice.** On creation the customer and the sales rep are mailed, and both are reminded while the invoice awaits party approval. The customer's mail carries the final invoice as a PDF.
- **Dispatch.** When a dispatch is approved the customer is mailed.

**Sales rep mail for the dispatch, an admin screen for the final invoice slabs, and a PDF on the sales rep mail are not included on purpose** - see "Decisions made" below. These were choices, not misses.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/103_company_api_key.sql` (new) | Adds `core.company.api_key VARCHAR(500)`. |
| `src/database/migrations/104_notification_reminders_and_pi_trigger.sql` (new) | Adds `master.party` email, phone and whatsapp, the PI reminder tables and the `pi_created` trigger if missing. No-op on a fresh database. |
| `src/database/migrations/105_notification_final_invoice.sql` (new) | Final invoice reminder tables and two insert triggers (customer and sales rep). |
| `src/database/migrations/106_notification_dispatch_approved.sql` (new) | Trigger on `invoice.dispatch` for the move into `CONFIRMED`. |
| `src/modules/superadmin/*` (repository, schema, service, types) | `apiKey` on create, update and list. Update now accepts `null` for GST, mobile and key (null clears, missing leaves alone). |
| `src/modules/notifications/providers/notification-service.provider.ts`, `notification-provider.interface.ts` | Key is resolved per company. A company without one fails with `COMPANY_API_KEY_MISSING` and is not retried. `getDelivery` and `retryDelivery` take a company id. |
| `src/modules/notifications/notification.container.ts`, `notification.service.ts` | Loads the key with `SELECT api_key FROM core.company`. Exposes `finalInvoiceReminderService`. |
| `src/config/env.ts`, `.env.example` | `NOTIFICATION_SERVICE_API_KEY` removed. Production now needs only the URL and the callback secret. |
| `src/modules/notifications/event-registry.ts` | Five new events: `final_invoice_created`, `final_invoice_created_sales`, `final_invoice_reminder`, `final_invoice_reminder_sales`, `dispatch_approved`. New skip reason `NOT_APPROVED`. |
| `src/modules/notifications/events/notification-source.repository.ts` | `findFinalInvoice` and `findDispatch` lookups. |
| `src/modules/notifications/reminders/final-invoice-reminder.repository.ts`, `.service.ts` (new) | Copies of the PI reminder code pointed at the final invoice tables. |
| `src/modules/notifications/reminders/pi-reminder.job.ts`, `notification.controller.ts`, `notification.routes.ts` | The 60 second job and `POST /notifications/reminders/run` run both passes. New `GET/PUT/DELETE /notifications/final-invoice-reminder-slabs`. |
| `src/modules/notifications/attachments/attachment-builder.ts` | New `final_invoice_pdf` kind, file `Rubine_FI_<code>.pdf`, email only. |
| `src/modules/notifications/attachments/pi-sheet/final-invoice-sheet.loader.ts` (new) | Builds the sheet data for a final invoice from the linked PI plus the invoice's own figures. |
| `src/modules/notifications/attachments/pi-sheet/pi-sheet.loader.ts`, `pi-sheet.types.ts`, `pi-sheet.pdf.ts` | Optional item filter, optional title and number label. PI output is unchanged by default. |
| `tests/notification-service-provider.test.ts` | Uses per-company keys, plus a missing key case. |
| `tests/final-invoice-notifications.test.ts` (new) | Events, loader skips, PDF builder, PDF determinism, reminder service, dispatch event. |
| `docs/NOTIFICATIONS.md` | Per-company key, section 16 (final invoice) and section 17 (dispatch). |

## Why

Each tenant has its own notification account, so one key in `.env` could not work for more than one company. The PI mail failure came from editing an already applied migration. Final invoice and dispatch mail were asked for so customers hear about each step the way they already do for a PI. The real work was the PDF (the PI sheet reused for the items of one invoice only) and keeping the reminders from chasing an invoice that was approved meanwhile.

## How

**Per-company key**: the provider takes a `resolveApiKey(companyId)` function instead of a fixed key. The company is already the tenant key on each call, so no other part of the pipeline changed.

**Event triggers**: the existing `notification.capture_event` function writes one message per active rule, with a 5 second hold so the invoice items are saved before the worker reads them. `capture_event` takes one event code per trigger, so the final invoice has two triggers (customer and sales rep). The dispatch trigger fires only when the status becomes `CONFIRMED` (`OLD.status IS DISTINCT FROM 'CONFIRMED'`). The dedupe key (event, company, code) keeps a postponed then re-approved dispatch to one mail.

**Reminders**: the timing rules in `pi-reminder.schedule.ts` are reused as they are. The final invoice has its own slab table and counter table. It reminds while the invoice is `DRAFT` with no `party_approved_at`, and the cycle starts at `created_at` (there is no reopening).

**PDF**: built at send time, not stored. It lists only the items of that invoice, titled TAX INVOICE, with figures from `taxable_amount`, `gst_rate`, `total_amount` and `collected_amount` (shown as advance). It is byte stable for unchanged data, so a retry matches the service's idempotency check. A build failure is a retryable `ATTACHMENT_FAILED` and the mail is never sent without the file.

**Dispatch skip**: if the dispatch is no longer `CONFIRMED` when the worker prepares the message (postponed again within the hold), it is skipped with `NOT_APPROVED`.

## Decisions made

**Never edit an applied migration.** `073` already contains the PI objects, but databases that applied the older copy need `104` to get them. It does nothing on a fresh database, so it is safe to keep.

**Reminders chase "awaiting party approval".** Chosen over reminding until the invoice is confirmed or paid, because that is the same rule the PI uses.

**The final invoice PDF reuses the PI sheet layout.** Chosen over the browser's e-invoice tax invoice sheet. It has no IRN or QR.

**PDF on the customer mail only.** The sales rep mail is text.

**Dispatch mail goes to the customer only**, as asked. Adding the sales rep is one more trigger and one more event.

**Final invoice slabs have their own table** instead of sharing the PI slabs, so the two can be tuned apart. There is no admin screen for them yet, the defaults in code apply until rows are saved.

## Bugs found while testing (already there before, not fixed here)

**1. Four PI reminder tests fail.** The default slab cases in `tests/pi-reminders.test.ts` fail because `pi-reminder.schedule.ts` carries testing default values (an edit made before this work, 2 minute first reminder for the upper bands). Not touched here. It affects only the default slab values when a company has saved none.

**2. Backend `CLAUDE.md` says "single company".** The code is multi-tenant (`core.company`). Docs only.

## Notes

- Rules are off by default per company, event and channel. Each new event has to be switched on for Email in the admin Notifications events tab. The customer needs an email address, the sales rep needs an email on `master.sales_rep`, and the company needs its API key.
- Migrations `103` to `106` were applied to the local database only (confirmed it was local). They still have to be run with `npm run migrate` on every other environment. `104` matters most on any database that applied the old `073`.
- Existing companies have no key until one is saved on the Companies page, so their sends fail with `COMPANY_API_KEY_MISSING` until then.

## Next steps

- Admin screen for the final invoice reminder slabs.
- Decide whether the sales rep should also be mailed when a dispatch is approved.
- Make the `pi-reminder.schedule.ts` defaults and the four PI tests agree again.

## Test plan

**Automated tests (new and changed notification tests passing, 4 existing PI reminder tests failing, see bug 1):**
- [x] `tests/final-invoice-notifications.test.ts`: 10 cases pass (events, loader skips for cancelled and declined, attachment builder, PDF determinism, reminder due logic, dispatch event ok and skip cases).
- [x] `tests/notification-service-provider.test.ts`: per-company key and the missing key case pass.
- [ ] `tests/pi-reminders.test.ts`: 4 cases fail on the default slab values, see bug 1.
- [x] `npx tsc --noEmit`: clean.

**Manual checks (local):**
- [x] Ran `npm run migrate` on the local database: `104`, `105` and `106` applied.
- [x] Built the final invoice PDF for an existing invoice in a smoke script: it rendered and was identical across two builds. Totals were only checked on test looking data.
- [ ] Created a PI, final invoice and approved a dispatch end to end with a live send - not done. No mail was sent from this work, so the triggers firing on a real approval and the delivered mail are not checked.
- [ ] Reminders firing and stopping after approval - covered only by the unit tests above, not run live.
