# 2026-10-28 - multi-language-support (frontend)

## Summary

The whole storefront was English only. This PR adds English, Tamil and Hindi with one language toggle that switches every page at once. The brand name "VegIn" is never translated. All screen text now comes from three dictionaries, product, category and collection names come back translated from Medusa, and backend error messages are shown in the chosen language.

Checked live in the browser: the toggle in Tamil on the home, store, category and product pages, and the category page in Hindi. **Hindi on the store and product pages, signed-in account, checkout and order pages, cart line items and the error message mapping were not checked live.** They compile (type check passes) but nobody clicked through them.

**Searching in Tamil or Hindi is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

About 120 existing files were touched, almost all of them a mechanical swap of hardcoded English text for a dictionary lookup. Only the files that carry the translation work are listed.

| File | Change |
|---|---|
| `src/lib/i18n/config.ts` (new) | Cookie name, the list of languages (UI code, Medusa code like `ta-IN`, label) and a helper that turns any stored value into a valid language. |
| `src/lib/i18n/dictionaries/en.ts` (new) | English text, about 380 keys. This is the source of truth for the key names. |
| `src/lib/i18n/dictionaries/ta.ts`, `hi.ts` (new) | Tamil and Hindi text. Typed so a missing key fails the type check. |
| `src/lib/i18n/dictionaries/index.ts` (new) | Loads the right dictionary for a language and exports the key type. |
| `src/lib/i18n/translate.ts` (new) | `createT`, looks up a key and fills `{placeholders}`. |
| `src/lib/i18n/server.ts` (new) | `getT()` and `getLocaleCode()` for server components. Reads the cookie. |
| `src/lib/i18n/client.tsx` (new) | `I18nProvider`, `useT()`, `useLocaleCode()`, the `T` component and `useTranslateError()` for client components. |
| `src/lib/i18n/errors.ts` (new) | Matches known backend error messages to dictionary keys. Unknown errors show a generic message in Tamil or Hindi. English keeps the original text. |
| `src/lib/i18n/catalog.ts` (new) | Translates option titles and values (for example "Pack Size", "1 kg") used in filters. |
| `src/lib/i18n/format.ts` (new) | Date formatting in the active language. |
| `src/lib/hooks/use-translated-titles.ts` (new) | The search index only has English titles, so this fetches translated product titles from the Store API for the hits on screen. |
| `src/lib/hooks/use-category-names.ts` (new) | Same idea for the category names in the store filters. |
| `src/modules/layout/components/language-switcher/index.tsx` (new) | The English / Tamil / Hindi toggle. Sets the language, then refreshes the page. |
| `src/modules/layout/components/language-select/index.tsx` (deleted) | Old country-style select that the switcher replaces. |
| `src/lib/data/locale-actions.ts` | New `setLanguage` action that writes the cookie. `getLocale` now maps old values like `ta` to `ta-IN`. |
| `src/app/layout.tsx` | Now async. Reads the language, sets `<html lang>`, builds the page metadata from the dictionary, wraps the app in `I18nProvider`, loads the Tamil and Devanagari fonts. |
| `tailwind.config.js`, `src/styles/globals.css` | Font stacks extended with the Tamil and Devanagari fonts. |
| `src/lib/util/brand.ts`, `src/lib/constants.tsx`, `src/lib/util/delivery-slot.ts` | Text moved out to dictionary keys. Slot labels and dates take the translate function and language. |
| `src/modules/checkout/components/error-message/index.tsx` | Now a client component that runs the message through `useTranslateError`. |
| `src/app/[countryCode]/(main)/` home, contact, privacy, returns, shipping, terms and product pages | Were `force-static`. Now dynamic so they can read the language cookie. |
| `src/modules/**` (nav, footer, search, store, product, cart, checkout, account, order, policy, not-found) | Hardcoded text replaced with dictionary keys. No layout or logic change. |
| `src/app/opengraph-image.tsx` | Uses the English tagline. No translation. |

