# End-to-End Dependency and Execution Plan: P1-059 Audit Trail & Sensitive Action Logging

## 1. Full Dependency Chain Overview

Case P1-059 (Auditability) is the target case of this plan. In the Phase 1 master workplan (`phase1_student_exam_prep_workplan.xlsx`), it sits in the Security & Compliance workstream and depends directly on four foundational feature implementations:
- **P1-059** depends directly on:
  1. **P1-020** (Enforce role and relationship authorization) in Authentication & Accounts.
  2. **P1-031** (Implement draft, publish, deactivate lifecycle) in Question Bank & Content.
  3. **P1-033** (Build exam template and category-weightage configuration) in Mock Exams.
  4. **P1-054** (Implement parent-child invite workflow) in Parent Experience.

In addition, the official detailed description of P1-059 mandates recording actor, action, target, time, and result across five specific sensitive operational categories:
1. **Content publishing** (`P1-030`, `P1-031`, `P1-027`): Question creation, updates, state transitions (`draft` -> `published` -> `archived` -> `restored`), and bulk imports.
2. **Taxonomy changes** (`P1-010`, `P1-030`): Subject, category, and topic creation, updates, archiving, restoring, and deletion.
3. **Exam configuration** (`P1-013`, `P1-033`): Mock exam template creation, duration/scoring adjustments, section category-weightage configurations, publishing, and unpublishing.
4. **Parent links** (`P1-016`, `P1-054`): Parent invite creation, resending, code redemption, cancellation, and student link request approval, rejection, or revocation.
5. **Auth-sensitive actions** (`P1-008`, `P1-018`, `P1-019`, `P1-020`): Admin account registration (`pending`), admin approval, admin suspension, user account deletion, and forced session revocations.

All of these feature modules rest upon the Phase 1 core architecture and database baseline:
- **P1-020** depends on **P1-003** (Role and Permission Matrix), **P1-016** (Parent Linking Model), and **P1-019** (Session Revocation).
- **P1-031** depends on **P1-030** (Question CRUD), which depends on **P1-010** (Question Taxonomy Model), **P1-011** (Answers), **P1-012** (Explanations), and **P1-029** (Media).
- **P1-033** depends on **P1-013** (Exam Configuration Model) and **P1-020** (Authorization).
- **P1-054** depends on **P1-016** (Parent Linking Model), **P1-019** (Session Revocation), and **P1-020** (Authorization).
- All database entities depend on **P1-007** (PostgreSQL Schema and Migration Workflow).
- Service architecture, error envelopes, and permissions depend on **P1-001** (Phase 1 Scope), **P1-002** (Technical Architecture), and **P1-005** (Service Bootstrap).

```
                            P1-001: Phase 1 Scope
                          (Discovery & Architecture)
                                 /     |     \
                                /      |      \
                               v       v       v
      P1-002: Architecture   P1-003: Roles   P1-005: Service Bootstrap
               |                   |                  |
               v                   |                  |
      P1-007: DB Schema            |                  |
        /      |      \            |                  |
       /       |       \           |                  |
      v        v        v          v                  |
   P1-010-012  |    P1-008: Core Identity             |
 (Taxonomy &   |    (Accounts, Profiles, Roles)       |
   Content)    |        /             \               |
      |        |       /               \              |
      v        v      v                 v             |
   P1-029    P1-016: Parent        P1-017 / P1-018    |
   (Media)   (Invite & Link Model)   (OAuth & OTP)    |
      |               |               /       \       |
      v               |              /         \      |
   P1-030: Question   |             v           v     v
   CRUD Forms         |          P1-019: Sessions & Tokens
      |               |                 \       /
      v               \                  \     /
   P1-031: Publishing  \---> P1-020: Authorization Guards
      |                        /                 \
      |                       /                   \
      |                      v                     v
      |             P1-033: Exam Config     P1-054: Parent Invites
      |             (Templates & Weights)   (Invite & Link Workflow)
      |                      \                     /
      |                       \                   /
      +------------------------+-----------------+
                               |
                               v
                     P1-059: Auditability
               (Admin & Sensitive Action Audit Trail)
```

In this plan, **100% completion of any case means bringing its code and documentation to the "Ready for Review" status**, with all acceptance criteria satisfied, edge cases protected, database triggers enforcing tamper resistance, and automated test suites passing.

---

