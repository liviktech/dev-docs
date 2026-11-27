# Admin Panel Guidance: P1-059 Auditability & Sensitive Actions

## 1. Overview

Case P1-059 implements an administrative audit trail to satisfy compliance, security monitoring, and operational traceability. In `learning-app-admin`, authorized reviewers and system administrators must be able to trace all relevant operational changes across five core domains: content publishing, taxonomy modifications, exam configuration, parent links, and administrative account changes.

This document outlines the technical specification for the admin panel frontend changes:
1. API client additions for querying and exporting audit logs.
2. The Audit Trail viewer page (`AuditPage`) with multi-attribute filtering and chronological timeline.
3. Event detail drawer with before/after state diffs.
4. Navigation integration into the admin shell.

---

## 2. API Client Additions (`src/features/audit/api/`)

A new API module `src/features/audit/api/audit-api.ts` connects the admin panel to the backend reviewer endpoints:

```typescript
export interface AuditEventDto {
  id: string;
  actorId: string | null;
  actorRole: string;
  actorEmail: string | null;
  action: string;
  targetType: string;
  targetId: string;
  targetLabel: string | null;
  result: 'success' | 'failure';
  details: Record<string, unknown>;
  ipAddress: string | null;
  userAgent: string | null;
  createdAt: string;
}

export interface AuditLogQuery {
  targetType?: string;
  action?: string;
  actorId?: string;
  result?: 'success' | 'failure';
  startDate?: string;
  endDate?: string;
  page?: number;
  limit?: number;
}
```

Key client methods:
- `getAuditLogs(query: AuditLogQuery): Promise<{ items: AuditEventDto[]; total: number; page: number; totalPages: number }>`
- `getAuditEventById(id: string): Promise<AuditEventDto>`
- `getAuditExportUrl(query: AuditLogQuery): string` (direct link for downloading RFC 4180 CSV)

---

## 3. UI Component Architecture

### 3.1 Audit Page (`src/features/audit/pages/audit-page.tsx`)
The main view renders:
- **Header**: Title ("Audit Trail"), live status indicator, and an "Export to CSV" button.
- **Filter Bar**:
  - Domain selector: All, Content (`content.*`), Taxonomy (`taxonomy.*`), Exams (`exam.*`), Parent Links (`parent.*`), Auth/Admin (`auth.*`).
  - Result filter: All, Success, Failure.
  - Date Range pickers: Start date and end date.
  - Search input: Filter by action name, actor email, or target ID.
- **Data Table**:
  - **Timestamp**: Formatted in both relative time (e.g. "5 minutes ago") and ISO UTC tooltip.
  - **Actor**: Avatar/icon, email address, and role badge (`admin`, `system`, `parent`, `student`).
  - **Action**: Semantic badge color-coded by category:
    - Content Publishing: Blue badge.
    - Taxonomy: Cyan badge.
    - Exam Configuration: Purple badge.
    - Parent Links: Green badge.
    - Auth & Administrative: Amber badge.
  - **Target**: Type label and human-readable identifier (e.g. `Question: "What is the slope..."`).
  - **Result**: Green pill for `success`, red pill for `failure`.
  - **Action Column**: "View Details" button to open the event drawer.
- **Pagination**: Standard limit/offset page controls (25 or 50 records per page).

### 3.2 Event Detail Drawer / Dialog
When clicking "View Details" on an audit event, a slide-out drawer or modal opens displaying:
1. Full event metadata: Event ID, exact timestamp, request ID, client IP, user agent.
2. Actor snapshot: ID, email, and role at the time of the event.
3. Target reference: Type, primary key, and human-readable label.
4. Changed state payload: Formatted JSON tree showing before/after diffs or operation context.

---

## 4. Navigation & Role Guard

- **Sidebar Entry**: An "Audit Trail" item added to the sidebar under Security / Settings (`/audit`).
- **Role Requirement**: Only authenticated administrators with the `admin` role can access this page. Non-admin users are redirected to the login screen.
