# Plan: Replace Medusa starter branding with vegetable store branding

Roadmap: Phase 2, item 4. Priority P1.

## What is wrong today

The storefront still looks like the Medusa demo. Found in the code:

- Store name "Medusa Store" is set in `src/lib/util/site.ts` (`STORE_NAME`). The same words are repeated by hand in the navbar (`src/modules/layout/templates/nav/index.tsx`), the footer (`src/modules/layout/templates/footer/index.tsx`), the checkout header (`src/app/[countryCode]/(checkout)/layout.tsx`), the side menu, and the page titles of categories, collections, account and profile pages.
- Home page title is "Medusa Next.js Starter Template" with a starter description (`src/app/[countryCode]/(main)/page.tsx`).
- The hero (`src/modules/home/components/hero/index.tsx`) says "Ecommerce Starter Template, Powered by Medusa and Next.js" and has a "View on GitHub" button. It has no picture.
- The footer links to GitHub and Medusa docs. There is a "Powered by Medusa" block (`src/modules/layout/components/medusa-cta`).
- The sign up page says "Become a Medusa Store Member".
- There is no logo, no favicon change, and no contact details.

## What we are changing

1. Make one place for brand text. Put the store name, tagline, phone, WhatsApp number, email and address in `src/lib/util/site.ts` (and read them from env values where it makes sense). Every other file reads from there instead of typing the name.
2. Replace every "Medusa Store" and starter text listed above with the brand values.
3. New hero: a real fresh vegetable photo as background, a short headline such as "Fresh vegetables, straight from the farm", a one line promise, and a "Shop now" button to `/store`. Remove the GitHub button.
4. Under the hero add a small strip with three trust points: Farm fresh, Delivered within a day, Easy returns for bad produce. The wording is a draft for the owner to approve.
5. Footer: remove GitHub and Medusa links and the Medusa call to action. Add contact details, WhatsApp link, and links to the existing shipping, returns, terms and privacy pages.
6. Navbar: add a logo (SVG) next to or in place of the text name.
7. Add a favicon and set page titles with a pattern: `Page name | Brand`. Add proper meta descriptions and the Open Graph image.
8. Fix account page copy: "Become a member" and "Create your profile" instead of the Medusa wording.
9. Make colours and fonts fit a fresh produce look (green accent). Change the design tokens in one place (Tailwind config or global CSS), not in each component.

## Why we are changing it

First impressions decide if a customer trusts the shop. The starter text makes the store look unfinished. A single source for brand text also means a rename later takes one edit.

## Files we expect to touch (frontend only)

- `src/lib/util/site.ts`
- `src/modules/layout/templates/nav/index.tsx`, `footer/index.tsx`, `side-menu/index.tsx`
- `src/modules/layout/components/medusa-cta/` (remove use)
- `src/modules/home/components/hero/index.tsx`
- `src/app/[countryCode]/(main)/page.tsx` and the other `generateMetadata` titles
- `src/app/[countryCode]/(checkout)/layout.tsx`
- `src/modules/account/components/register/index.tsx` (text only)
- `public/` (logo, favicon, hero image), Tailwind config

## Risks and things to watch

- **We do not have a brand name, logo or photos yet. The owner must supply them. Until then we use clear placeholders, and we will say so in the pull request.**
- Hero photos must be ones we have the right to use.
- Large hero images slow the page. Use a compressed image and the Next.js image component.
- The login and register screens were checked and already move to the account page after success, so no separate auth plan exists.
- Policy pages (returns, shipping) read numbers from region metadata. Do not hard code them.

## Out of scope

- A full redesign of every page layout.
- Writing the legal text of the terms and privacy pages.
- Tamil language switch.

## How to test

- Search the source for "Medusa" and "Starter" and "github": only code imports and package names remain.
- Check the browser tab title on home, category, collection, account and checkout.
- Hero, trust strip, footer and logo look right at phone, tablet and desktop width.
- Lighthouse performance on the home page stays reasonable (no big drop).
