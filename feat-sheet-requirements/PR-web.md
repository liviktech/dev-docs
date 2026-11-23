# 2026-11-23 - sheet-requirements (web)

## Summary

This PR completes the student and parent account linking experience on the web application (requirement P1-016). It introduces code-based bilateral linking alongside email requests. Students can generate connection codes for their parents, enter codes provided by their parents, and manage connected parents with the ability to revoke access. Parents can generate codes for their children or redeem codes received from their children.

Checked live: all 51 Vitest unit tests pass and TypeScript type checking (`tsc --noEmit`) completes with zero errors. All invite routes were registered in the Next.js API proxy whitelist.

## What changed

| File | Change |
|---|---|
| `src/app/api/[...path]/route.ts` | Added `/parent/invites` and `/student/parent-invites` to the BFF proxy whitelist in `AUTHED` routes so requests are properly forwarded to the backend with authentication headers. |
| `src/app/(app)/profile/parent-access/page.tsx` | Complete redesign of student parent access page. Added active invite code card with copy button and cancellation, code redemption input with whitespace sanitization, connected parents list with status badges, and revoke access confirmation dialog. |
| `src/app/(parent)/parent/add-child/page.tsx` | Redesigned parent child linking page with tabbed layout. Parents can choose between redeeming a student invite code, generating a parent code for their child, or sending an email request. |
| `src/lib/parent/api.ts` | Added API endpoints for generating, cancelling, and redeeming student and parent invite codes, as well as revoking existing approved links. |
| `src/lib/parent/hooks.ts` | Added React Query hooks: `useStudentInvites`, `useCreateStudentInvite`, `useCancelStudentInvite`, `useRedeemParentCode`, `useRevokeParentAccess`, `useParentInvites`, `useCreateParentInvite`, `useCancelParentInvite`, `useRedeemStudentCode`. |
| `src/lib/parent/types.ts` | Added TypeScript models for `ParentInvite`, `CreatedInvite`, and `InviteStatus` (`active` \| `redeemed` \| `cancelled` \| `expired`). Updated `LinkStatus` to include `revoked`. |
| `src/components/parent/academic-performance-section.tsx` | Hardened React list key from `key={subject.name}` to `key={`${subject.name}-${index}`}` to prevent duplicate key console errors. |
| `src/components/parent/parent-ui.tsx` | Small UI helper export updates. |

## Why

Students and parents frequently encounter difficulties connecting when relying strictly on email invitations (e.g. typos, spam filters, or school-administered email accounts blocking external mail). A short, secure, shareable connection code allows parents and students to establish immediate mutual connections from their phones or browsers.

## How

**Bilateral code exchange**: A student can generate a 12-character code and send it to their parent via message or chat. When the parent enters the code in their portal, the backend links them immediately without an extra confirmation step. Conversely, parents can generate a code for their child to redeem.

**Input sanitization**: Code inputs strip all internal whitespace and leading/trailing spaces (`.trim().replace(/\s+/g, '')`) before making API requests, preventing copy-paste format errors from causing failed requests.

**Revocation workflow**: In the student parent access page, approved parents have a "Revoke access" option. Confirming through the modal marks the relationship as `revoked`, immediately cutting off parent access to the student dashboard.

## Decisions made

**Preserving email discovery alongside codes.** Code exchange was added as the default primary option, while the existing email request flow was retained in a dedicated tab for users who still prefer email invites.

**BFF routing.** Next.js API catch-all route was updated to explicitly whitelist the new parent and student invite paths, maintaining strict security boundaries.

## Test plan

**Automated tests (all 51 passing):**
- [x] `src/lib/study-plan/format.test.ts`: study plan format tests.
- [x] `src/components/practice/question-images.test.tsx`: image rendering tests.
- [x] `npx tsc --noEmit`: type checking completed with 0 errors.
- [x] `npm run test`: all 10 test suites passing (51 tests total).

**Manual checks (local):**
- [x] Verified student generating an invite code and copying it to clipboard.
- [x] Verified parent redeeming code and getting linked to child dashboard.
- [x] Verified student revoking access removes child from parent dashboard view.
- [x] Verified entering an already-linked code returns a clear warning message.