## 2. Code-Based Status Evaluation of the Entire Chain

The statuses below are evaluated strictly against what currently exists in the live repositories (`learning-app-backend`, `learning-app-admin`, `learning-web-app`, and `D:\lm`):

| Case ID | Workstream | Priority | Code-Based Status | What Exists in Code Today | What Is Missing to Reach 100% (Ready for Review) |
|---|---|---|---|---|---|
| **P1-001** | Discovery & Architecture | P0 - Critical | **Ready for Review** (100%) | Phase 1 scope, boundaries (iOS priority, web/admin readiness), and core features documented in `architecture.md`. | None. Scope boundaries are established and active. |
| **P1-002** | Discovery & Architecture | P0 - Critical | **Ready for Review** (100%) | Layered monolith, service boundaries, and URI versioning (`/api/v1/`) active in `architecture.md` and `src/shared/http/configure-http.ts`. | None. Architectural baseline is active. |
| **P1-003** | Discovery & Architecture | P0 - Critical | **Ready for Review** (100%) | Role matrices for student, parent, and admin defined in `architecture.md` (§8); enforced via `StudentRoleGuard`, `ParentRoleGuard`, and `AdminRoleGuard`. | None. Permission models are fully active. |
| **P1-005** | Backend/API Foundation | P0 - Critical | **Ready for Review** (100%) | NestJS baseline, Zod env validation, error envelope, Pino logging, Terminus health probes, Swagger OpenAPI, and passing unit/integration suites. | None. Foundational service baseline is complete. |
| **P1-007** | Database & Data Model | P0 - Critical | **Ready for Review** (100%) | PostgreSQL migrations with `node-pg-migrate`, deterministic seeding, UTC timestamps, and UUID primary keys active. Shared `prevent_direct_event_mutation()` append-only trigger function installed in migration 0018. | Migration 0019 required for the centralized `audit_events` table. |
| **P1-008** | Database & Data Model | P0 - Critical | **Ready for Review** (100%) | Tables `users`, `user_profiles`, `user_roles`, and `auth_identities` supporting student, parent, and admin accounts. | None. Identity schema is complete. |
| **P1-010** | Database & Data Model | P0 - Critical | **Ready for Review** (100%) | Normalized taxonomy tables (`subjects`, `topics`, `categories`, `questions`) supporting question classification. | None. Taxonomy model is complete. |
| **P1-011** | Database & Data Model | P0 - Critical | **Ready for Review** (100%) | Schema `question_options` and evaluation logic supporting single and multi-correct options. | None. Answer model is complete. |
| **P1-012** | Database & Data Model | P1 - High | **Ready for Review** (100%) | Question explanations table and columns supporting text explanations and media references. | None. Explanation schema is complete. |
| **P1-013** | Database & Data Model | P0 - Critical | **Ready for Review** (100%) | Exam blueprints and category weightage tables (`assessment_blueprints`, `assessment_blueprint_categories`). | None. Exam schema is complete. |
| **P1-016** | Database & Data Model | P0 - Critical | **Ready for Review** (100%) | Schema `parent_child_links`, `parent_student_invites`, and append-only `parent_link_events` table with trigger in migration 0018. `ParentLinksRepository` actively records events on invite and link transitions. | None. Linking and event schema is complete. |
| **P1-017** | Authentication & Accounts | P0 - Critical | **Ready for Review** (100%) | Google OAuth authorization code exchange, ID token validation, and account linkage in `GoogleIdentityService`. | None. Google sign-in is operational. |
| **P1-018** | Authentication & Accounts | P0 - Critical | **Ready for Review** (100%) | Email OTP generation, HMAC-SHA256 hashing, 5-minute expiry, single-use consumption, and 60-second resend cooldown in `OtpService`. Rate limits applied. | None. OTP service is complete. |
| **P1-019** | Authentication & Accounts | P0 - Critical | **Ready for Review** (100%) | Refresh token rotation, SHA-256 token hashing, session revocation, and device tracking in `TokenService` and `SessionsRepository`. | Audit logging needed when sessions are revoked or accounts deleted. |
| **P1-020** | Authentication & Accounts | P0 - Critical | **Ready for Review** (100%) | Role guards (`AdminRoleGuard`, `StudentRoleGuard`, `ParentRoleGuard`) and relationship authorization in `ParentLinksService`. | Audit query endpoints must be secured by `AdminRoleGuard`. |
| **P1-029** | Media & Assets | P0 - Critical | **Ready for Review** (100%) | Question and explanation image storage, pre-signed URL generation, and status tracking in `MediaService`. | None. Media service is operational. |
| **P1-030** | Question Bank & Content | P0 - Critical | **Ready for Review** (100%) | Admin question create, edit, archive, and restore endpoints in `QuestionsController` and `QuestionsService`. | State transitions and updates do not emit audit trail records. |
| **P1-031** | Question Bank & Content | P1 - High | **Ready for Review** (100%) | Question lifecycle (`draft`, `published`, `archived`) enforced in `QuestionsService.updateQuestionStatus`. | Content publishing, unpublishing, and archiving actions do not record actor, target, time, and result. |
| **P1-033** | Mock Exams | P1 - High | **Ready for Review** (100%) | Exam template creation, blueprint editing, category-weightage calculation, publish, and unpublish in `ExamsService`. | Blueprint edits, duration modifications, and exam publish/unpublish actions do not record audit records. |
| **P1-054** | Parent Experience | P0 - Critical | **Ready for Review** (100%) | Parent invite creation, resending, code redemption, and cancellation in `ParentInvitesService` and `ParentRequestsController`. Transitions actively write to `parent_link_events`. | Dual-recording into the centralized `audit_events` table for master reviewer auditing. |
| **P1-059** | Security & Compliance | P1 - High | **Target Case (In Planning)** | Shared append-only guard function `prevent_direct_event_mutation()` exists in PostgreSQL. Table `parent_link_events` is actively recorded. | 1. Dedicated `audit_events` table with immutable trigger not yet created.<br>2. Centralized `AuditService` not yet implemented.<br>3. Interceptors/hooks missing across content publishing, taxonomy changes, exam config, parent links, and auth-sensitive actions.<br>4. Reviewer query and export endpoint `GET /api/v1/admin/audit-logs` missing.<br>5. Admin panel Audit Trail viewer UI and CSV export missing. |

