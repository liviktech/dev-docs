# 2026-11-24 - rive-confetti-rate-limits-audit-headers (mobile)

## Summary

Three things on this branch. The "You are all set" screen after onboarding now plays a full-screen Rive confetti animation. The app now handles the backend's new 429 rate limit answers (P1-058) with clear messages and automatic retries for answer saves. Every API request now carries audit headers (P1-059) so the backend can tie a sensitive action to one request and one device. A missing web dependency that broke the EAS build was also added.

None of this was run against the real backend or on a device. Type check and lint pass, a local Android export bundles, and a small script checked the retry helper. See "Test plan".

**Sign out of all devices is not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/components/ui/rive-overlay.tsx` (new) | Reusable full-screen Rive animation. Autoplay, `Fit.Cover`, wrapped in a `View` with `pointerEvents="none"` so taps pass through. |
| `src/app/(onboarding)/welcome.tsx` | Renders `<RiveOverlay>` with `assets/rive/party-popper.riv` over the screen. No layout change. |
| `assets/rive/party-popper.riv` (new) | The confetti animation (about 230 KB). |
| `metro.config.js` (new) | Adds `riv` to Metro asset extensions. |
| `src/services/rate-limit.ts` (new) | Reads `Retry-After` (60 s if missing), builds the 429 messages, and runs answer saves with a shared wait-and-retry. |
| `src/services/api-client.ts` | `ApiError` gets `retryAfterSeconds`. A 429 becomes an `ApiError` with the right message. Answer and submit calls go through the retry helper. Adds the audit headers to every request. |
| `src/services/client-headers.ts` (new) | Builds `x-request-id`, `User-Agent` and `x-client-platform`. Native only. |
| `src/app/(auth)/verify-email.tsx` | A rate-limited resend restarts the "Resend code" countdown at the `Retry-After` value. |
| `package.json`, `yarn.lock` | Adds `rive-react-native`, `@lottiefiles/dotlottie-react` and `expo-crypto`. |

## Why

The welcome screen is the first thing a student sees after finishing onboarding, so it should feel like a finish line. The backend now returns 429 on OTP, login, invite redeem and answer calls, and the app needs to explain this and not lose answers. The audit trail needs a request ID and device context from the client to be useful in disputes.

## How

**Rate limit messages**: the API client turns any 429 into an `ApiError` with the text from the spec, chosen by path (code requests, sign-in attempts, redemption attempts). `INVITE_LOCKED` gets the 5 minute lockout text. Screens already show `error.message`, so login, OTP and the invite alerts needed no change.

**Answer saves keep the student's choice**: any call ending in `/answer` or `/submit` (practice, mock exam, and section submits) waits for the `Retry-After` window on a 429 and tries again by itself. All such calls share one pause. It retries up to 3 times and gives up if the wait is over 30 seconds. No error is shown while it waits.

**OTP resend**: the 60 second countdown already existed. A 429 on resend now sets it to the server's wait time.

**Audit headers**: `x-request-id` is `req-<uuid v4>` (the backend accepts this format), `User-Agent` is `LearningAppMobile/<version> (<iOS|Android>; <os version>; <device model>)`, and `x-client-platform` is `ios` or `android`. They are added in the one place every request passes, so invites, link approvals and account deletion all carry them.

**Build fix**: `app.json` sets `web.output` to `server`, so the EAS step `expo export:embed --eager` also bundles the web version. `lottie-react-native`'s web file imports `@lottiefiles/dotlottie-react`, which was not installed, and the build failed. Installing it fixed that step.

## Decisions made

**No separate answer queue.** The spec suggested a background queue with 100 ms spacing. In practice mode the student needs the right answer and explanation straight back from each save, so a queue would delay it. I used a shared wait-and-retry instead. The 60 per minute limit is hard to reach at human tapping speed, so I left out the 100 ms spacing.

**Audit headers are native only.** The backend CORS list allows only `Content-Type`, `Authorization` and `x-request-id`, so extra headers would make browser requests fail. The web app would need the backend CORS list changed first.

**Sign out of all devices is not built.** The app has no button for it, and its logout does not send a `revokeAll` flag. It needs new UI, so it was left out.

## Notes

- `rive-react-native` and `expo-crypto` are native modules. They need a new native build (`npx expo run:android` or EAS). A Metro reload is not enough.
- The Rive file's looping is set inside the file, not in code. If the confetti does not show, `artboardName` or `stateMachineName` may need to be passed to `RiveOverlay`.
- Google sign-in `DEVELOPER_ERROR` and `12500` were looked at in the same session. They came from Firebase and Google Cloud setup (Android SHA-1 and OAuth clients), and `google-services.json` was updated in a separate commit. There is no other code change from it.
- Google sign-in 429s show the generic sign-in message, with no countdown.
- The iOS Google sign-in plugin is skipped on EAS because `GOOGLE_IOS_URL_SCHEME` is not set there. This does not affect Android builds.

## Next steps

- Add a "Sign out of all devices" action if the audit trail should record `auth.session.logout_all`.
- Allow the new headers in the backend CORS list if the web app should send them too.

## Test plan

**Automated tests (type check and lint passing, no unit test files for this change):**
- [x] `npx tsc --noEmit`: passes with no errors.
- [x] `npx expo lint`: passes with no output.
- [x] `rate-limit.ts` retry helper: a throwaway script showed one limited call is retried and returns `ok`, and the messages read correctly. The script was not kept.

**Manual checks (local):**
- [x] `npx expo export --platform android` and `expo export:embed --eager`: both bundle with no errors after the dependency fix.
- [ ] Confetti plays on the welcome screen and the dashboard button still works - not checked, needs a native build on a device.
- [ ] 429 messages and the resend countdown against the real backend - not checked.
- [ ] Rapid answer taps in practice and mock exam while the limit is hit - not checked.
- [ ] Headers visible in backend logs and audit records - not checked.
- [ ] EAS build after the dependency fix - not re-run.
