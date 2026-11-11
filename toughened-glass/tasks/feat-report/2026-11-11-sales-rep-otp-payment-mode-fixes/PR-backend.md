# 2026-11-11 - sales-rep-otp-payment-mode-fixes (backend)

## Summary

The forgot password request used to do nothing when the mobile number matched no login, and still answered success, so the user reached the OTP page for a number that never got an OTP. It now returns an error: "This mobile number is not registered."

This was not run or tested. It needs a backend restart to apply.

## What changed

| File | Change |
|---|---|
| `src/modules/auth/password-reset.service.ts` | `requestOtp` throws `BadRequestError` (`MOBILE_NOT_REGISTERED`) when no login owns the number. The comment above it is updated to match |

## Why

The user wanted an error when the number does not belong to any employee, instead of an OTP page for a number that was never sent anything.

## How

**Unknown number**: `findUserByMobile` finds nothing, so the request is rejected before the cooldown and channel rule checks. A known number follows the same path as before.

## Decisions made

**Reveal whether a number is registered.** The old behaviour was deliberate, so nobody could use the form to find out which numbers are registered. This PR reverses that on request. The other option was to keep the generic message and only change the screen wording.

## Notes

- If no channel (email or WhatsApp) is switched on for the "Forgot password OTP" event, the backend only logs a warning and the user still sees success. This was not changed.

## Next steps

- Rate limit the forgot password endpoint, since it can now be used to check whether a number is registered.

## Test plan

**Automated tests (none run):**
- [ ] No test was added or changed for `requestOtp`
- [ ] Build and type check: not run

**Manual checks (local):**
- [ ] Unknown number shows "This mobile number is not registered." - not checked
- [ ] Registered number still reaches the OTP page and gets an OTP - not checked
