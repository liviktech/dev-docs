# Mobile App Guidance: P1-059 Auditability & Sensitive Actions

## 1. Overview

Case P1-059 establishes an immutable, database-enforced audit trail for administrative and sensitive operations. While students and parents using the mobile application (`D:\lm` - React Native / Expo) do not have access to administrative audit dashboards, several user actions initiated from the mobile application represent sensitive operations that are recorded in the system audit trail.

This document details:
1. Which mobile user actions trigger backend audit log entries.
2. Client request header requirements to ensure audit fidelity.
3. User privacy, transparency, and data export compliance.

---

## 2. Sensitive Mobile Actions Captured in the Audit Trail

The following user interactions initiated in the mobile app are classified as sensitive actions and generate immutable audit events on the backend:

| User Flow | Mobile Endpoint Trigger | Recorded Action Key | Captured Target & Context |
|---|---|---|---|
| **Parent Invites Student** | `POST /api/v1/parent/invites` | `parent.invite.create` | Actor: Parent ID; Target: Invite code hash; Details: student email/phone if provided. |
| **Student Invites Parent** | `POST /api/v1/student/parent-invites` | `parent.invite.create` | Actor: Student ID; Target: Invite code hash. |
| **Redeem Invite Code** | `POST /api/v1/parent/invites/redeem` or `POST /api/v1/student/parent-invites/redeem` | `parent.invite.redeem` | Actor: Redeeming user; Target: Invite ID and established Link ID. |
| **Cancel Active Invite** | `DELETE /api/v1/parent/invites/:id` or `DELETE /api/v1/student/parent-invites/:id` | `parent.invite.cancel` | Actor: Creator user; Target: Cancelled Invite ID. |
| **Approve Link Request** | `POST /api/v1/student/parent-requests/:linkId/approve` | `parent.link.approve` | Actor: Student ID; Target: Link ID; establishes parent access. |
| **Reject Link Request** | `POST /api/v1/student/parent-requests/:linkId/reject` | `parent.link.reject` | Actor: Student ID; Target: Link ID. |
| **Revoke Existing Link** | `POST /api/v1/student/parent-requests/:linkId/revoke` | `parent.link.revoke` | Actor: Student or Parent; Target: Severed Link ID. |
| **Account Deletion** | `DELETE /api/v1/auth/account` | `auth.account.delete` | Actor: User ID; Target: User account (soft-deleted / purged). |
| **Sign Out All Devices** | `POST /api/v1/auth/logout` (with revokeAll flag) | `auth.session.logout_all` | Actor: User ID; Target: Revoked session tokens. |

---

## 3. Client Header & Correlation Requirements

To ensure that backend audit events record high-fidelity forensics for compliance and dispute resolution, the mobile HTTP client in `D:\lm` should ensure the following headers accompany every request:

1. **`x-request-id`**:
   - Format: UUID v4 (`req-xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx`).
   - Purpose: Correlates client network attempts with backend audit records and server logs. If a user reports an issue with an invite redemption or link approval, the request ID allows immediate lookup in the audit trail.
2. **`User-Agent`**:
   - Format: `LearningAppMobile/<version> (<platform>; <os_version>; <device_model>)` (e.g. `LearningAppMobile/1.0.0 (iOS; 17.4; iPhone15,2)`).
   - Purpose: Enables compliance reviewers to verify device context during sensitive operations.
3. **`x-client-platform`**:
   - Values: `ios` or `android`.

---

## 4. Privacy & Data Handling Considerations

1. **Zero Secret Leakage**:
   - The backend audit trail never stores plaintext passwords, raw refresh tokens, or one-time verification passcodes. The mobile app can safely invoke authentication endpoints knowing secrets are scrubbed prior to audit persistence.
2. **Account Deletion Retention**:
   - When a student or parent triggers account deletion (`DELETE /api/v1/auth/account`), their profile and personal identifiable information are deleted per GDPR/COPPA guidelines. However, historical audit records retain an anonymized actor ID (`actor_id` set to null on cascade or pseudonymized) with action timestamps to maintain compliance integrity without violating user erasure rights.
3. **Parental Consent Traceability**:
   - Because parent linking grants access to minor student test results and progress analytics, both the student approval action (`parent.link.approve`) and student revocation action (`parent.link.revoke`) are permanently recorded in the audit trail with UTC timestamps to provide indisputable records of consent.
