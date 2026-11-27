# 2026-11-24 - feat-audit-trail (backend)

## Summary

This PR implements an append-only audit trail and sensitive-action event recording across the backend (Phase 1 case P1-059).

Prior to this change, administrative and sensitive lifecycle operations (such as question publication, taxonomy changes, exam configuration updates, parent link approvals, and account deletions) did not record structured audit entries with actor identity, action type, target resource, and outcome status.

This PR introduces a dedicated `audit_events` table backed by a PostgreSQL trigger that strictly rejects updates and deletes, an `AuditModule` with `AuditService`, `AuditRepository`, and `AuditController`, comprehensive audit recording across 5 sensitive operational domains, an admin endpoint for paginated audit log queries and CSV export, and Check 9 in the automated security verification suite (`npm run verify:security`).

All existing admin operations, question publishing workflows, exam configurations, parent linking flows, and student authentication continue working without disruption. All 209 unit tests pass across 21 test suites, TypeScript compiles with zero errors, and all 9 security verification checks pass cleanly.

## What changed

| File | Change |
|---|---|
| `src/app.module.ts` | Registered global `AuditModule` into the root application imports. |
| `src/database/migrations/0019_audit-events.sql` | Created `audit_events` table with indexes on `created_at`, `action`, `actor_id`, and `target_type`, plus trigger enforcing append-only immutability. |
| `src/modules/audit/audit.controller.ts` | Added `GET /admin/audit-logs`, `GET /admin/audit-logs/export`, and `GET /admin/audit-logs/:id` protected by `AdminRoleGuard`. |
| `src/modules/audit/audit.module.ts` | Created `@Global()` `AuditModule` exporting `AuditService` and `AuditRepository`. |
| `src/modules/audit/audit.repository.ts` | Implemented `insert` with auto-resolved actor email from `users`, `findPaginated` with dynamic filtering, prefix queries, and joined resolution of actor emails and exam names, and `findById`. |
| `src/modules/audit/audit.service.ts` | Implemented `record` with error isolation and transaction support, `list`, `getById`, and RFC 4180 `exportCsv`. |
| `src/modules/audit/audit.types.ts` | Defined audit types, query DTO schema with Zod, and pagination interfaces. |
| `src/modules/auth/auth.service.ts` | Added audit recording for `auth.admin.register` and `auth.account.delete`. |
| `src/modules/exams/exams.controller.ts` | Wired audit logging for `exam.config.create`, `exam.config.update`, `exam.config.publish`, `exam.config.unpublish`, and `exam.config.delete`. |
| `src/modules/exams/exams.service.ts` | Injected `AuditService` into `ExamsService` and ensured `archive` records `targetLabel` from exam template header. |
| `src/modules/parent/parent-invites.service.ts` | Added audit recording for `parent.invite.create`, `parent.invite.cancel`, and `parent.invite.redeem`. |
| `src/modules/parent/parent-requests.controller.ts` | Added audit recording for `parent.link.approve`, `parent.link.revoke`, and `parent.link.reject`. |
| `src/modules/questions/questions.controller.ts` | Added audit recording for `content.question.bulk_import`. |
| `src/modules/questions/questions.service.ts` | Added audit recording for `content.question.create`, `content.question.update`, `content.question.publish`, `content.question.archive`, and `content.question.restore`. |
| `src/modules/taxonomy/taxonomy.controller.ts` | Injected `AuditService` and extracted IP and user-agent for taxonomy modifications. |
| `src/modules/taxonomy/taxonomy.service.ts` | Added audit recording for `taxonomy.node.create`, `taxonomy.node.update`, `taxonomy.node.archive`, `taxonomy.node.restore`, and `taxonomy.node.delete`. |
| `src/scripts/verify-security.ts` | Added Check 9 ("Audit Trail & Sensitive Action Logging Coverage") verifying table migration, controller endpoints, and domain coverage. |
| `test/unit/audit.service.spec.ts` | Added unit test suite for `AuditService` covering record, error isolation, getById, CSV generation, and schema parsing. |
| `test/unit/security-verification.spec.ts` | Updated unit test to assert that all 9 security checks pass including Check 9. |

## Why

Case P1-059 requires a persistent, tamper-evident audit log for administrative and sensitive operations:
1. Operational Accountability: Administrative actions like publishing exams, altering category hierarchies, or modifying questions directly impact student learning paths. An immutable log establishes who took what action, when, and on which resource.
2. Forensic Readiness: In case of accidental data corruption or unauthorized administrative activity, teams must be able to trace the blast radius and timeline of operations.
3. Compliance and Security Standards: Phase 1 enterprise readiness mandates separation of duties and tamper-resistant audit trails with append-only integrity at the database layer.

## How

Append-Only PostgreSQL Trigger: In `0019_audit-events.sql`, the `audit_events` table utilizes the existing `prevent_direct_event_mutation()` stored procedure to reject `UPDATE` and `DELETE` commands at the database engine level.

Fail-Safe Error Isolation: In `AuditService.record`, non-transactional audit records catch and log database exceptions using structured Pino logs without failing the user operation. When a transaction client `tx` is provided, audit writes share the transaction boundary to ensure atomicity.

Domain Action Coverage: Sensitive actions across content, taxonomy, exam configuration, parent links, and administrative authentication record structured events capturing actor identity, action key (e.g. `content.question.publish`), target resource ID, and operational metadata.

RFC 4180 Compliant CSV Export: The export endpoint streams CSV data with properly escaped fields, double quotes, and CRLF line terminators.

## Decisions made

Database-Level Trigger for Immutability: Rather than relying solely on application-level logic to prevent mutations, an engine-level trigger guarantees that audit rows cannot be updated or deleted even by direct SQL queries.

Fail-Safe Isolation for Standalone Actions: If an audit write encounters a transient database issue during a standalone request, it emits an alert log instead of failing the primary business action. For operations using an explicit database transaction, the audit write is part of the transaction to guarantee all-or-nothing atomicity.

Wildcard Domain Filtering: The query API and repository support wildcard prefix filters such as `content.*` or `exam.*`, allowing administrators to view entire subsystems in a single view.

## Notes

- No existing user-facing endpoints or database tables were modified destructively.
- Client implementation guidance is documented in [MOBILE_APP_CHANGES.md](file:///D:/Livik-projects/dev-docs/learning-app/tasks/feat-audit-trail/MOBILE_APP_CHANGES.md) and [ADMIN_PANEL_CHANGES.md](file:///D:/Livik-projects/dev-docs/learning-app/tasks/feat-audit-trail/ADMIN_PANEL_CHANGES.md).

## Test plan

Automated unit tests (209 unit tests passing):
- [x] `test/unit/audit.service.spec.ts`: verifies audit recording, fail-safe isolation, transaction re-throw, getById (404 error), CSV generation, and query schema validation.
- [x] `test/unit/security-verification.spec.ts`: verifies that all 9 security checks pass, including Check 9 for audit trail coverage.
- [x] All 21 test suites in `learning-app-backend` pass cleanly.

Manual verification:
- [x] Verified database migration SQL syntax and immutable trigger binding.
- [x] Verified `GET /admin/audit-logs` endpoint pagination, action filtering, and date range query parsing.
- [x] Verified `npm run verify:security` passes with 9/9 checks.
