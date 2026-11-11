# 2026-11-06 - admin-self-approve-item-add (frontend)

## Summary

Admins could not approve their own production or dispatch plans, and the Add Product dialog sent people to the Thickness and Type master pages to add values that only exist when an item has them. This PR lets ADMIN and SUPER_ADMIN approve and reject their own plans, adds "+ Add Product", "+ Add Thickness" and "+ Add Type" in the Add Product dialog (all open the Add Item form), lets the Add Item form pick an existing product name, and fixes the table fonts on the Items page. The frontend type check (`npx tsc -b --noEmit`) passes. The new flows were not clicked through in the browser.

## What changed

| File | Change |
|---|---|
| `src/features/production-plan/plan-approve-requests-dialog.tsx` | An admin's own plan shows Cancel Request and Approve in one row, with no Reject. Other users keep Cancel Request only on their own plan. |
| `src/features/dispatch-plan/dispatch-plan-approve-requests-dialog.tsx` | Same change as the production plan dialog. |
| `src/features/production-plan/production-plan-panel.tsx` | Approve action is allowed on an own plan for admins. The disabled tooltip only shows for non admins. |
| `src/features/dispatch-plan/dispatch-plan-panel.tsx` | Same change for the dispatch plan table. |
| `src/features/production-plan/production-plan-view-modal.tsx` | `canDecide` allows admins on their own plan. |
| `src/features/dispatch-plan/dispatch-plan-view-modal.tsx` | Same change as the production plan view modal. |
| `src/features/products/product-dialog.tsx` | Product Name dropdown gets "+ Add Product". The thickness and type dropdowns in the rate rows now open the Add Item form instead of the Thickness and Type master dialogs. The saved item's thickness and type are picked in that row once the refreshed lists contain them. The add options only show for users with Add rights on Items. |
| `src/features/items/item-dialog.tsx` | New `defaultProductName` prop. The Product field is now the shared `OptionSelect` with the existing product names (from products and active items), with "+ Add New Product" switching to a text box for a brand new name. |
| `src/features/items/items-page.tsx` | Colours, Thickness, Type, HSN Code and HSN Description cells now use `text-xs font-semibold text-slate-800` like the other master tables. The `font-mono` on HSN Code is removed. |

## Why

Admins are the final approvers, and the backend rule that blocked self approval had no exception for them (see the backend PR). A thickness or type is only offered when an item has it, so sending users to the Thickness and Type master pages did not help. The Add Item form is the one place that adds them. The Product Name on the Add Item form was free text, which made it easy to create near duplicate product names.

## How

**Admin self approval**: The UI reads `isAdminRole` from `useAuth()`. `canDecide` becomes `pending && canReview && (!mine || isAdminRole)`. On an own plan the card shows Cancel Request and Approve in one row and hides Reject. On someone else's plan nothing changes.

**Add Item from the Product dialog**: One `quickAdd` state holds the row index. `null` means the product name itself is being added. For a rate row the item's thickness and type are stored in `pendingPick`, and an effect selects them once the refreshed lookup lists contain them. For a new product name the dialog sets the name and item code from the saved item and does not touch the rate rows.

**Product field on Add Item**: Options are the product names from `useLookupList('products')` plus the product names of active items, de-duplicated ignoring case. The current value is always added to the options so an edited item still shows its product. "+ Add New Product" clears the value and shows a text box, with a "Choose an existing product" link to go back. If a typed name matches an existing one ignoring case, it takes the existing spelling. If there are no names at all, the field is just the text box as before.

**Items table fonts**: Data cells now use the same `text-xs font-semibold text-slate-800` as the other master tables. Code, Status and Actions already used the shared components and were not changed.

## Decisions made

**Reused `OptionSelect` instead of a browser datalist.** I first tried a native datalist for the product name. It looked different from every other dropdown, so I replaced it with the shared component. `OptionSelect` has no free text mode, so new names use the "+ Add New Product" switch to a text box.

**Admin check is also kept in the backend.** The frontend only shows the buttons. The backend still decides, so a non admin cannot get around it by calling the API.

## Bugs found while testing (already there before, not fixed here)

**1. Rate rows are not filled when a new product is added.** When "+ Add Product" saves a new item, only the product name and item code are set. Thickness and type in the rate rows stay as the user left them. This is how the rate rows worked before, and it is left as is.

## Notes

- `npx tsc -b --noEmit` exited 0 after the last change. An earlier editor warning about `typingProduct` was stale.
- The dispatch plan approve dialog hooks fix (a `useEffect` moved above an early return) was made during this work and is not a separate change.

## Test plan

**Automated tests (no tests added):**
- [x] `npx tsc -b --noEmit`: passes with exit code 0.
- [ ] Lint: not run in this session.

**Manual checks (local):**
- [ ] Admin approves own production plan and dispatch plan from the Approve Requests dialogs and tables - not run in the browser. The backend change is needed first.
- [ ] Add Product dialog, "+ Add Product", "+ Add Thickness", "+ Add Type" open the Add Item form and select the saved value - not run in the browser.
- [ ] Add Item form product dropdown, "+ Add New Product" and "Choose an existing product" - not run in the browser.
- [ ] Items page table fonts - checked against the screenshot only, not re-checked after the final change.
