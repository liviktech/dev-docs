# 2026-11-11 - pwa-master-data (frontend)

## Summary

The installed phone app (PWA) had no Master Data. The menu left it out, and opening a Master Data page on a phone showed the desktop page squeezed into the mobile shell. This PR adds a mobile screen for every Master Data page under `/m/master/*`, adds the Master Data section to the mobile menus, and makes the Add and Edit forms open as bottom sheets, like the Layers form does.

All 21 Master Data pages are covered. The simple code and name lists share one screen. Party, Products, Items, Sales Reps, Users, Roles, Glass Thickness, Sequences, Company Profile and Notifications get their own screens or adapters. Same permissions, same fields and same API calls as desktop. Desktop pages only got a redirect line in `beforeLoad`, nothing else changed there.

`tsc -b` and `vite build` pass. **I did not test this on a real phone or in a browser**, so the layout fixes for Party and Product Details are checked by reading the code only. See "Test plan".

**Backend is not touched on purpose.** Every endpoint already existed, so there is no `PR-backend.md` for this task. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/pwa/master/master-types.ts` (new) | The `MasterAdapter` type: one config object per page (fields, list row, validation, payload, API calls). |
| `src/features/pwa/master/master-screen.tsx` (new) | The shared mobile list screen: search, status filter, sibling tabs (scrollable), cards, add and edit sheets, activate and deactivate with confirmation. |
| `src/features/pwa/master/master-form.tsx` (new) | The shared form sheet used by the simple pages. |
| `src/features/pwa/master/adapters.ts` (new) | Adapters for the simple pages: Charges, Work Centers, Payment Terms, Invoice Terms, Tax Profiles, Order Charges, Shifts. |
| `src/features/pwa/master/heavy-adapters.tsx` (new) | Adapters for Party, Products, Items, Glass Types, Colors, Finishes, Glass Thickness, Sales Reps, Users and Roles. The big forms reuse the desktop dialogs through small `Editor` wrappers. |
| `src/features/pwa/master/tax-slab-presets.tsx` (new) | Helper for the tax profile slabs. |
| `src/features/pwa/master/mobile-sequences.tsx` (new) | Native mobile Sequences screen. |
| `src/features/pwa/master/mobile-company.tsx` (new) | Native mobile Company Profile: contact, statutory, address, bank accounts, UPI QR, profile form and bank account form. |
| `src/features/pwa/master/mobile-notifications.tsx` (new) | Native mobile Notifications: Events tab and Notification log tab (status chips, event filter, retry, detail). |
| `src/features/pwa/master/embedded-screens.tsx` (new) | Maps `sequences`, `company` and `notifications` to their own screens. |
| `src/routes/m.master.$slug.tsx` (new) | The one mobile route for all Master Data. Picks the adapter or own screen from the slug, shows "Page not found" otherwise, and turns on the bottom sheet style for dialogs. |
| `src/components/ui/dialog-sheet-context.ts` (new) | A small React context (`DialogSheetContext`, default false). When true, any dialog renders as a bottom sheet. |
| `src/components/ui/dialog.tsx` | `DialogContent` reads the context and adds the bottom sheet classes. Nothing changes when the context is false. |
| `src/components/ui/form-dialog.tsx` | Form height is capped at `90dvh` inside a sheet. |
| `src/features/pwa/mobile-modules.ts` | New `MASTER_MOBILE_PATHS` (desktop path to mobile path), `mobilePathFor` looks it up, and new `guardWithMobileRedirect` (permission guard, then redirect to the mobile screen). |
| `src/features/pwa/mobile-shell.tsx` | The mobile menu now lists Master Data by category, only the items the user can view. |
| `src/components/layout/mobile-menubar.tsx` | Same Master Data section in the small screen menu bar. |
| `src/features/pwa/components/list.tsx` | `SegmentedTabs` got an optional `scrollable` prop (sideways scroll, no wrapping). Default behaviour is unchanged. |
| `src/features/parties/party-dialog.tsx` | Card header can wrap, so "Add Project" no longer runs off the edge. |
| `src/features/bank-accounts/bank-accounts-section.tsx` | Same wrap fix for "Add Bank Account". |
| `src/features/products/product-dialog.tsx` | Sticky and scrollable step tabs, grid columns can shrink (`min-w-0`), routing header wraps, work center select goes full width on small screens. |
| 21 files in `src/routes/` (`admin.charges`, `admin.company`, `admin.destinations`, `admin.invoice-terms`, `admin.items`, `admin.layers`, `admin.logos`, `admin.notifications`, `admin.order-charges`, `admin.payment-terms`, `admin.ports`, `admin.roles`, `admin.sales-reps`, `admin.sequences`, `admin.shifts`, `admin.suppliers`, `admin.tax-profiles`, `admin.users`, `admin.work-centers`, `party`, `products`) | Only `beforeLoad` changed to `guardWithMobileRedirect(<same guard>, '/m/master/<slug>')`. No real change to the desktop pages. |
| `PWA.md` | Documents the Master Data mobile screens and how they are wired. |

## Why

PWA.md asks for every module to have a mobile screen. Master Data was the last big group without one. The real work was the heavier pages (Party, Products, Company, Notifications) where the form is large, so those reuse the desktop dialogs and components instead of being rebuilt.

The follow up request was to make the Add and Edit forms open from the bottom like the Layers form, and to fix the Party and Product Details layout shown in the phone screenshots.

## How

**One screen, many adapters**: each simple page is a config object (`MasterAdapter`). `MobileMasterScreen` renders the list and the form from it. Adding a page later means adding an adapter, not a new screen.

**Redirect only on the installed app**: `guardWithMobileRedirect` runs the old permission guard first, then redirects to the mobile path only when the user is on the installed phone app (`?pwa=true` simulates it). Desktop users see the same pages as before.

**Bottom sheets through a context**: `m.master.$slug.tsx` wraps its content in `DialogSheetContext.Provider`. `DialogContent` reads it. Because it is a context, it also works for dialogs inside dialogs and for portaled dialogs (for example Bank Account inside Party). Pages outside `/m/master/*` never set it, so they keep the centered dialog.

**Same rules as desktop**: view and edit rights use the same module codes. Validation uses the same schemas (for example `companyFormSchema` for Company Profile). The payload sent to the API is built the same way as desktop. The SUPER_ADMIN role cannot be deactivated.

**Layout fixes**: Party and Product dialogs have `flex-wrap`, `min-w-0` and a scrollable sticky step bar so nothing is wider than the sheet. These classes also apply on desktop but only let things shrink or wrap when there is no room, so the desktop look is the same.

## Decisions made

**Reuse the desktop dialogs for big forms.** Party, Products, Roles, Users, notification event edit and message detail open the existing dialogs as bottom sheets instead of new mobile forms. Rebuilding them would copy a lot of logic and could drift from desktop. The cost is that the layout is only as mobile friendly as the desktop dialog, which is why the Party and Product alignment fixes were needed.

**Native screens for Sequences, Company and Notifications.** These were first shown as the desktop page inside the mobile shell. The request was to give them a real mobile layout, so they now have their own screens.

**Context instead of a prop for the sheet style.** A prop would have to be passed through every dialog, including nested ones. The context needs one provider and leaves desktop and non-master pages alone, as requested.

**The Retry button in the notification log sits next to the card, not inside it.** The card is a `<button>`, and a button inside a button is invalid HTML.

## Bugs found while testing (already there before, not fixed here)

**1. Lint error in `src/features/pwa/final-invoice-payment-sheet.tsx`.** `react-hooks/set-state-in-effect`. It was there before this change and is not part of this work.

**2. `react-refresh/only-export-components` in route files.** The same warning appears on `m.master.$slug.tsx` (2 errors) and on other route files that export both a route and a component. It follows the existing route pattern, so I left it.

## Notes

- Only the frontend changed. The backend repo has no changes for this task.
- The Rates and Charges tabs of the Product form are wide tables. I did not check how they fit in the sheet.
- Company Profile reuses the desktop bank account fields (`BankAccountFields`) inside the mobile bank account sheet.

## Next steps

- Check the Party and Product forms, and the Rates and Charges tabs, on a real phone and fix anything that still overflows.
- Decide whether to clean up the `react-refresh` lint errors in the route files across the app.

## Test plan

**Automated tests (no test files added or changed):**
- [x] `npx tsc -b`: passes.
- [x] `npx vite build`: passes.
- [x] `eslint` on the touched files: only the 2 `react-refresh` errors in `m.master.$slug.tsx` (same pattern as other routes). One older error in `final-invoice-payment-sheet.tsx`, not from this PR.

**Manual checks (local):**
- [ ] Open each Master Data page on a phone (or `?pwa=true` at phone width) and check list, search, Add, Edit and activate/deactivate - not done, I did not run the app in a browser.
- [ ] Party Add form: "Add Bank Account" and "Add Project" fit on screen - not checked live, fixed by wrapping the header.
- [ ] Product Details tab bar shows "Thickness" and scrolls sideways - not checked live.
- [ ] Add and Edit Product form: tabs, inputs and Routing Sequence fit inside the sheet - not checked live.
- [ ] Desktop pages still open normally on a desktop browser (no redirect) - not checked live, only the `beforeLoad` line changed.
