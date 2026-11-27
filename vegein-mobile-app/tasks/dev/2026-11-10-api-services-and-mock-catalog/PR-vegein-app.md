# 2026-11-10 - api-services-and-mock-catalog (mobile app)

## Summary

The app only had API code for login, products and categories, and the Home, Categories and Product Details screens called those endpoints directly. The cart, checkout, orders, account and search endpoints of the Medusa backend had no code in the app at all.

This PR adds the service and React Query hook layer for all of those remaining endpoints (cart, checkout, delivery slots, payments, orders, addresses, profile, collections, regions, currencies and search). None of it is called from a screen yet. It also moves the Home, Categories and Product Details screens back to local mock data for now, while keeping the product and category services and hooks in place, so a screen can switch back to the API by changing one import.

It also fixes a dev crash where the toast component threw "GestureDetector must be used as a descendant of GestureHandlerRootView".

Types and lint were checked and pass. Nothing was run against the backend or on a device or emulator, see the Test plan.

**Wiring the cart, checkout and orders screens to the new hooks is not included on purpose** - see "Next steps" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/features/cart/types.ts`, `services/cartService.ts`, `services/cartStorage.ts`, `hooks/useCart.ts` (new) | Cart endpoints: create, get, update, add / update / remove line items, apply / remove promo codes, add shipping method, refresh taxes, attach customer, complete. The cart id is kept in `expo-secure-store` (memory only on web). Hooks write the returned cart straight into the query cache. |
| `src/features/checkout/**` (new) | Delivery slots (list and attach to cart), shipping options, payment collection, payment session and the mock payment endpoint, with hooks. |
| `src/features/orders/**` (new) | `GET /store/orders` (paged) and `GET /store/orders/:id`, with hooks. |
| `src/features/account/**` (new) | Update profile, addresses (list, create, update, delete) and set a new password, with hooks. |
| `src/features/catalog/**` (new) | Collections (list and by id), regions and currencies, with hooks. |
| `src/features/search/**` (new) | `POST /store/search` and a `useSearchProducts` hook that waits for at least 2 characters. |
| `src/features/categories/categoryService.ts` | Added `fetchCategory` (with direct children), `fetchSubCategories` and the hooks `useCategory` and `useSubCategories`. The existing list call is unchanged. |
| `src/features/products/services/productService.ts`, `hooks/useProducts.ts` | Added `fetchProductByHandle`, `fetchProductsByIds` and their hooks. The product mapper is now exported as `toProductFromRaw` so search can reuse it. |
| `src/data/mock-catalog.ts` (new) | Six mock products and the nine mock categories, plus `useMockProducts`, `useMockProduct` and `useMockCategories`. They return the same shape as the real hooks. |
| `src/app/(tabs)/index.tsx`, `categories.tsx`, `src/app/product-details.tsx` | Now use the mock hooks instead of the API hooks. Only the imports and the hook calls changed. |
| `src/features/products/types.ts`, `format.ts`, `components/ProductCard.tsx` | A product `thumbnail` can now be a bundled image (number) as well as a URL. A small `imageSource()` helper picks the right form. Product Details and the card use it. |
| `src/app/_layout.tsx` | The root `View` is now a `GestureHandlerRootView`. This fixes the toast crash in dev. |
| `src/app/(tabs)/account.tsx` | The "Verified B2B" tag was removed from the header. This is a small edit made outside this task, included because it is in the working tree. |

## Why

Cart, checkout and orders need the backend contract coded before the screens can be switched over. Doing the service layer first keeps that work separate from the UI, so each screen can be moved over on its own later. The screens go back to mock data meanwhile so the app does not depend on the demo backend being up while the new layer is built.

The easy part was the standard Medusa endpoints, which follow known response shapes. The real work was the four custom endpoints (delivery slots, attach slot, mock payment, search), where the response shape is not documented in the repo.

## How

**One client**: Every call goes through the existing `apiRequest`, so the publishable key, bearer token, timeout and `ApiError` handling are the same as for login and products.

**Services and hooks are split**: Each feature has a `services` file (plain async functions that map the API response to a clean type) and a `hooks` file (React Query wrappers). Screens only need to import the hook.

**Cart**: `ensureCart` returns the stored cart or creates one. Every cart mutation calls it first, so a hook like `useAddToCart` works even when no cart exists yet. If the stored cart is gone on the server (404), the saved id is cleared and a new cart is made. Completing the cart clears the saved id.

**Prices**: Amounts are read as major units (rupees), the same way the existing product mapper does. Money fields default to 0 when missing.

**Mock hooks**: `useMockProducts` filters locally by search text and category and returns a single page, with the same fields the home screen reads (`products`, `total`, `isPending`, `hasNextPage`, `fetchNextPage` and so on).

## Decisions made

**Kept the product and category hooks, removed only the screen calls.** The screens now use the mock hooks, but `useProducts`, `useProduct` and `useCategories` still exist and still work. This was asked for so the API calls can be switched back on screen by screen.

**Mock data lives in its own file.** It is in `src/data/mock-catalog.ts`, not in `mock-data.ts`, so it is easy to delete once the screens use the API again. The mock categories reuse the existing `CATEGORIES` list.

**Guessed the custom endpoint shapes.** The delivery slots response, the attach slot body, the mock payment body and the search body and response are assumptions, each marked with an `ASSUMPTION` comment in the code. The standard Medusa endpoints follow the Medusa v2 docs.

**Payment provider id is a parameter.** `createPaymentSession` takes the provider id from the caller (for example `pp_razorpay_razorpay`) instead of hard coding it.

## Notes

- The Medusa transfer order endpoint (`POST /store/orders/:id/transfer/request`) is not added because the app does not need it.
- The cart `taxes` and `complete` calls are not in the endpoint list that was shared, but they are standard Medusa cart endpoints and are needed for checkout.
- `splash.mp4` shows as deleted in the working tree. That is not part of this change.

## Next steps

- Confirm the real shapes of the four custom endpoints with the backend and fix the mappers where the guesses are wrong.
- Confirm the exact Razorpay provider id in Medusa Admin.
- Switch the Cart, Checkout, Orders and Account screens over to the new hooks, one screen at a time. The cart screen still runs on the mock `AppContext` store.
- Switch Home, Categories and Product Details back to `useProducts`, `useProduct` and `useCategories` when the mock data is no longer wanted, and delete `mock-catalog.ts`.

## Test plan

**Automated tests (no test suite in this repo, type check and lint pass):**
- [x] `npx tsc --noEmit`: no errors.
- [x] `npx expo lint`: 0 errors, 1 warning (`require()` import in `splash-video-overlay.tsx`, already there before this change).

**Manual checks (local):**
- [ ] Home, Categories and Product Details showing the mock products and categories - not run on a device or emulator, only type checked.
- [ ] Toast no longer crashing with the `GestureHandlerRootView` error - not run, the fix is the root wrapper only.
- [ ] Any of the new cart, checkout, orders, account, catalog or search calls against the backend - not run, no screen calls them yet. The four custom endpoints in particular are unverified guesses.
