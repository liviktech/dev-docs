# 2026-11-21 - landing-page-redesign (web)

## Summary

The public home page (`/`) was a small placeholder: a logo, one headline and two buttons. This PR replaces it with a full Prepify landing page: navbar, hero, brand statement, "Why Prepify", features, a try-it-now sample question, an SAT and ACT section, a final call to action and a footer. It is all front end, with static content. No API calls, no backend change.

`npm run typecheck`, `npx eslint` on the changed files and `npx vitest run` (48 tests) all pass. I did not open the page in a browser, so the layout, the animations and dark mode were not checked visually.

**Pricing, sample questions and blog are not included on purpose** - the old placeholder comment said these would live on this page later. This PR does not add them. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/app/page.tsx` | Rebuilt as a list of the landing sections in order. The old inline header, headline and buttons are gone. |
| `src/app/globals.css` | Adds four brand colours (`--prepify-navy`, `--prepify-yellow`, `--prepify-blue`, `--prepify-surface-light`) for light and dark, registered in `@theme inline`. Adds six floating and pulse animations. The reduced motion rule now switches off all of them. |
| `src/components/landing/landing-nav.tsx` (new) | Sticky navbar with anchor links, theme toggle, Log in and Get started, and a mobile menu. |
| `src/components/landing/landing-hero.tsx` (new), `hero-illustration.tsx` (new) | Hero text with the two buttons, and the animated illustration beside it. |
| `src/components/landing/brand-statement.tsx` (new), `why-prepify.tsx` (new) | The statement strip and the three benefit cards. |
| `src/components/landing/features-showcase.tsx` (new) | Cards for Practice Tests, Mock Exams, Timed Tests and Study Plans. |
| `src/components/landing/interactive-question.tsx` (new) | One sample math question. Pick an answer, check it, see feedback, try again. |
| `src/components/landing/exam-journey.tsx` (new) | SAT and ACT cards. |
| `src/components/landing/final-cta.tsx` (new) | Closing banner with the sign up button. |
| `src/components/landing/landing-footer.tsx` (new), `info-modal.tsx` (new) | Footer links, and a modal for About, Contact, Privacy Policy and Terms. |
| `src/components/landing/landing.test.tsx` (new) | Tests for the landing components. |

## Why

The home page is the first thing a new student sees and it said almost nothing. The work was mostly the new sections and their look. The wiring is small: every button goes to `/signup` or `/login`, and the nav and footer links jump to sections on the same page.

## How

**Page order**: `page.tsx` renders `LandingNav`, then in `<main>` the hero, brand statement, why, features, challenge, exams and final CTA, then `LandingFooter`.

**Anchor links**: the nav and footer use `#why-prepify`, `#features`, `#challenge` and `#exams`. The hero "Explore Prepify" button goes to `#features`.

**Sample question**: the state lives in the component only. Nothing is sent to the server. After checking, a link to `/signup` is shown.

**Client components**: only the pieces that need state are `"use client"`: the nav (mobile menu), the sample question, the footer and the modal. The rest stay server components.

**Theme and motion**: colours come from the new CSS variables, so the dark theme follows the existing toggle. The animations are turned off under `prefers-reduced-motion`.

**Kept the same**: the logo path `/images/logo.png`, the `ThemeToggle` component and the `/login` and `/signup` routes are used as they were.

## Decisions made

**Static copy, no CMS or API.** The text for the modal pages (About, Contact, Privacy Policy, Terms) is written into `info-modal.tsx`. I did not add separate routes for them, to keep this PR small. The legal text is placeholder-level copy and needs a proper review before launch.

**Own brand colours.** I added `--prepify-*` variables instead of reusing the existing `brand` tokens, because the landing page needs a navy and a blue that the app does not use.

## Notes

- The two changed files and the new `landing` folder are uncommitted on `dev`. There is no feature branch yet, so the folder name `feat-landing-page` is my pick, not a branch name.
- Backend, mobile and admin repos have no changes for this work, so there is no PR file for them.
- `src/app/page.tsx` shows a "LF will be replaced by CRLF" git warning. Check the line endings before committing.

## Next steps

- Look at the page in a browser (light and dark, mobile width) before merging.
- Get real Privacy Policy and Terms text, and a real contact address, to replace the modal copy.

## Test plan

**Automated tests (48 passing across 9 files):**
- [x] `src/components/landing/landing.test.tsx`: nav links go to `/login` and `/signup`, hero text and buttons, why cards, feature cards, SAT and ACT cards, final CTA.
- [x] `src/components/landing/landing.test.tsx`: sample question starts with Check answer disabled, option B shows the correct message and the `/signup` link, Try again resets, option A shows the incorrect message.
- [x] `src/components/landing/landing.test.tsx`: footer Privacy Policy opens the dialog and Close dialog removes it.
- [x] `npm run typecheck`: passes.
- [x] `npx eslint src/app/page.tsx src/components/landing`: no output, no errors.

**Manual checks (local):**
- [ ] Open `/` in light and dark theme, desktop and phone width - not done, I did not open a browser. The tests only cover text, links and clicks, not layout.
- [ ] Click the nav and footer anchor links and check they scroll to the right section - not done, the tests only check the hero link target.
- [ ] Check the animations stop with reduced motion on - not done, the CSS rule was only read.
