# 2026-11-02 - fir-status-items-accounts (frontend)

## Summary

The FIR page had no priority or status, purchases picked a plain product, bank details were typed on the Company Profile, and the Price List report was one long set of tables with no filter or paging. The product dialog was also split into four tabs with Next and Previous buttons.

This PR adds priority, status and internal remarks to the FIR page, form and report, adds Items and Accounts pages under Master Data, makes the purchase dialog pick an Item, shows bank details on the Company Profile as a read-only list taken from Accounts, turns the Price List into a filterable paged table, and puts the product dialog on one scrolling page.

`tsc --noEmit` passes. The screens were **not clicked through in the browser**, see "Test plan". The backend and migration 112 must be in place first.

## What changed

| File | Change |
|---|---|
| `src/features/fir/fir-options.ts` (new) | FIR status and priority types, option lists and label helpers |
| `src/features/fir/api.ts`, `src/features/fir/queries.ts` | New fields, `statusCounts`, status and priority filters. Update takes only `status` and `internalRemarks` (`FirUpdateInput`). `useSaveFir` takes `input` for create or `update` for edit |
| `src/features/fir/fir-page.tsx` | Status tabs with counts (opens on Open), a priority filter, and Priority and Status columns. The details column is as wide as its longest value, up to 400px |
| `src/features/fir/fir-form-modal.tsx` | Priority, Status and Internal Remarks fields. When editing, everything except Status and Internal Remarks is disabled. An error clears as soon as the field is changed |
| `src/features/fir/fir-view-dialog.tsx` | Shows priority, status and internal remarks |
| `src/features/reports/fir/api.ts`, `src/features/reports/fir/fir-report.tsx` | Status and priority filters, columns, PDF and Excel cells, and filter text in the PDF |
| `src/components/ui/status-label.tsx` | New tones and icons for FIR priority and status (`toneForPriority`, `toneForStatus`) |
| `src/features/accounts/` (new, 5 files) | Accounts page, add and edit dialog, view dialog, queries and types. Sales or Purchase type with optional bank details |
| `src/features/items/` (new, 5 files) | Items page, add and edit dialog, view dialog, queries and types. Product, glass type, colour, thickness, HSN and the Sales and Purchase accounts |
| `src/routes/admin.items.tsx`, `src/routes/admin.accounts.tsx` (new) | Routes for the two pages |
| `src/components/layout/nav-items.ts` | Product Details moves into a new "Product Catalog" group with Items. Accounts is added under Billing & Finance |
| `src/features/auth/accessible-routes.ts` | Adds the two routes with modules `master_item` and `master_account` |
| `src/lib/lookup.ts` | New `items` and `accounts` lookup types and names |
| `src/features/inventory/components/purchase-dialog.tsx` | Product column is now Item. Picking an item fills product, HSN, glass type, colour and thickness, and locks those three. "+ Add Item" opens the Item form (needs the Add right on `master_item`). The old "+ Add Product" and "+ Add Type" are removed. Older lines with no item keep their product |
| `src/features/inventory/components/purchase-view-dialog.tsx`, `src/features/inventory/purchase-types.ts` | Shows the item name. Types carry `itemCode`, `itemName`, `hsnDescription`. `productId` is optional |
| `src/features/company/company-page.tsx`, `src/features/company/schema.ts`, `src/features/company/types.ts` | Bank section removed from the edit form and schema. New read-only `BankAccountsCard` lists Sales and Purchase accounts with bank details, with a hint to add them on the Accounts page |
| `src/features/reports/price-list/api.ts` | Response type changed to `items`, `totalCount`, `glassTypes`. Fetch takes `glassTypeId` and paging |
| `src/features/reports/price-list/price-list-report.tsx` | One paged table instead of a table per glass type. New product type filter. Downloads fetch every row from the server |
| `src/features/products/product-dialog.tsx` | Four tabs and Next and Previous removed. One scrolling page with sections, and validation scrolls to the first section with an error. Default routing work center is now empty instead of the first one |
| `src/features/products/product-view-modal.tsx` | Simplified to match the one page layout |

## Why

The FIR register needed progress and urgency. Purchases should only use agreed item combinations tied to Sales and Purchase accounts, and bank details should live with those accounts. The Price List and product dialog were changed to match how the other pages already work.

## How

**FIR edit**: after a FIR is raised only status and internal remarks can change, so the form disables the rest in edit mode and sends just those two.

**Status tabs**: the counts come from the server and ignore the status filter, so each tab shows what it would hold. The page opens on Open.

**Item on a purchase row**: the dropdown lists active raw material items, labelled Product, Glass Type, Colour, Thickness. A deactivated item that a row already uses stays selectable so older purchases still show their pick. When an item has all of glass type, colour and thickness, those three are not editable on the row.

**Company bank details**: shown from `company.bankAccounts`. Editing is only on the Accounts page.

**Price List paging**: the table asks for one page. Excel and PDF call the same endpoint with no paging to get every row.

## Decisions made

**Product dialog is one page.** The tabs were removed so Rates, Charges and Routing are always visible. The "open on the Rates tab" shortcut now scrolls to that section.

**"+ Add Product" and "+ Add Type" were removed from the purchase dialog.** New glass combinations are added through the Item form. "+ Add Thickness" and the location add stay.

## Notes

- The Accounts and Items menus depend on `master_account` and `master_item` existing as modules with rights for each role. I could not confirm they are seeded.
- Because the product dialog default work center is now empty, a new product's routing row needs a work center picked before saving.

## Next steps

- Click through the Items, Accounts, FIR and purchase screens once migration 112 is applied.

## Test plan

**Automated tests (no new tests):**
- [x] TypeScript build (`npx tsc --noEmit -p .`): no errors.
- [ ] No component tests were written for these screens.

**Manual checks (local):**
- [ ] Create a FIR, change its status in edit mode, and see the tab counts move - not done, the browser was not used.
- [ ] Add an Account with bank details and see it on the Company Profile - not done.
- [ ] Add an Item and pick it on a new purchase row, then edit an older purchase without items - not done.
- [ ] Filter the Price List by product type, change page, and download Excel and PDF - not done.
- [ ] Save a product from the one page dialog, including a validation error in Rates - not done.
