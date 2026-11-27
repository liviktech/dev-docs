# 2026-10-28 - multi-language-support (backend)

## Summary

Medusa had no Tamil or Hindi data, so the storefront could only show English product names. This PR turns on Medusa's translation module and adds a seed script that writes Tamil and Hindi names and descriptions for products, categories and collections. The storefront then gets translated data just by sending the `x-medusa-locale` header it already sends.

Checked live: Store API calls with `x-medusa-locale: ta-IN` and `hi-IN` return translated products, descriptions, options, categories and collections.

**Translating the search index is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `medusa-config.ts` | Added the translation module (`@medusajs/medusa/translation`) to `modules`, with a comment explaining why. |
| `.env.template` | Added `MEDUSA_FF_TRANSLATION=true` with a short comment. |
| `src/scripts/seed-catalog-translations.ts` (new) | Adds the `ta-IN` locale, sets the store's supported locales to `en-US`, `ta-IN` and `hi-IN`, marks the fields as translatable, and writes the translations. Safe to run twice. |
| `src/scripts/data/catalog-translations.ts` (new) | The Tamil and Hindi text: 100 product titles, 95 origin sentences, the description ending, 6 categories with descriptions, 4 collections, the "Pack Size" option and the "1 kg" value. |

## Why

The storefront now has a language toggle. UI text lives in the frontend, but product, category and collection names live in Medusa, so they need translations stored there. Using Medusa's own translation module means shop owners can edit the text later in Medusa Admin instead of changing code.

## How

**Module and flag**: the translation module is behind the `MEDUSA_FF_TRANSLATION` flag. `defineConfig` checks the flag before the `.env` file is fully loaded, so the module was not picked up from the flag alone. Listing it explicitly in `medusa-config.ts` fixes that. The flag is still needed for the feature to be active.

**Seed script**: it runs in five steps. Add missing locales, update the store's supported locales, set which fields are translatable for each entity (product title and description, variant title, option title, option value, category name and description, collection title), build the translation rows by looking up each English text in the data file, then create new rows and update existing ones through `batchTranslationsWorkflow`. Anything with no translation is listed in the log instead of failing.

**Data file**: English text is the key, and each value is a Tamil and Hindi pair. Product descriptions are rebuilt from the title, the origin sentence and the fixed ending, so only the parts that vary need a translation.

**Run it with**: `npx medusa db:migrate` once (creates the translation tables), then `npx medusa exec ./src/scripts/seed-catalog-translations.ts`.

## Decisions made

**Explicit module entry in `medusa-config.ts`.** Relying on the feature flag alone gave "Could not resolve 'translation'". Adding the module by hand is a small, safe change.

**Seed script instead of a migration.** Translations are content, not schema. A script can be re-run when text changes and does not tie data to the migration history.

**Search index left in English.** `/store/search` returns English titles, so the frontend fetches translated titles for the hits shown. Making the index itself multilingual is a larger search change.

## Notes

- The local database was changed while testing: translation tables were created by `db:migrate`, the `ta-IN` locale was added, the store's supported locales were updated, and 820 translations were written. Other environments need `db:migrate` and the seed script run once.
- Tamil and Hindi text is a first draft and has not been reviewed by a native speaker. Owners can fix it in Medusa Admin under Translations.
- Every product description ends with placeholder text ("should be verified with the supplier", "for catalog setup"). It is translated as it is. It would be better to rewrite the English descriptions first.
- The `.env` file also got the flag locally. Its values were not printed or copied anywhere.
- Medusa Admin has its own separate language setting.

## Next steps

- Native speaker review of `catalog-translations.ts`.
- Replace the placeholder product descriptions, then re-run the seed script.
- Decide whether search should work in Tamil and Hindi.

## Test plan

**Automated tests (type check passing, no test suite run):**
- [x] `npx tsc --noEmit`: passes for the changed files.
- [ ] Unit or integration tests: none added for the seed script.

**Manual checks (local):**
- [x] `npx medusa db:migrate` after enabling the module: fixed the earlier "relation does not exist" error (42P01).
- [x] Ran the seed script: 820 translations written, store locales updated.
- [x] Store API with `x-medusa-locale: ta-IN` and `hi-IN`: translated products, descriptions, options, categories and collections came back.
- [x] Storefront store page in Tamil showed translated titles, category filters and "Pack Size" / "1 kg".
- [ ] Running the seed script a second time - written to be repeatable, but the second run was not checked.
- [ ] Fresh database from scratch - not checked. Only the local database was migrated and seeded.
