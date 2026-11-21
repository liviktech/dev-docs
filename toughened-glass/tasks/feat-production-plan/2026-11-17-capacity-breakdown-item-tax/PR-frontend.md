# 2026-11-17 - capacity-breakdown-item-tax (frontend)

## Summary
This branch (`feat/production-plan`, one commit on top of `dev`) updates the production plan editor, the item form and the PI approval modal. The capacity meter on the plan editor only showed one date, the first one in the plan. The item form only asked for an HSN code and Tax Profile on sold finished goods, and the user had to fill in a lot by hand. The factory approval modal had no quick way to enter a surcharge. This PR adds a date picker to the capacity meter, the new `breakdown` data type, auto-fill on the item form, required HSN and Tax Profile, and surcharge presets. Only the type check was run (`npx tsc -p tsconfig.app.json --noEmit`, no errors). I did not open the app in a browser, so none of the screens were checked live.

**The capacity breakdown is not shown on screen on purpose** - see "Decisions made" below. This PR only adds the type and the field for the API response. This was a choice, not a miss.

## What changed
| File | Change |
|---|---|
| `src/features/production-plan/production-plan-editor.tsx` | `CapacityBanner` now takes all plan lines and shows a row of date buttons (with a line count each) when the plan has more than one date. The meter shows the selected date. Before, it always showed the first date. |
| `src/features/production-plan/daily-capacity-meter.tsx` | Label weight changed from bold to semibold and the numbers use tabular figures so they do not jump. No change in logic. |
| `src/features/production-plan/api.ts` | Adds the `CapacityBreakdownItem` type and an optional `breakdown` list on `DailyCapacityUsage`. |
| `src/features/items/item-dialog.tsx` | HSN code and Tax Profile are now required for every item and always show the red star. Choosing a product that looks like finished glass (toughened, tempered, laminated, DGU, double, insulated) on a new item sets the type to Finished Good, turns Sales on and Purchase off. A blank Tax Profile is set to the 18 percent profile when there is one. Changing the item type does the same for Sales and the Tax Profile. |
| `src/features/items/queries.ts` | Create, update and deactivate now refresh the lookup data and the products list too, and add-to-product refreshes the lookup. |
| `src/features/pi-status/components/approve-party-modal.tsx` | The surcharge label now says Same-Day or Express to match the delivery type. It shows the new PI total when a surcharge is entered, and has preset buttons (500, 1000, 2000, 2500). |

## Why
Planners need the meter to follow the date they are working on, and they need to know what makes up the figures. The breakdown type is the first part of that. The item change matches the backend, which now needs an HSN code and Tax Profile on every item, so the form has to ask for them too. The auto-fill saves time because almost all finished glass items are sold and taxed at 18 percent. The surcharge presets and total are a small help for the factory approver. The real work was the capacity date picker, since the meter has to keep working when the dates in the plan change.

## How
**Date picker**: the unique planned dates are taken from the lines and sorted. The selected date is kept in state. If it is no longer in the plan (and is not today), the first plan date is used. If there are no dated lines, today is used. The meter is given `onDateChange` and `interactiveDate`, which the meter already supported.

**Item auto-fill**: runs on product blur and on product select, only when adding a new item. It does not run when editing, so saved items are never changed behind the user's back. The finished-glass test is a word match on the product name.

**Query refresh**: the lookup key and the `products` key are refreshed after any item change, so other screens that read items show the new data without a reload.

**Kept the same on purpose**: the meter limits (25, 150, 300, 250, 400) are still written in the component. The surcharge value is still sent as typed, the presets only fill the input.

## Decisions made
**Breakdown not drawn yet.** I added the type and field only. The screen to show it was not part of this change. The backend already returns the list.

**Auto-fill uses a name match.** I did not add a product flag for finished goods. The word list is simple and may miss a product. The user can still change the type by hand.

**18 percent is the default tax rate.** It is looked up by `taxPercentage === 18` in the tax profile list, not by a code. If no 18 percent profile exists, the field stays blank and the required message shows.

## Bugs found while testing (already there before, not fixed here)
**1. Changing the item type to Finished Good always turns Sales on.** This also happens when the user had turned Sales off on purpose. It comes from the new auto-fill rule in this PR, so it is a risk of this change and not an old bug. It is listed here so it is not lost. The user has to turn Sales off again by hand.

## Notes
- The type check passed with no errors. The app was not opened, so the date buttons, the auto-fill and the preset buttons were not seen working.
- The 18 percent default and the finished-glass word list should be confirmed with the business.
- The matching backend PR is `PR-backend.md` in this folder. The backend change should go out first, or the old rule still lets items save without an HSN code.

## Next steps
- Show the capacity breakdown (list of plans and same-day PIs) under the meter.
- Add a Playwright or component test for the item form auto-fill and the required fields.
- Move the capacity limits into one shared place so the meter and the backend do not drift.

## Test plan
**Automated tests (none added or run):**
- [ ] No test file was added or changed for this PR.
- [x] `npx tsc -p tsconfig.app.json --noEmit`: no errors.

**Manual checks (local):**
- [ ] Plan editor with lines on two or more dates shows the date buttons and the meter follows the chosen date - not run, app not opened.
- [ ] New item with a product like "Toughened Glass" sets Finished Good, Sales on, Purchase off, and the 18 percent profile - not run.
- [ ] Saving an item with a blank HSN code or Tax Profile shows the required message - not run.
- [ ] Factory approval modal shows the surcharge presets and the new total - not run.