---

## 3. Product and Architectural Decisions (Resolved)

All architectural and security decisions for Case P1-059 have been established as follows:

| Decision Item | Chosen Approach | Implementation Details |
|---|---|---|
| **D1: Dedicated Audit Table vs Event Sourcing** | **Dedicated Immutable `audit_events` Table** | Implement a standalone relational audit table in PostgreSQL (`audit_events`) rather than a complex distributed event store. The table records structured audit rows with `actor_id`, `actor_role`, `actor_email`, `action`, `target_type`, `target_id`, `target_label`, `result`, `details` (JSONB), and `created_at`. |
| **D2: Immutability & Tamper Resistance** | **PostgreSQL Engine-Level Append-Only Trigger** | Attach the existing database function `prevent_direct_event_mutation()` to `audit_events` via `BEFORE UPDATE OR DELETE` trigger. Any attempt by an administrator, compromised service account, or SQL injection to alter or delete an existing audit log entry is immediately rejected at the PostgreSQL database engine level with code `P0001`. |
| **D3: Synchronous vs Asynchronous Audit Writes** | **Fail-Safe Synchronous Logging with Isolated Transaction** | Write audit records synchronously within the service execution flow using a resilient wrapper (`AuditService.record`). If the audit write fails due to unexpected database connectivity, the wrapper logs a critical security alert via Pino without terminating valid read-only requests, while ensuring that state-altering transactions (e.g. content publishing, admin approval) commit their audit event in the same atomic transaction. |
| **D4: Action Nomenclature Standard** | **Hierarchical Dot-Notation Action Keys** | Structure all audit action strings as `<domain>.<entity>.<verb>`:<br>1. Content: `content.question.create`, `content.question.update`, `content.question.publish`, `content.question.archive`, `content.question.restore`, `content.question.bulk_import`<br>2. Taxonomy: `taxonomy.node.create`, `taxonomy.node.update`, `taxonomy.node.archive`, `taxonomy.node.restore`, `taxonomy.node.delete`<br>3. Exams: `exam.config.create`, `exam.config.update`, `exam.config.publish`, `exam.config.unpublish`, `exam.config.delete`<br>4. Parent Links: `parent.invite.create`, `parent.invite.resend`, `parent.invite.redeem`, `parent.invite.cancel`, `parent.link.approve`, `parent.link.reject`, `parent.link.revoke`<br>5. Auth/Sensitive: `auth.admin.register`, `auth.admin.approve`, `auth.admin.reject`, `auth.session.logout_all`, `auth.account.delete` |
| **D5: Sensitive Data Scrubbing in Details** | **Strict PII & Secret Redaction** | The `details` JSONB column stores before/after snapshots and request context (client IP, user agent, requestId). Secrets (passwords, tokens, OTP codes) and full raw tax identification or payment details are strictly scrubbed before saving. User emails are stored bare for actor traceability, while passwords and tokens are never included. |
| **D6: Authorized Reviewer Access Model** | **Restricted Admin Query Endpoint with Multi-Filter Support** | Expose `GET /api/v1/admin/audit-logs` guarded by `AdminRoleGuard`. Support filtering by `targetType`, `action`, `actorId`, `startDate`, `endDate`, and `result` with cursor-based or limit/offset pagination (max 100 per page). Expose `GET /api/v1/admin/audit-logs/export` returning RFC 4180 compliant CSV. |
| **D7: Integration with `parent_link_events`** | **Dual Recording for Specialized Compliance** | Maintain the dedicated `parent_link_events` table (defined in migration 0018) for fine-grained parent-child relationship history, while also recording parent link operational events into the centralized `audit_events` table so the master audit trail provides a single pane of glass for compliance reviewers. |

