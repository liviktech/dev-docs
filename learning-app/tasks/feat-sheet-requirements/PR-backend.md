# 2026-11-23 - sheet-requirements (backend)

## Summary

This PR implements the backend foundations required by the product feature sheet (points P1-006, P1-008, P1-010, P1-011, P1-012, P1-013, P1-014, P1-015, and P1-016). It introduces database migration 0018, adds multi-select questions with all-or-nothing scoring, rich explanation support, question cooldown filtering with never-starve fallbacks, dynamic exam blueprints with preview validation, and a complete code-based parent-child linking system with link limit enforcement and subject performance consolidation.

Checked live: all 192 Vitest unit tests pass and the NestJS build compiles with zero errors. All changed schemas have matching Zod response schemas and OpenAPI documentation.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/0018_feature-sheet-foundations.sql` (new) | Migration adding question format types, explanation columns, exam section blueprint tables, parent invite code tracking table, and performance indexes for question cooldowns. |
| `src/common/idempotency/` (new) | Idempotency key extraction and header parsing to prevent duplicate mutations on invite creation and code redemption. |
| `src/common/pagination/` (new), `src/common/schemas/pagination.schemas.ts` | Added keyset cursor encoding and decoding utilities alongside existing offset pagination schemas. |
| `src/modules/cooldown/` (new) | Core cooldown logic service calculating question eligibility based on recent practice attempts and exam submissions. |
| `src/modules/exams/blueprint.logic.ts` (new) | Deterministic and randomized question selection algorithms based on subject, topic, and difficulty distribution rules. |
| `src/modules/exams/exams.controller.ts`, `exams.service.ts`, `repositories/exams.repository.ts` | Added endpoints and service logic for exam blueprints, rule validation, and blueprint preview generation. |
| `src/modules/exams/schemas/exam-request.schemas.ts`, `schemas/exam-response.schemas.ts` | Updated schemas for section blueprints, rule specifications, and preview response payloads. |
| `src/modules/questions/questions.controller.ts`, `questions.service.ts`, `repositories/questions.repository.ts` | Added multi-select question handling, explanation updates, and validation ensuring questions have at least one valid answer option. |
| `src/modules/questions/bulk-questions.service.ts` | Updated CSV/JSON bulk import parser to support multi-select format flags and explanations. |
| `src/modules/practice/practice.service.ts`, `repositories/practice.repository.ts` | Integrated cooldown filtering into question selection and implemented all-or-nothing scoring for multi-select questions. |
| `src/modules/mock-exams/mock-exams.service.ts`, `repositories/mock-exams.repository.ts` | Connected exam assembly to blueprint logic and section rule definitions with cooldown consideration. |
| `src/modules/parent/parent-invites.controller.ts` (new), `student-invites.controller.ts` (new) | Added controllers for parent-generated and student-generated invite codes (`/parent/invites` and `/student/parent-invites`). |
| `src/modules/parent/parent-invites.service.ts` (new), `repositories/parent-invites.repository.ts` (new) | Implemented secure code generation (12-char nanoid), 7-day expiration, one-time redemption, link limit enforcement, and self-linking guards. |
| `src/modules/parent/parent-links.service.ts`, `repositories/parent-links.repository.ts` | Added revocation endpoint logic (`/student/parent-requests/:id/revoke`), pending/approved count verification, and cooldown checks. |
| `src/modules/parent/parent-dashboard.logic.ts`, `parent-dashboard.service.ts` | Added `consolidateSubjects` to merge multiple taxonomy tracks with the same name into a unified parent performance bar. |
| `src/shared/assessment/assessment-activity.repository.ts` (new), `cooldown.sql.ts` (new) | Shared SQL fragments and query helpers for assessment tracking and cooldown window calculations. |
| `src/shared/constants/http.constants.ts` | Added error code constants for invite expiration, invite invalid, link limits, and cooldown conflicts. |
| `test/unit/feature-sheet-logic.spec.ts` (new), `test/integration/feature-sheet.int-spec.ts` (new) | Comprehensive unit and integration test suites covering blueprints, cooldown queries, and invite lifecycles. |
| `test/unit/parent-dashboard.spec.ts` | Added test case validating `consolidateSubjects` combines attempted questions and averages accuracy correctly. |
| `architecture.md` | Updated architecture documentation to reflect invite flows, cooldown architecture, and blueprint schema. |

## Why

The learning platform required structured data models and logic to support:
1. Flexible question formats beyond single-choice (P1-010, P1-011) with clear student explanations (P1-012).
2. Preventing students from repeatedly seeing the same questions too quickly through automated cooldown windows (P1-015).
3. Authoring standardized test blueprints that dynamically sample questions matching SAT/ACT specifications (P1-013).
4. Enabling bilateral code-based account linking between parents and students without requiring email discovery alone (P1-016).

## How

**Cooldown evaluation**: Queries filter out questions answered by the student within the configured cooldown interval. If the candidate pool falls below the requested section size, a deterministic fallback relaxes the window to ensure practice sessions never starve.

**Multi-select scoring**: Evaluated as strict equality between the sorted set of student-selected options and the sorted set of correct options. Partial credit is explicitly omitted per product decision D1.

**Dynamic exam blueprints**: Blueprint rules define target counts per subject, topic, and difficulty. The exam generator evaluates rules against active published questions and validates feasibility before publishing.

**Parent invite codes**: Codes are 12 characters, URL-safe, expire in 7 days, and are single-use. Redemption runs inside a database transaction that checks parent capacity (maximum 10 children) and student capacity (maximum 2 parents) before transitioning the link to approved.

**Consolidating subject analytics**: When a student practices topics under different taxonomy versions or exams sharing the same name (such as Mathematics), `consolidateSubjects` aggregates their totals into a single dashboard item, eliminating duplicate names and duplicate UI keys.

## Decisions made

**All-or-nothing multi-select scoring.** We opted for strict set equality rather than partial credit to align with standard standardized testing methodologies.

**Code redemption grants immediate approval.** When a student enters a code generated by a parent, or a parent enters a code generated by a student, the mutual exchange of the secret code represents mutual consent; no further approval step is required.

**Subject consolidation by display name.** Merged duplicate subject rows at the parent presentation boundary rather than modifying the underlying taxonomy tables, preserving granular historical tracking while providing clean UI rollups.

## Test plan

**Automated tests (all 192 passing):**
- [x] `test/unit/parent-dashboard.spec.ts`: verifies `consolidateSubjects` merges duplicate subject records and computes combined accuracy.
- [x] `test/unit/feature-sheet-logic.spec.ts`: verifies cooldown interval calculations, blueprint rule satisfaction, and invite expiration checks.
- [x] `test/integration/feature-sheet.int-spec.ts`: verifies database constraints, invite creation, and redemption transactions.
- [x] `nest build`: builds with 0 errors and validates all TypeScript typings.

**Manual checks (local):**
- [x] Verified code generation and redemption end to end between parent and student accounts.
- [x] Verified duplicate code entry returns 409 CONFLICT with friendly error message.
- [x] Verified parent dashboard accurately displays merged subject bars without duplicate keys.
