# 2026-10-15 - final-invoice-tax-sheet-and-gst (backend)

## Summary
The Final Invoice backend had wrong money figures in several places, and the new e-Invoice and e-Way Bill code threw an internal server error and a "total does not match taxable value" error on real data. This PR fixes the Final Invoice amounts (invoice total, advance share, remaining balance, collected amount), stops a Work Order from staying in "Create Final Invoice" after its product was invoiced, stores the taxable value and GST rate on every Final Invoice so e-Invoice uses the same numbers, removes the HSN validation, and adds server checks for vehicle numbers and Party approval. It also adds the e-Invoice and e-Way Bill modules, the GST provider layer, the new database tables and the access rights they need.

The money fixes were checked against the real database with a dry run and then applied. The e-Invoice and e-Way Bill calls were only run against the MOCK provider. Nothing was run against the Sandbox provider.

**Final Invoices that could not be fixed automatically are not included on purpose** - see "Notes" below. This was a choice, not a miss.

## What changed
| File | Change |
|---|---|
| `src/modules/final-invoice/final-invoice.repository.ts` | The "already invoiced" lookup now treats an old invoice with no item rows as covering the whole Work Order, so finished products do not keep it in "Create Final Invoice". Added the per-invoice advance share and remaining balance in SQL, the collected amount in the list, and the collected KPI. The detail query now returns this invoice's share of the PI charges and discount, and a CGST/SGST or IGST breakdown. `createFinalInvoiceInTx` now saves `taxable_amount` and `gst_rate`. Exports `HOME_STATE_MATCH` and `roundMoney`. |
| `src/modules/final-invoice/final-invoice.service.ts` | Creation caps the amount on the invoice total minus its own advance share. Creation rejects a non-standard GST rate or a total that is more than 2 rupees away from taxable value plus GST (`FINAL_INVOICE_TOTALS_MISMATCH`). Party approval now collects the full remaining balance and ignores any amount sent by the client. The eligible items call returns `advanceRatio`. |
| `src/modules/final-invoice/final-invoice.types.ts` | Added `advance_amount`, `taxable_amount`, `gst_rate` and `discount_amount`. |
| `src/modules/dispatch/dispatch.schema.ts` | Vehicle number on dispatch approval is trimmed, upper-cased, stripped of spaces and dashes, and must match the road vehicle pattern. |
| `src/database/migrations/064_final_invoice_gst_figures.sql` (new) | Adds nullable `taxable_amount` and `gst_rate` to `invoice.final_invoice`. Applied to the database. |
| `src/database/backfill-final-invoice-gst.ts` (new) and `package.json` | Script `backfill:final-invoice-gst`. Dry run by default, `--apply` to write. Stores the figures for consistent invoices and rebuilds draft invoices whose totals were wrong. Writes a backup JSON before every rebuild. |
| `src/shared/gst/gst-codes.util.ts` (new) | GST code helpers, `ROUND_OFF_TOLERANCE` (2) and `DEFAULT_HSN_CODE`. |
| `src/shared/gst/gst-invoice-snapshot.ts` (new) | Builds the e-Invoice data from the Final Invoice. HSN validation is removed, items use the default HSN, and the GST rate and taxable value come from the stored figures when present. |
| `src/shared/gst/gst-invoice-source.repository.ts` (new) | Reads the invoice, items, party and company for the snapshot. Does not select `hsn_code` any more. |
| `src/shared/gst/gst-provider.factory.ts`, `gst-provider.types.ts`, `mock-gst.provider.ts`, `sandbox-gst.provider.ts`, `gst-api-log.repository.ts` (new) | The GST provider layer (MOCK and SANDBOX) and the audit log of every provider call. |
| `src/modules/e-invoice/` (new) | Routes, controller, service, repository, schema, payload and types for generating, reading and cancelling an e-Invoice (IRN). |
| `src/modules/e-way-bill/` (new) | Same set for the e-Way Bill: generate, read, update vehicle, cancel, and a list endpoint for active bills. |
| `src/database/migrations/063_gst_einvoice_eway_bill.sql` (new) | Tables `invoice.gst_einvoice`, `invoice.gst_ewaybill`, `invoice.gst_api_log` and a nullable `master.product.hsn_code`. |
| `src/database/seeds/019_gst_compliance_rbac.sql` (new) | Registers the `e_invoice` and `e_way_bill` modules and grants them to ACCOUNTS and SALES_MANAGER. |
| `src/api/route-mapper.ts` | Mounts the e-Invoice and e-Way Bill routes under `/final-invoices/:id`, plus `/e-way-bills`. |
| `src/config/env.ts` and `.env.example` | New GST settings (`GST_PROVIDER` defaults to MOCK, Sandbox keys and NIC logins are optional). |

