# 2026-11-06 - auth-and-live-products (mobile app)

## Summary

The mobile app had no real login. Every screen ran on mock data, including the product list, the cart and the categories. This PR adds a real customer login flow (login, register, forgot password) against the Medusa 2.22.0 backend, moves navigation to a protected Stack, and replaces the mock products and categories with live data from `https://demo.api-vegetables.liviktech.com`.

The home screen now lists real products with infinite scroll and a loading text at the bottom. Product details and the Categories tab also use live data. Types and lint were checked and pass. Login, register and the product list were not run end to end on a device or emulator, see the Test plan.

The PR also adds an animated video splash and a redesigned login screen. On launch the app plays a short logo video that zooms into the letter "e" and ends on white. Then the login page, or the home tabs for a signed in customer, fades in with no loader in between. The login screen now follows the supplied design with a 50% header and 50% form.

**Orders, purchase orders, stock and reports are not included on purpose** - see "Notes" below. They still use mock data. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/services/api/apiClient.ts` (new) | The single HTTP client. Adds the publishable key and bearer token, times out after 15 seconds and turns every failure into an `ApiError` with a clear `kind`. |
| `src/services/api/queryClient.ts` (new) | Shared TanStack Query client. It does not retry unauthorized, validation, conflict or config errors. |
| `src/features/auth/**` (new) | Auth types, validation, `authService`, `tokenStorage`, `AuthContext`, `useAuth`, the mutation hooks and the Login, Register and Forgot Password screens. |
| `src/components/splash-video-overlay.tsx` (new) | Full screen splash video player built on `expo-video`. It plays once, muted. When the video ends the next screen mounts underneath and the overlay fades out over 1.4 s. A 6 second safety timeout and an error handler stop it blocking the app. |
| `src/components/animated-icon.tsx`, `animated-icon.web.tsx` | Now only re-export the video overlay. The old animated basket and loading bar splash was removed. |
| `assets/videos/splash-zoom.mp4` (new), `assets/videos/splash.mp4` (new) | `splash-zoom.mp4` is the video the app plays: portrait 1080x1920, about 3.0 s, zooming into the 4th letter "e" and ending on white. `splash.mp4` is the original source clip, kept as a backup. |
| `app.json`, `assets/images/splash-blank.png` (new), `android/app/src/main/res/drawable-*/splashscreen_logo.png` | The native splash image is now fully transparent on a white background, so the "Vegein" icon no longer shows before the video. `expo-video` was added to the plugins. |
| `src/features/auth/screens/LoginScreen.tsx` | Redesigned to the supplied layout: a green header artwork in the top 50% of the screen height with a curved edge, then the title, two pill fields, Forgot Password, Log In button and a register pill. Sizes scale with the screen width. The sign in logic, validation and banners are unchanged. |
| `src/components/auth/LoginField.tsx` (new), `assets/images/login-header.png` (new) | The pill shaped input used by the new login screen, and the header artwork (logo, vegetables and curve). |
| `src/components/auth/**` (new) | `AuthInput`, `PasswordInput`, `PrimaryButton`, `Banner` and `AuthScreenLayout`. |
| `src/app/login.tsx`, `register.tsx`, `forgot-password.tsx` (new) | Routes that re-export the auth screens. |
| `src/app/_layout.tsx` | Root is now a Stack with providers (Query, Auth, App). Guest screens and customer screens sit in separate protected groups. The splash overlay stays mounted. The navigator is held back, on a plain white screen with no spinner, until the video has ended and the session has been restored. The login screen uses a `fade` animation. |
| `src/app/(tabs)/_layout.tsx` (new) | Tab bar whose height and padding follow the device bottom inset. Larger icons and labels. |
| `src/app/(tabs)/index.tsx`, `categories.tsx`, `orders.tsx`, `account.tsx` | Moved into the `(tabs)` group. Home now has the paged product grid. Categories use live data. Account has a real Sign Out and shows the customer profile. |
| `src/features/products/**` (new) | Product types, `productService`, `useProducts` (infinite query), `useProduct`, `ProductCard`, `ListLoader` and a price formatter. |
| `src/features/categories/categoryService.ts` (new) | Fetches `/store/product-categories` and caches it for 30 minutes. |
| `src/app/product-details.tsx` | Shows the tapped product from the backend with its image, price, pack sizes and description. Loading and error states added. |
| `src/store/app-context.tsx` | `addToCart` takes a small `CartProductInput` instead of the old mock `ProductItem`. The cart now starts empty. |
| `src/data/mock-data.ts` | Removed `PRODUCTS`, `INITIAL_CART`, `ProductItem` and `PriceTier`. |
| `src/constants/theme.ts` | Added `AuthColors`. |
| `.env.example` (new) | Lists `EXPO_PUBLIC_API_URL` and `EXPO_PUBLIC_MEDUSA_PUBLISHABLE_KEY`. |
| `.githooks/pre-push` (new), `package.json` | Pre-push hook that runs `yarn lint`. A `postinstall` script sets `core.hooksPath`. Added `expo-secure-store`, `@tanstack/react-query` and `expo-video`, and aligned Expo packages with `expo install --fix`. |
| `tsconfig.json`, `.vscode/settings.json` | Set `jsx` to `react-jsx` and pointed the editor at the workspace TypeScript. This fixes the `--jsx` editor error. |

## Why

The app needed real accounts and real data before any ordering work can start. Login, session handling and the product list are the base for the cart, checkout and orders that come next. Most of the work was the session handling (restore on launch, expiry, logout) and keeping every call going through one client.

## How

**Login and session**: Login uses `POST /auth/customer/emailpass`. Register calls `/auth/customer/emailpass/register`, then `POST /store/customers`, then logs in again. The token is kept in `expo-secure-store` (memory only on web). On launch the app reads the token and checks it with `GET /store/customers/me`. A 401 on a request that carried a token signs the user out and clears the query cache.

**Secure store loading**: `expo-secure-store` is loaded lazily inside a try/catch. An old dev build without the native module no longer crashes the app. Sessions just do not persist until the dev client is rebuilt.

**Navigation**: Guests only reach login, register and forgot password. Signed in customers reach the tabs and the other screens. The router moves the user automatically when the session changes.

**Product list**: One request per page of 10 products, using `limit` and `offset`. Each request asks only for the fields the screen shows. Pages are cached for 5 minutes and duplicates are dropped. Search is debounced by 400 ms and runs on the server. The next page loads when the list is near the bottom, and a spinner with "Loading more products..." shows while it loads. There are also text states for first load, a failed page (tap to retry), the end of the list and no results.

**Categories**: Top level categories come from `/store/product-categories`. Tapping one filters products with `category_id[]`. Tapping the same tile again clears the filter. The Categories tab sends the chosen category to the home screen through the route.

**Product details**: The screen opens instantly from the list cache, then fills in the details. A pack size picker shows only when a product has more than one variant. Add to Cart and Buy Now are disabled when the variant has no price.

**Splash video**: The overlay loads `expo-video` inside a try/catch. If the installed dev build does not contain the native module, the splash is skipped instead of crashing the app. The native splash is hidden as soon as the video is ready to play. The video ends on plain white. The overlay then waits for the session check, so no spinner appears, and dissolves to the login page or the tabs. The zoom part of the video was rendered from the sharp 1080p logo frame, which keeps the letters crisp. The framing is portrait so the video fills the whole phone screen.

**Login screen**: Every size comes from the 941 px wide design (`s = screen width / 941`). The top 50% of the screen is the header image. The form sits in the bottom half and starts slightly under the curve, using a negative top margin. The header artwork was built from the supplied reference. Its old curve was cut out and a smooth S shaped curve was drawn in its place, following the supplied sketch. The background is a visible light to dark green gradient. The pepper in the artwork is stretched slightly so it meets the curve.

## Decisions made

**Product images use the backend URL, with a fallback icon.** The seeded products point at `loremflickr.com`, which returned HTTP 500 when checked. A failed image now shows a leaf icon instead of an empty box. Real images need to be uploaded in Medusa Admin.

**Category tiles use an icon, not a picture.** Medusa categories have no image field, so the old mock category pictures were dropped.

**The cart now starts empty.** The old starting cart held mock products, so it was removed along with them.

**No phone or OTP login and no Terms or Privacy links.** The backend has no phone or OTP login. No Terms or Privacy URLs were provided.
**The login header is a baked image, not drawn in code.** The logo, vegetables and curve are one picture, because the vegetables could not be cut out cleanly. The picture is made for tall phones (height to width about 2.2). On a much shorter screen the sides of the header are cropped slightly.

**The first field says "Email or Mobile Number" as in the design, but sign in still accepts only an email address.** The backend has no phone login (see above). The placeholder and the validation do not match yet.

**The native splash cannot be removed completely.** Android always shows a blank window until the app's JavaScript is ready. It is now plain white with no icon, and the video's first frame is white as well. A debug build waits for the Metro bundle, so the white time is longer than in a release build.

## Notes

- The publishable key is read from `.env` as `EXPO_PUBLIC_MEDUSA_PUBLISHABLE_KEY`. Every `/store` call needs it in the `x-publishable-api-key` header.
- Checked live with curl: `/health` returns 200, a wrong login returns 401 "Invalid email or password", `/store/products` without a key returns 400, and with the key returns 2 products.
- `/store/product-categories` returned 0 categories on the live backend, so the category tiles are empty until categories are created and products are assigned in Admin.
- Prices are shown as the backend sends them. A 1 kg tomato pack shows as 4500 INR. Check how the amounts were seeded.
- Orders, purchase orders, stock dashboard and reports still use mock data and were not touched.
- The dev client must be rebuilt (`npx expo run:android`) so the secure store module exists in the app, then restart Metro with `npx expo start --dev-client -c`.
- `expo-video` is a native module. The dev client must be rebuilt (`npx expo run:android`) before the splash video plays. The native splash change also needs a rebuild.
- The header image was made with Python (`pillow`, `scipy`) and `imageio-ffmpeg`, outside the repo. Only the finished files are committed. To change the curve or the colours, the artwork has to be rendered again.
- The splash video length (about 3 s) now sets how long the splash shows, because the app waits for the video to end.

## Next steps

- Create categories and upload real product images in Medusa Admin.
- Add a password reset email subscriber on the backend, since the forgot password screen cannot send an email without one.
- Connect the cart, checkout and orders to the Medusa store API.
- Add Terms and Privacy URLs to the register screen.
- Decide whether sign in should accept a mobile number, or change the first login field back to email only.
- Check the splash and login on iOS and on a release build.

## Test plan

**Automated tests (no test suite in this app, type check and lint passing):**
- [x] `npx tsc --noEmit`: passes with no errors.
- [x] `yarn lint`: passes with no errors.
- [x] Pre-push hook: blocked a push when lint failed (tested earlier).

**Manual checks (local):**
- [x] `curl` against the live backend: health, wrong login, the product list with the key and without it, and the categories list. Results are in Notes.
- [x] Splash video plays on the Android emulator (Pixel_10): the video starts, the zoom ends on white, then the login page fades in. Checked from screenshots, not on a physical device.
- [x] Login screen layout (50/50 split, curve, gradient, form spacing) checked from emulator screenshots.
- [ ] Splash and login on iOS, on a physical device and on a release build: not run.
- [ ] Login with a wrong or empty field on the new layout (error messages under the pill fields): not seen on screen.
- [ ] Valid login, register, session restore after app restart, logout and expired session: not run end to end on a device. The product list, infinite scroll, category filter and product details were also not run in the app.
- [ ] Infinite scroll loading text: not seen yet, because the live database has only 2 products and fits in one page of 10.
- [ ] Product images: not displayed, because the image host returned 500. The fallback icon was not seen on a device.
