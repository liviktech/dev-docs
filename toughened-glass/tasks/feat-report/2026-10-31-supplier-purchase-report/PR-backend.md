# 2026-10-31 - supplier-purchase-report (backend)

## Summary

The Sundry Creditors report query now returns one row per purchase with the fields from the purchase form, instead of paid and outstanding amounts. It was also leaving out new purchases, because it only listed approved ones and added up an amount that is empty on the one step purchase form. Both are fixed.

`tsc` passes. I did not run the query against a database from here. The user's screen of the report showed purchases with amounts coming through, so the query ran on their setup.

**Paid and outstanding amounts are not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/modules/report/sales-income/sales-income-report.repository.ts` | `CreditorRow` and `findCreditorRows` now return `purchase_no`, `purchase_date`, `supplier_name`, `supplier_invoice_no` and `total_amount`. The status filter and the "balance above zero" filter are gone. The payments join is gone. Total is the lines' `total_cost` plus the purchase's `additional_charges`. Date and supplier filters and the ordering (newest purchase first) are unchanged. |
| `src/modules/report/sales-income/sales-income-report.service.ts` | `CreditorReportRow` and `getCreditorsReport` return `purchaseNumber`, `purchaseDate`, `supplierName`, `supplierInvoiceNumber` and `totalAmount`. |

## Why

The report was asked to show only supplier name, supplier invoice number, purchase number, purchase date and total amount from the purchase form. The old query was built for payables (what is still owed), so it joined payments and hid anything fully paid. A purchase entered on the one step form (migration 101) did not show up in it.

## How

**Total amount**: Same number as the form's Final Invoice Total. Line `total_cost` values are added up per purchase in a sub query before the join, so a purchase with many lines still gives one row. The additional charges are added once on top.

**Supplier invoice number**: Read from `inventory_purchase.supplier_invoice_no`. The old code used the purchase number as the invoice number. `invoice_date` is not returned any more.

**Kept the same**: The route, the query string (`fromDate`, `toDate`, `supplierId`), paging and the unpaged download path did not change. Rows with no supplier invoice number come back as `null`.

## Decisions made

**Dropped the approved only filter.** A purchase saved on the new form can be in a status other than approved, and the report is meant to list purchases from the form. This also means pending purchases (stock not added yet) now appear in the report.

**Dropped paid and outstanding.** The request was for five fields only. The payments table is no longer read by this report.

**Used `total_cost`, not `approved_amount`.** `approved_amount` is only filled during the older approval step, so it was 0 for new purchases.

**Report id and API path kept** (`/reports/sales-income/creditors`) so permissions and any saved links still work.

## Notes

- The frontend setup points to the deployed backend, so the change shows only after this backend is deployed or the frontend proxy is pointed to the local backend.
- The function names (`findCreditorRows`, `getCreditorsReport`) still say "creditors". They were left alone to keep the diff small.

## Next steps

- If an outstanding payables report is still needed, add it as a separate report that reads the payments again.

## Test plan

**Automated tests (type check only):**
- [x] `tsc --noEmit`: no errors
- [ ] No test file covers the creditors report, so none was added or run

**Manual checks (local):**
- [x] User's screen of the Supplier Purchases report: rows for PUR-20261031-124 (Ais Glass, 2,500.00) and several Testing Company purchases came back, newest first
- [ ] Several lines plus additional charges give one row with the right total - not checked
- [ ] Date and supplier filters on the new query - not run by me
- [ ] Pending purchases appearing in the list - not checked directly