---

## 4. Sequential Execution Pipeline

The execution of Case P1-059 is structured into four sequential phases to ensure full testability, immutability, and zero regression across existing services:

```
+---------------------------------------------------------------------------------------------------------+
|                                       SEQUENTIAL EXECUTION PIPELINE                                     |
+------------------------------------+----------------------------------+---------------------------------+
| Phase 1: Database & Audit Core     | Phase 2: Domain Integrations     | Phase 3: Reviewer API & UI      |
| (Migration, Entity, Service)       | (Content, Taxonomy, Exams, Auth) | (Endpoints, Export, Admin Panel)|
+------------------------------------+----------------------------------+---------------------------------+
| 1. Create migration 0019 with      | 1. Hook into QuestionsService    | 1. Create AuditController with  |
|    audit_events schema and trigger.|    (publish, archive, restore).  |    GET /admin/audit-logs.       |
| 2. Implement AuditModule and       | 2. Hook into TaxonomyService     | 2. Add CSV export endpoint      |
|    AuditService with fail-safe     |    (node create, update, delete).|    GET /admin/audit-logs/export.|
|    actor context extraction.       | 3. Hook into ExamsService        | 3. Build Audit Trail viewer     |
| 3. Unit test append-only trigger   |    (blueprint, publish, delete). |    page in learning-app-admin.  |
|    and audit recording methods.    | 4. Hook into ParentInvitesService| 4. Add filter dropdowns and     |
|                                    |    and ParentRequestsService.    |    metadata inspect drawer.     |
|                                    | 5. Hook into AuthService &       |                                 |
|                                    |    AuthAdminController (admins). |                                 |
| -> Target: Core Store Ready        | -> Target: 100% Operational Cov. | -> Target: Traceability Active  |
+------------------------------------+----------------------------------+---------------------------------+
                                                     |
                                                     v
                                      +-----------------------------+
                                      | Phase 4: E2E Verification   |
                                      | (Unit, Integration, Audit) |
                                      +-----------------------------+
                                      | 1. Unit tests for query DTOs|
                                      |    and CSV generator.       |
                                      | 2. Integration tests for all|
                                      |    5 sensitive domains.     |
                                      | 3. Add Check 9 to CLI:      |
                                      |    verify-security.ts.      |
                                      | -> Target: 100% Pass Rate   |
                                      +-----------------------------+
```

---

## 5. Detailed Breakdown of the Dependency Chain Cases

### 5.1 Case P1-020: Enforce Role and Relationship Authorization
- **Role in Chain**: Provides role definitions (`student`, `parent`, `admin`), role guards (`AdminRoleGuard`, `StudentRoleGuard`, `ParentRoleGuard`), and relationship authorization checks.
- **Audit Requirement**:
  - The reviewer query endpoint `GET /api/v1/admin/audit-logs` must be strictly restricted to authenticated users possessing the `admin` role. Non-admin users or unauthenticated actors attempting to read the audit log must receive HTTP 403 Forbidden or HTTP 401 Unauthorized.
  - Role-changing actions (such as promoting a user or approving a pending admin) represent critical authorization events that must be recorded in the audit trail with the authorizer's ID.
