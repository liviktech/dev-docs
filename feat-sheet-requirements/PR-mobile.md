# 2026-11-23 - sheet-requirements (mobile)

## Summary

This PR implements the mobile app changes for parent-child linking (requirement P1-016) and hardens parent dashboard UI stability. Students can generate shareable connection codes, enter parent connection codes, and manage or revoke parent access directly in their profile settings. Parents can generate codes for their children, redeem student codes, or send email requests.

Checked live: Expo TypeScript compilation (`tsc -p tsconfig.json --noEmit`) passes with zero errors. Fixed edge-case crashes related to uninitialized profiles, invalid date strings, and duplicate subject keys in the parent dashboard.

## What changed

| File | Change |
|---|---|
| `src/app/parent-requests.tsx` | Redesigned student parent access screen. Added styled invite code card with `selectable={true}`, copy/share button using `Share.share`, code cancellation, sanitized code redemption input, connected parents list, and revoke access confirmation dialog. |
| `src/app/add-child.tsx` | Added code exchange flow with tabbed switcher between "Code" and "Email". Parents can enter a child's code to link immediately or generate a parent code with share/copy functionality. |
| `src/app/parent/profile.tsx` | Added safe navigation for user email splitting, guarded against uninitialized links arrays, added null-safe fallbacks for child names, and corrected font weight typing. |
| `src/components/parent/academic-performance-section.tsx` | Hardened React list key from `key={subject.name}` to `key={`${subject.name}-${index}`}` to prevent duplicate key console errors when multiple curriculum tracks share a subject name. |
| `src/components/parent/child-profile-section.tsx` | Standardized `fontWeight` style definition to `'700'`. |
| `src/components/parent/link-status-badge.tsx` | Added styling and label for the `revoked` link status. |
| `src/hooks/use-parent.ts` | Added React Query hooks for student invites, parent invites, code redemption, and link revocation. |
| `src/hooks/use-selected-child.tsx` | Guarded query data with `Array.isArray` to guarantee `links` is always an array regardless of server response envelope. |
| `src/services/parent-api.ts` | Added endpoints and TypeScript types for `ParentInvite`, `CreatedInvite`, invite status, and link revocation. |
| `src/utils/parent-format.ts` | Hardened `initialOf`, `shortDay`, and `examDay` helpers to safely handle null, undefined, or invalid input values. |

## Why

1. **Parent-student linking (P1-016)**: Parents and students needed an effortless, mobile-native mechanism to connect accounts using short, shareable invite codes.
2. **UI & typography improvements**: The previous code presentation was awkward on smaller screens and could not be selected or shared via native sheets.
3. **Runtime safety**: Hardened helper functions against undefined states during initial application load and network interruptions.

## How

**Native share integration**: When a student or parent generates an invite code, a prominent "Share code" button triggers the native mobile share sheet (`Share.share`), allowing them to send the code directly via SMS, WhatsApp, or messaging apps.

**Input sanitization**: The code entry field automatically strips spaces and whitespace before sending to the backend, while keeping exact case sensitivity.

**Safe rendering guards**: `initialOf(name)` now checks `typeof name === 'string'` before calling `.trim()`, and date formatters check for valid timestamp values before formatting.

**Defense against duplicate keys**: The academic performance section uses composite keys combining the subject name and iteration index, ensuring React list reconciliation remains completely stable.

## Decisions made

**Native Share instead of plain clipboard on mobile.** Using `Share.share` provides an integrated system action sheet with copy to clipboard, SMS, and messaging apps in one touch, which is much more convenient on mobile devices.

**Non-blocking input.** Did not force automatic uppercase on code input so that user typing remains responsive and handles various keyboard auto-suggestions cleanly.

## Test plan

**Automated tests:**
- [x] `npx tsc -p tsconfig.json --noEmit`: type checking completed with 0 errors.

**Manual checks (local):**
- [x] Verified generating student code and sharing via native share sheet.
- [x] Verified redeeming code links student and parent accounts instantly.
- [x] Verified revoking parent access updates status badge and removes child view.
- [x] Verified parent dashboard renders with merged subject progress bars and no duplicate key warnings.
