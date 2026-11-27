# 2026-11-06 - update-labels-and-bank-upi (backend)

## Summary

Bank accounts can now be marked as accepting UPI, and UPI payments in Party Approval and Add to Stock can now record the sender and receiver bank account (before, only the two UPI ids were stored). This repo adds a `upi_enabled` flag on bank accounts and lets the UPI payment mode accept account ids.

The Save to Update label work is frontend only, so nothing in this repo changed for it.

**Filtering the payment accounts by the UPI flag is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/125_bank_account_upi.sql` (new) | Adds `upi_enabled BOOLEAN NOT NULL DEFAULT FALSE` to `master.bank_account`. Safe to run again. |
| `src/modules/master/bank-account/bank-account.types.ts` | Row and API types carry `upi_enabled` and `upiEnabled`. |
| `src/modules/master/bank-account/bank-account.schema.ts` | The bank account input accepts `upiEnabled` (default false). |
| `src/modules/master/bank-account/bank-account.repository.ts` | Insert, update and the API mapping read and write `upi_enabled`. |
| `src/modules/payment-detail/payment-detail.service.ts` | For the UPI mode, the sender and receiver account ids are now accepted and checked against the party's and the company's own accounts, and saved with the approval like a bank transfer. |
| `src/modules/inventory-purchase/inventory-purchase.service.ts` | Same for Add to Stock: UPI now accepts the company sender and supplier receiver account ids and checks them. |

## Why

Needed so the new UPI checkbox is saved, and so the UPI account dropdowns in the frontend have somewhere to store what was picked. The code change was small. The migration is the part that needs care.

## How

**Flag only**: `upi_enabled` is a yes or no. No UPI id or UPI name is stored on the bank account.

**Accounts for UPI**: the existing account checks (the account must belong to that party, supplier or company) now run for UPI too. The two typed UPI ids and the transaction id and date rules did not change.

## Decisions made

**The UPI flag does not filter the payment accounts.** I first made the payment dropdowns accept only UPI ticked accounts, then reverted it, so any account of the party, supplier or company is accepted for UPI. This can be tightened later.

**Account ids are optional for UPI on the server.** The frontend asks for them when the party, supplier or company has accounts, but the server does not force them, so older callers keep working.

## Notes

- Type check (`tsc --noEmit`) in the backend passes.
- The `upi_enabled` column was already present in the local database when I checked, so migration 125 has been applied there. Other environments still need to run it.
- This repo had other uncommitted changes from other work (dispatch, customer lot, work order inventory). They are not part of this PR and are not listed above.

## Next steps

- Run migration 125 in every environment before deploying.
- Decide whether UPI payments should only allow accounts with the UPI flag set.

## Test plan

**Automated tests (no test files added):**
- [x] `tsc --noEmit` in the backend: passes.
- [ ] `npm test`: not run for this change.

**Manual checks (local):**
- [x] Checked the local database has the `upi_enabled` column on `master.bank_account`.
- [ ] Save a bank account with UPI ticked and read it back - not done by hand.
- [ ] Approve a party payment and add a purchase to stock using UPI with accounts selected - not done by hand.