- **Status & Verification**: Active and passing. Unit and integration tests verify role enforcement.

### 5.2 Case P1-031: Implement Draft, Publish, Deactivate Lifecycle
- **Role in Chain**: Governs question status transitions (`draft`, `published`, `archived`) in `QuestionsService`.
- **Audit Requirement**:
  - Whenever an administrator publishes a draft question, archives an active question, restores an archived question, or modifies question text/options/explanations, an audit record must be recorded:
    - `action`: `content.question.publish`, `content.question.archive`, `content.question.restore`, or `content.question.update`.
    - `target_type`: `'question'`.
    - `target_id`: Question UUID.
    - `target_label`: Truncated question stem (first 60 characters).
    - `details`: Changed status, previous status, and administrator email.
- **Status & Verification**: Endpoint active in `QuestionsController` (`PATCH :id`, `POST :id/archive`, `POST :id/restore`). Hooking `AuditService` into these transitions satisfies P1-059 content publishing requirements.

### 5.3 Case P1-033: Build Exam Template and Category-Weightage Configuration
- **Role in Chain**: Manages mock exam templates, section configurations, category-weightage blueprints, and publishing in `ExamsService`.
- **Audit Requirement**:
  - Operational changes to exams alter the scoring and structure for all students taking exams. The audit trail must capture:
    - Blueprint and weightage updates: `action: 'exam.config.update'`.
    - Publishing an exam live: `action: 'exam.config.publish'`.
    - Withdrawing/unpublishing an exam: `action: 'exam.config.unpublish'`.
    - Deleting an exam: `action: 'exam.config.delete'`.
    - Target: Exam UUID, with `target_label` containing the exam title (e.g. "SAT Full Practice Test 1").
- **Status & Verification**: Controller endpoints active in `ExamsController` (`POST /admin/mock-exams`, `PATCH :examId`, `POST :examId/publish`, `POST :examId/unpublish`, `DELETE :examId`). Hooking `AuditService` into these methods provides exam configuration traceability.

### 5.4 Case P1-054: Implement Parent-Child Invite Workflow
- **Role in Chain**: Handles invitations between parents and children via 6-character codes and link requests in `ParentInvitesService` and `ParentRequestsController`.
- **Audit Requirement**:
  - Sensitive operations on relationship links grant or revoke access to student learning data. The audit trail must capture:
    - Invite creation: `action: 'parent.invite.create'`.
    - Invite redemption: `action: 'parent.invite.redeem'`.
    - Invite cancellation: `action: 'parent.invite.cancel'`.
    - Link request approval: `action: 'parent.link.approve'`.
    - Link revocation: `action: 'parent.link.revoke'`.
    - In addition, the dedicated `parent_link_events` table must record each transition for child-parent relationship integrity.
- **Status & Verification**: Controller routes active and `ParentLinksRepository` actively writes to `parent_link_events`. Dual-recording into `audit_events` fulfills the centralized auditability requirement.

---

## 6. Step-by-Step Implementation Plan for Case P1-059

### Stage 1: Database Migration (`0019_audit-events.sql`)
1. **Create Table `audit_events`**:
   - `id uuid PRIMARY KEY DEFAULT gen_random_uuid()`
   - `actor_id uuid REFERENCES users(id) ON DELETE SET NULL`
   - `actor_role text NOT NULL DEFAULT 'system'`
   - `actor_email text`
   - `action text NOT NULL`
   - `target_type text NOT NULL`
   - `target_id text NOT NULL`
   - `target_label text`
   - `result text NOT NULL DEFAULT 'success'`
   - `details jsonb NOT NULL DEFAULT '{}'::jsonb`
   - `ip_address text`
   - `user_agent text`
   - `created_at timestamptz NOT NULL DEFAULT now()`
2. **Apply Constraints & Indexes**:
   - `CONSTRAINT audit_events_result_chk CHECK (result IN ('success', 'failure'))`
   - `CONSTRAINT audit_events_action_not_blank CHECK (btrim(action) <> '')`
   - `CREATE INDEX audit_events_created_at_idx ON audit_events (created_at DESC)`
   - `CREATE INDEX audit_events_target_idx ON audit_events (target_type, target_id)`
   - `CREATE INDEX audit_events_actor_idx ON audit_events (actor_id)`
   - `CREATE INDEX audit_events_action_idx ON audit_events (action)`