## Why

The shop is for customers who mostly speak Tamil or Hindi, so an English only site was a barrier. The request was one toggle that changes the entire app, with product data and error messages included, and the brand name staying as VegIn.

The easy part was the UI text. The real work was making product, category and collection data follow the language, and keeping the search page right because the search index only knows English.

## How

**Language is a cookie**: `_medusa_locale` holds `en-US`, `ta-IN` or `hi-IN`. The root layout reads it, sets `<html lang>` and loads the dictionary. The toggle writes the cookie and calls `router.refresh()`, so one click changes every page.

**Dictionaries**: three flat TypeScript files with dotted keys (for example `cart.title`). `en.ts` defines the keys and `ta.ts` and `hi.ts` must have the same ones, so a missing translation is caught at type check time. Server components use `getT()`, client components use `useT()` or `<T>`.

**Product data**: the frontend already sends an `x-medusa-locale` header on Store API calls. With the cookie now holding `ta-IN` or `hi-IN`, Medusa returns translated titles, descriptions, categories and collections. The text itself is added by the backend PR.

**Search page**: the search index returns English titles, so the hit titles and the category filter names are fetched again from the Store API in the chosen language. Option names and values in the filters go through a small dictionary map.

**Errors**: backend messages are matched against known patterns and shown as the translated text. An unknown message shows a generic "Something went wrong" in Tamil or Hindi. English users still see the original text.

**Kept the same on purpose**: the brand name, prices, currency and order of the existing components. Only the text changed.

## Decisions made

**Dictionaries in TypeScript, not a translation library.** The first idea was three JS files. Three typed `.ts` files give the same result with compile-time key checking and no new dependency.

**Pages that were static are now dynamic.** Reading the cookie needs a request, so the home, policy and product pages lost `force-static`. This costs some caching. Serving one cached page per language would need the language in the URL, which is a bigger change and was not asked for.

**Search by Tamil or Hindi words is not supported.** The index holds English text only. Translating the index is a backend search change and was left out.

**Unknown backend errors show a generic message instead of English.** Mixing English into a Tamil page looked worse, and the exact text can be added to `errors.ts` as it is found.

## Notes

- Tamil and Hindi text is a first draft and has not been reviewed by a native speaker.
- Product descriptions end with placeholder text ("should be verified with the supplier", "for catalog setup") that customers can see. It is translated as it is. The real descriptions should be rewritten in English first.
- Lint still reports two errors in `cart.ts` and the side menu `Text` import. Both were there before this change.

## Next steps

- Native speaker review of `ta.ts`, `hi.ts` and the product translations.
- Replace the placeholder product descriptions.
- Click through signed-in account, checkout and order pages in Tamil and Hindi.
- Translate cart line item titles and, if wanted, add Tamil and Hindi search.

## Test plan

**Automated tests (type check passing, no test suite run):**
- [x] `npx tsc --noEmit`: passes for the changed files.
- [x] `npx eslint` on the changed files: no new errors. The two existing errors (`cart.ts`, side menu `Text`) are unrelated.
- [ ] Unit tests: none added. The project has no test setup for this area.

**Manual checks (local):**
- [x] Switched to Tamil with the toggle. The home, store, category and product pages changed language, and the brand name stayed "VegIn".
- [x] Store page in Tamil: product titles, category filters and the "Pack Size" / "1 kg" filter labels showed in Tamil.
- [x] Category page in Hindi showed Hindi text.
- [x] Main pages return 200 in Tamil and Hindi.
- [ ] Hindi on the store and product pages - not clicked through. Covered only by the type check and the Tamil run.
- [ ] Signed-in account, checkout and order pages - not clicked through in either language.
- [ ] Backend error messages in Tamil and Hindi - not triggered live. The matching is in `errors.ts` only.
- [ ] Cart line item titles in Tamil and Hindi - not checked.
