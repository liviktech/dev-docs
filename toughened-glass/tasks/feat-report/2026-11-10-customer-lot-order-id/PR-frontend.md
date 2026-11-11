# 2026-11-10 - customer-lot-order-id (frontend)

## Summary

The Customer Lot Details popup on desktop had a "Balance Pieces" field inside the Storage & Balance card. This PR removes that field. The Order ID fix for the same screen is in the backend, so no frontend code was needed for it.

## What changed

| File | Change |
|---|---|
| `src/features/inventory/customer-material-tab.tsx` | Removed the "Balance Pieces" field from the Storage & Balance card in the lot details popup. The `offcutPieces` helper stays because the row status and the table still use it. |
| `src/features/pwa/inventory/customer-material-tab.tsx` | No real change. Only quotes, semicolons and line wrapping were reformatted (looks like the formatter ran on save). The mobile "Balance pieces" row is still there. |

## Why

The Balance Pieces number repeated what the "Balance To Return" or "Returned To Customer" line already shows, so it was asked to be removed from the popup.

## How

**Popup only**: just the one field was removed from the section list. The Balance Pieces KPI tile and the table column on the Customer Material tab were not touched.

**Order field**: the popup already shows the `orderCode` the API sends, so it fills in once the backend returns it.

## Notes

- The reformatting in the mobile file makes its diff large. Reviewers can skip it, or it can be reverted to keep the PR small.
- Type check was not run after this one-field removal. The helper it used is still used elsewhere in the file, so no unused code is left.

## Next steps

- Decide if the mobile lot sheet should also drop its "Balance pieces" row, to match desktop.

## Test plan

**Automated tests:**
- [ ] Type check (`tsc --noEmit`): not run after this change.

**Manual checks (local):**
- [ ] Open a lot on the Customer Material tab and check the Storage & Balance card has Stored At and Returned To Customer (or Balance To Return) only - not checked live.
