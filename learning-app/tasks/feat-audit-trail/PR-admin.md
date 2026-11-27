# 2026-11-24 - feat-audit-trail (admin)

## Summary

This PR implements the administrative audit trail interface in `learning-app-admin` (Phase 1 case P1-059).

Prior to this change, administrators had no interface to inspect sensitive system activities, track content revisions, monitor parent link approvals, or review administrative account actions.

This PR adds the `/audit` page featuring real-time log querying, action domain filtering (`content.*`, `taxonomy.*`, `exam.*`, `parent.*`, `auth.*`), result status filtering (`all`, `success`, `failure`), an interactive JSON details drawer for deep inspection of metadata, an RFC 4180 CSV export trigger, and navigation integration in the sidebar menu.

All existing administrative features (content management, taxonomy editor, mock exams, user statistics, abuse protection settings) continue working without disruption. All 107 Vitest unit tests pass across 16 test suites, TypeScript compilation passes with zero errors, Vite builds cleanly for production, and oxlint linter passes with zero warnings.

## What changed

| File | Change |
|---|---|
| `src/features/audit/schemas/audit.schemas.ts` | Defined Zod validation schemas for `AuditEvent`, `PaginatedAuditLogs`, and `AuditLogFilterValues`. |
| `src/features/audit/lib/audit-formatters.ts` | Added user-friendly UUID truncation, humanized action key translations, and formatted actor and target label resolvers. |
| `src/features/audit/lib/audit-formatters.test.ts` | Added unit tests verifying UUID detection, short ID generation, action humanization, and actor/target formatting. |
| `src/features/audit/components/copy-chip.tsx` | Built a compact, copyable ID badge component with clipboard integration and hover tooltips. |
| `src/features/audit/api/audit.api.ts` | Implemented `fetchAuditLogs`, `fetchAuditEventById`, and query parameter builder with RFC 4180 CSV download helper. |
| `src/features/audit/api/audit.queries.ts` | Implemented React Query hooks `useAuditLogs` and `useAuditEvent` with 30-second stale time caching. |
| `src/features/audit/pages/audit-page.tsx` | Built the full `AuditPage` component with user-friendly actor and target displays, copyable ID chips, humanized actions, filter toolbar, tabular log display, details modal, and pagination. |
| `src/features/audit/pages/audit-page.test.tsx` | Added component tests verifying empty state rendering, log table display, details modal opening, raw UUID fallback handling, and domain filter query generation. |
| `src/layouts/dashboard-layout.tsx` | Added `{ to: '/audit', label: 'Audit Trail', icon: <ShieldCheckIcon /> }` to the main navigation menu. |
| `src/routes/routes.tsx` | Registered the `/audit` route mapping to `AuditPage`. |

## Why

Case P1-059 mandates an administrative interface for compliance and forensic analysis:
1. Operational Transparency: Administrators need a unified view to track who published questions, updated taxonomy nodes, or modified exam blueprints.
2. Rapid Incident Investigation: When an error or unexpected change occurs, staff can immediately filter by action domain or actor email to inspect the change payload.
3. Compliance Reporting: Administrators must be able to export audit records into CSV format for external auditing and compliance archiving.

## How

Domain-Based Filtering: The toolbar provides a streamlined domain filter dropdown allowing staff to isolate actions across Content, Taxonomy, Exam Config, Parent Links, and Authentication without typing raw action keys.

Inspect Details Drawer: Clicking the Inspect button on any log entry opens a dialog rendering the structured JSON payload, actor IP address, and browser user agent in a formatted, monospaced view.

CSV Export Integration: Clicking Export CSV constructs the active query string (respecting date ranges, actor email, and domain filters) and triggers a direct browser download of the RFC 4180 CSV report.

Resilient Zod Parsing: All API responses are validated at runtime via `auditEventSchema` and `paginatedAuditLogsSchema`, ensuring malformed or null fields do not crash the view.

## Decisions made

Wildcard Domain Filtering Dropdown: Rather than exposing dozens of granular action strings in a flat dropdown, grouping by domain (`content.*`, `taxonomy.*`, etc.) simplifies filtering for everyday operational tasks.

In-Memory CSV Download via Blob: Export uses the authenticated API client to fetch the CSV payload and generates a client-side Blob URL download, maintaining seamless JWT authentication without opening insecure raw browser tabs.

Humanized Action Badges: Raw action keys like `content.question.publish` are visually formatted with color-coded badges to differentiate creation, modification, publication, and deletion events at a glance.

## Notes

- No backend code was modified in this repository.
- All modified files conform to Prettier and oxlint standards.

## Test plan

Automated component and unit tests (107 tests passing):
- [x] `src/features/audit/pages/audit-page.test.tsx`: verifies empty state, table populating with logs, details modal opening with formatted JSON, and domain filter query generation.
- [x] All 16 test suites in `learning-app-admin` pass cleanly.
- [x] `npm run typecheck` (`tsc -b`) passes with 0 errors.
- [x] `npm run build` (`vite build`) completes successfully.
- [x] `npx oxlint` passes with 0 warnings.

Manual verification:
- [x] Verified navigation menu displays 'Audit Trail' with shield icon.
- [x] Verified domain filter selects `content.*`, `taxonomy.*`, `exam.*`, `parent.*`, `auth.*` accurately.
- [x] Verified inspect modal displays details JSON and actor IP.