3. **Apply Append-Only Protection**:
   - `CREATE TRIGGER audit_events_append_only BEFORE UPDATE OR DELETE ON audit_events FOR EACH ROW EXECUTE FUNCTION prevent_direct_event_mutation();`

### Stage 2: Audit Module & Service (`src/modules/audit/`)
1. **Create `audit.types.ts`**:
   - Define TypeScript interfaces: `AuditEvent`, `CreateAuditEventInput`, `AuditLogQueryDto`, `AuditLogSummary`.
   - Define valid `AuditAction` union type and `AuditTargetType` union type.
2. **Create `audit.service.ts`**:
   - Method `record(input: CreateAuditEventInput, client?: PoolClient): Promise<AuditEvent>`
   - Method `list(query: AuditLogQueryDto): Promise<{ items: AuditEvent[]; total: number; nextCursor?: string }>`
   - Method `getById(id: string): Promise<AuditEvent | null>`
   - Method `exportCsv(query: AuditLogQueryDto): Promise<string>`
3. **Create `audit.module.ts`**:
   - Export `AuditService` as a shared global module or inject into domain modules.

### Stage 3: Operational Integrations across Sensitive Categories
1. **Category 1: Content Publishing (`QuestionsService`)**:
   - On question create: record `content.question.create`.
   - On question update: record `content.question.update` with modified field list.
   - On question archive: record `content.question.archive`.
   - On question restore: record `content.question.restore`.
   - On bulk import: record `content.question.bulk_import` with chunk count, row count, and error summary.
2. **Category 2: Taxonomy Changes (`TaxonomyService`)**:
   - On node create (subject, topic, category): record `taxonomy.node.create`.
   - On node update: record `taxonomy.node.update`.
   - On node archive/restore: record `taxonomy.node.archive` / `taxonomy.node.restore`.
   - On node delete: record `taxonomy.node.delete`.
3. **Category 3: Exam Configuration (`ExamsService`)**:
   - On exam create: record `exam.config.create`.
   - On exam blueprint/weightage update: record `exam.config.update`.
   - On exam publish: record `exam.config.publish`.
   - On exam unpublish: record `exam.config.unpublish`.
   - On exam delete: record `exam.config.delete`.
4. **Category 4: Parent Links (`ParentInvitesService` & `ParentRequestsService`)**:
   - On invite created: record `parent.invite.create` and insert into `parent_link_events`.
   - On invite redeemed: record `parent.invite.redeem` and insert into `parent_link_events`.
   - On invite cancelled: record `parent.invite.cancel` and insert into `parent_link_events`.
   - On link approved: record `parent.link.approve` and insert into `parent_link_events`.
   - On link revoked: record `parent.link.revoke` and insert into `parent_link_events`.
5. **Category 5: Auth-Sensitive Actions (`AuthService` & `AuthAdminController`)**:
   - On admin registration (`pending`): record `auth.admin.register`.
   - On admin account approval: record `auth.admin.approve`.
   - On admin account rejection/suspension: record `auth.admin.reject`.
   - On user account deletion: record `auth.account.delete`.
   - On session logout-all / revocation: record `auth.session.logout_all`.

### Stage 4: Reviewer API Endpoints (`AuditController`)
1. Create `src/modules/audit/audit.controller.ts`:
   - `@Controller({ path: 'admin/audit-logs', version: '1' })`
   - `@UseGuards(AdminRoleGuard)`
   - `@Get()`: Paginated list with filters (`targetType`, `action`, `actorId`, `startDate`, `endDate`, `result`, `limit`, `cursor`).
   - `@Get('export')`: Stream or return RFC 4180 CSV export.
   - `@Get(':id')`: Retrieve individual event details.

### Stage 5: Admin Panel Reviewer UI (`learning-app-admin`)
1. **API Client Additions**:
   - Add audit methods in `src/features/audit/api/audit-api.ts`: `fetchAuditLogs`, `fetchAuditEventById`, `exportAuditLogsUrl`.
