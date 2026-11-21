# 2026-11-14 - theme-auth-profile-tests (frontend)

## Summary

The app had a purple look, a plain login, and placeholder Tests and Profile tabs. This PR moves the whole app to the new yellow and navy theme with Inter and IBM Plex Mono fonts, redesigns login and signup, adds the onboarding steps after exam and subject selection, adds a mock practice test flow in the Tests tab, builds a real Profile tab with logout, and changes the splash screen to the new logo on white. Everything uses mock data because the backend is not connected yet.

**Tutor flow is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/constants/theme.ts` | New colour tokens (yellow accent, blue link, navy text, pale page), new `link`, `linkSoft` and `onAccent` tokens, and the font setup (Inter and IBM Plex Mono). This is now the one place to change fonts and colours |
| `src/components/themed-text.tsx` | Picks the font by text role (heading, card heading, body, label, stat, button). New types `label`, `stat` and `button` |
| `src/app/_layout.tsx` | Loads the fonts before hiding the native splash, and registers the practice route and the practice provider |
| `src/app/index.tsx` | Splash is now the Prepify logo centred on white, then goes to login |
| `app.json` | Native splash is plain white with no image. Added the `expo-build-properties` plugin |
| `src/components/auth/auth-shell.tsx` | New layout: tinted header with logo and headline, white rounded sheet with a handle, bottom padding for the Android navigation bar and iOS home indicator |
| `src/components/auth/auth-field.tsx` | Uppercase mono labels, optional right side label (for Forgot password) and optional icon |
| `src/components/auth/primary-button.tsx` | Yellow pill button with dark bold text |
| `src/components/auth/social-login.tsx` (new) | OR divider and Google and Apple buttons with SVG logos. Apple first on iOS, Google first on Android |
| `src/components/auth/mail-icon.tsx` (new) | Mail icon for the email field |
| `src/app/(auth)/login.tsx`, `src/app/(auth)/signup.tsx` | New layout, social buttons, and they now save name and email into the study state |
| `src/app/(onboarding)/role.tsx`, `profile-setup.tsx`, `academic.tsx` (new) | Role, profile and school details steps. Exam and subjects screens now show 5 steps |
| `src/components/onboarding/option-button.tsx` (new) | Single choice button used by the new steps |
| `src/state/study-context.tsx` | Added profile (name, email, role, gender, bio, location, school, grade), `updateProfile` and `reset` |
| `src/state/practice-context.tsx` (new) | Answers, last score, start, end and reset for the practice test |
| `src/data/mock-practice.ts` (new) | Five mock questions |
| `src/app/(tabs)/tests.tsx` | One Practice tests card that starts the quiz and shows the last score |
| `src/app/practice/_layout.tsx`, `src/app/practice/[index].tsx` (new) | One question per page, green and red answer states with flash, bounce and shake animations, and an End test button on the last page |
| `src/app/(tabs)/profile.tsx` | Profile card, progress tiles, target card and Log out that clears the state and goes to login |
| `src/components/app-tabs.tsx` | Tab names always shown, iOS glass effect, dark selected label |
| `src/app/(tabs)/hub.tsx`, `src/components/hub/*`, `src/components/dashboard/summary-card.tsx`, `src/components/onboarding/choice-card.tsx`, `src/data/mock-dashboard.ts` | Colour changes only (yellow fills, dark text on yellow, blue link text) |
| `.agents/skills/expo-native-ui/SKILL.md`, `references/visual-effects.md` | New rule and example for hold and drag liquid glass buttons on iOS |
| `package.json`, `yarn.lock` | Added the font packages, `react-native-svg`, `expo-build-properties`, `lodash` and `sonner-native` |
| `eas.json` | No real change from this work |

## Why

The reference screens asked for a new look (yellow and navy theme, Inter and IBM Plex Mono), a login layout with social sign in, onboarding after subject selection, a practice test in the Tests tab, and a real profile with logout. The theme was the real work because colours were spread across many files, so the colours now live in `theme.ts` and the font choice lives in one place.

## How

**Fonts**: `theme.ts` exports the font files and `ThemedText` chooses the family from the text type. Inter Bold for main headings and buttons, Inter SemiBold for card headings, Inter Regular for body text, and IBM Plex Mono Medium for uppercase labels and stats. Heading sizes are kept at 18 on purpose, two sizes above the 16 body text.

**Theme**: Yellow is used for fills, and blue (`link`) is used for any accent text, because yellow text is not readable on white. Text placed on yellow uses the `onAccent` dark colour.

**Practice test**: The question index is a route param, so each question is its own page and the back stack is the page navigation. Answers are locked after the first pick. The score is stored when End test is pressed and shown on the card.

**Logout**: Resets the study state and the practice state, then replaces the route with login so the user cannot go back.

**Android navigation bar**: The auth sheet adds the bottom safe area inset to its padding so the links stay above the system buttons.

## Decisions made

**Tutor flow is not built.** Both Student and Tutor go to the student app for now, as asked. The role is saved in state so the tutor flow can be added later.

**Heading sizes stay small.** The reference screens have bigger headings, but the rule from the earlier request (headings only two sizes above body text) was kept.

**Splash needs a rebuild.** The white native splash with no image only applies after a rebuild (`npx expo prebuild --clean` or a new EAS build). It will not show in Expo Go.

**Gradient header was not used.** `expo-linear-gradient` is not installed, so the login header is a flat tint with a soft glow.

**Profile photo upload was skipped.** It needs `expo-image-picker`. The avatar shows the first letter of the name.

## Bugs found while testing (already there before, not fixed here)

**1. Type error in `src/components/app-tabs.web.tsx`.** `tsc` reports a ref type mismatch around line 45. The file was not touched by this work. It only affects the web tab bar.

## Notes

- Profile progress tiles, the "On track" chip and the practice questions are mock data.
- Google and Apple buttons do nothing yet. They are ready to be wired to the auth provider.
- Nothing is saved to the device. Profile data and the score reset when the app is closed.
- `expo-blur`, `expo-video` and `expo-haptics` are not installed. The package age rule (24 hours) blocked the first two, and `expo-haptics` was not tried.
- `@expo-google-fonts/figtree` and `@expo-google-fonts/roboto` are still in `package.json` but no longer used.
- The Android adaptive icon background in `app.json` is still the old dark purple (`#0F0B1A`).
- The Hub and dashboard numbers do not use the mono font yet.
- Social logos use `react-native-svg`, so a development build needs a rebuild.

## Next steps

- Wire login, signup and the social buttons to the backend.
- Build the tutor flow.
- Move Hub and dashboard numbers to the mono font.
- Build the hold and drag liquid glass button as a real component and use it for the translucent buttons on iOS.
- Remove the unused Figtree and Roboto packages.
- Fix the type error in `app-tabs.web.tsx`.

## Test plan

**Automated tests (no test files in this repo, so type check and lint only):**
- [x] `npx tsc --noEmit`: no errors in `src` except the existing one in `app-tabs.web.tsx`
- [ ] `npx expo lint`: the command printed nothing, so a pass was not confirmed. Run it once more before pushing

**Manual checks (local):**
- [ ] Not checked on a device or simulator. No screen was run, so the layouts, fonts, animations and the Android navigation bar fix have only been checked through the type checker
- [ ] Login to exam to subjects to role to profile to school details to Hub: needs a run on a device
- [ ] Practice test: answer all five questions, press End test, and check the score on the Tests card: needs a run on a device
- [ ] Profile Log out goes back to login: needs a run on a device
- [ ] Splash on white after a rebuild: needs a rebuild and a device
- [ ] Apple and Google button order on iOS and on Android: needs both platforms