## Why
The Work Order stayed in "Create Final Invoice" because old invoices had no item rows, so the code could not tell which products were already invoiced. The amount errors came from every Final Invoice taking the whole Proforma Invoice advance and charges instead of its own share. The "total does not match taxable value" error came from the e-Invoice step recomputing taxable value and GST on its own, so any invoice whose saved total did not follow the active GST rate was rejected. Saving the taxable value and rate at creation makes both sides use the same numbers. The HSN check blocked invoices for a field the product details do not hold.

## How
**Per-invoice advance share**: advance for one invoice is its total times (PI advance / PI total). Remaining is total minus that share, minus the collected amount once Party approval is done. This is done in SQL (`ADVANCE_SHARE_SQL`, `REMAINING_SQL`) so the list, KPIs and detail agree.

**Prorated charges and discount**: the detail query scales PI charges and discount by this invoice's items total over the PI items total. Old whole-Work-Order invoices use a ratio of 1. The e-Invoice snapshot does not scale them a second time, which was a bug that appeared in between and was fixed.

**Stored GST figures**: `taxable_amount` and `gst_rate` are saved at creation. The e-Invoice builder uses them as they are. If they are missing (old rows) it falls back to the nearest valid slab.

**Creation check**: before saving, the service rejects a non-standard GST rate or a total that is more than 2 rupees away from taxable value plus GST, so a bad invoice can never reach e-Invoice.

**Party approval**: the collected amount is always the full remaining balance. The value from the client is ignored. The approval dialog on the frontend shows it read-only.

**Kept the same on purpose**: `master.product.hsn_code` stays in the database as a nullable column. It is just not read or validated any more.

## Decisions made
**Party approval collects the full balance only.** The other option was an editable amount capped at the balance. That would let someone approve with a part amount and leave no way to collect the rest afterwards, so the full balance was chosen.

**HSN uses a default value instead of failing.** The validation was removed as asked and the default HSN `70071900` is sent. The only checks left are the ones based on product details.

**Backfill rebuilds drafts, and leaves the rest alone.** Drafts whose totals were wrong are rebuilt at the active GST rate with a backup saved first. Invoices that were already approved or collected are not touched, because changing them would change money that was already recorded.

## Notes
- Backfill was run with `--apply` on the shared Neon database: 2 invoices got stored figures, 17 drafts were rebuilt, 16 were left alone. The backup is in `backfill-final-invoice-gst.backup.json` in the scratchpad of this session and can be used to restore the old totals.
- The 16 left-alone invoices still do not balance and need a manual cancel and recreate by someone in accounts.
- Migration 063 was already applied when `npm run migrate` was run again. Migration 064 was applied by that run.
- e-Invoice generation could not be completed on some dev data because a party has no pincode and the company GSTIN is not valid. These are data problems, not code problems.
- The `master.product.hsn_code` column is still in the database and still empty for new products.

## Next steps
- Cancel and recreate the 16 invoices the backfill skipped.
- Fill the party pincode and the company GSTIN in dev data before testing the Sandbox provider.
- Update `E_INVOICE_INFO.md`, `E_INVOICE_DEVELOPER_GUIDE.md`, `E_WAY_BILL_INFO.md` and `E_WAY_BILL_DEVELOPER_GUIDE.md`, which still describe the HSN check, the transport mode choice and the old layout.

## Test plan
**Automated tests (no test suite run, type check only):**
- [x] `npx tsc` on the backend: no errors.
- [ ] No unit tests were added for the advance share, the creation check or the backfill. There is no test file for the final invoice service yet.

**Manual checks (local):**
- [x] `npm run backfill:final-invoice-gst` dry run, then `--apply`: counts matched the dry run (2 stored, 17 rebuilt, 16 skipped).
- [x] Re-checked invoice FI-20261015-118 after removing the second proration: the e-Invoice total matched taxable value plus GST.
- [x] Generated an e-Invoice with the MOCK provider on a consistent invoice and saw the IRN saved.
- [ ] Sandbox provider - not run, no keys or NIC logins are set up.
- [ ] Creation check with a deliberately bad total - covered by reading the code, not run through the API.