2. **Audit Trail Page (`src/features/audit/pages/audit-page.tsx`)**:
   - Chronological table showing: Timestamp (UTC / localized), Actor (email + role badge), Action badge (color-coded by domain: content=blue, taxonomy=cyan, exam=purple, parent=green, auth=amber), Target (type + label/ID), Result badge (success=green, failure=red).
   - Filter bar: Target domain select, action search, date picker range, actor email input.
   - Detail view modal / drawer: Formatted JSON before/after state diff, request ID, client IP, user agent.
   - Export button: Triggers CSV download.
3. **Navigation Integration**:
   - Add "Audit Trail" link to the admin panel sidebar under Security / Settings.

### Stage 6: Security Verification Script
1. Update `src/scripts/verify-security.ts`:
   - Add Check 9 (`verifyAuditTrailCoverage`):
     - Assert that table `audit_events` exists and has the append-only trigger attached.
     - Assert that `prevent_direct_event_mutation()` rejects UPDATE and direct DELETE.
     - Assert that all 5 required operational domains are wired to `AuditService`.
   - Run `npm run verify:security` and ensure all 9/9 checks pass cleanly.

---

## 7. Implementation Impact on Other Repositories

### 7.1 Student and Parent Mobile App (`D:\lm` - React Native / Expo)
- **Client Transparency**: The mobile app does not feature an administrative audit viewer, as mobile users are strictly students and parents.
- **Metadata Propagation**: Mobile API client passes consistent client headers (`x-request-id`, user agent) during parent invite creation, redemption, and link approvals so backend audit logs capture complete device context.
- **Guidance Document**: Created in `MOBILE_APP_CHANGES.md`.

### 7.2 Admin Web Portal (`learning-app-admin` - Vite + React)
- **Reviewer Capabilities**: Authorized reviewers and system administrators gain an interactive, filterable Audit Trail viewer to trace all operational changes made to questions, taxonomy, exams, parent links, and admin accounts.
- **CSV Export**: Compliance officers can export filtered audit trails directly to CSV for external audits or compliance reviews.
- **Guidance Document**: Created in `ADMIN_PANEL_CHANGES.md`.

### 7.3 Student / Parent Web App (`learning-web-app` - Next.js)
- **Account Actions**: Web app users triggering account deletion or link approvals will generate audit records on the backend. No reviewer UI changes needed.

---

## 8. Comprehensive Test Plan

### 8.1 Database Immutability Unit & Migration Tests
- [ ] Attempt `UPDATE audit_events SET action = 'tampered' WHERE id = :id` -> Throws PostgreSQL exception code `P0001` (*"audit_events is append-only: rows cannot be updated"*).
- [ ] Attempt direct `DELETE FROM audit_events WHERE id = :id` -> Throws PostgreSQL exception code `P0001` (*"audit_events is append-only: rows cannot be deleted directly"*).
- [ ] Foreign key cascade: deleting a non-critical reference or setting null succeeds without breaking log integrity.

### 8.2 Audit Service Unit Tests
- [ ] `record()` persists structured fields with valid timestamp and defaults.
- [ ] `record()` safely redacts passwords, tokens, and authorization secrets.
- [ ] `list()` filters correctly by `targetType`, `action`, `actorId`, `startDate`, `endDate`, and `result`.
- [ ] `exportCsv()` generates compliant RFC 4180 CSV with escaped commas and quotes.

### 8.3 Operational Domain Integration Tests
- [ ] Content Publishing: Publishing a question via `PATCH /admin/questions/:id` creates an audit row with `action: 'content.question.publish'`.
- [ ] Taxonomy: Archiving a category via `POST /admin/taxonomy/nodes/:id/archive` creates an audit row with `action: 'taxonomy.node.archive'`.
- [ ] Exams: Updating an exam blueprint creates an audit row with `action: 'exam.config.update'`.
- [ ] Parent Links: Redeeming an invite code creates an audit row with `action: 'parent.invite.redeem'` and a corresponding row in `parent_link_events`.
- [ ] Auth: Approving a pending admin creates an audit row with `action: 'auth.admin.approve'`.

### 8.4 Reviewer Authorization Tests
- [ ] `GET /api/v1/admin/audit-logs` by unauthenticated caller -> HTTP 401 Unauthorized.
- [ ] `GET /api/v1/admin/audit-logs` by student or parent -> HTTP 403 Forbidden.
- [ ] `GET /api/v1/admin/audit-logs` by admin -> HTTP 200 OK with paginated items.
