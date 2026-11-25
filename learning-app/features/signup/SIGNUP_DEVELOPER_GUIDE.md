# Sign Up Developer Guide: Required Fields & Implementation

> **Target Audience:** Developers and AI Agents.  
> **Purpose:** Detailed reference for the Sign Up feature across Mobile App, Backend, Web App, and Admin Panel, covering required fields, validation rules, code paths, APIs, and database schemas.

---

## 1. Required Fields & Originating Modules

Sign Up collects registration details depending on whether the user is a student, parent, or administrator.

### 1.1 Mobile & Web Student/Parent Registration
*Captured in multi-step onboarding forms in the Mobile and Web apps.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Role** | Yes | Step 1 (`/signup`) | `users.role` | Either `'student'` or `'parent'`. |
| **Full Name** | Yes | Step 2 (`/signup/name`) | `users.full_name` | Stored as text. Minimum 1 character. |
| **Target Test** | Yes | Step 3 (`/signup/test`) | `users.target_test`, `user_exam_preferences` | `'sat'`, `'act'`, or `'both'`. (`'other'` is hidden in UI). |
| **Test Experience** | Yes | Step 4 (`/signup/experience`) | `users.test_experience` | Either `'first_time'` or `'familiar'`. |
| **Email** | Yes | Step 5 (`/signup/contact`) | `users.email` | Normalized to lower case. Verified via OTP on login. |
| **Phone** | Yes | Step 5 (`/signup/contact`) | `users.phone` | Stripped of spaces, dashes, brackets (7 to 15 digits). |

---

### 1.2 Admin Panel Registration
*Captured on the Admin Portal signup page.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Full Name** | Yes | Admin Signup form | `users.full_name` | 1 to 100 characters. |
| **Email** | Yes | Admin Signup form | `users.email` | Normalized to lower case. Unique within admin audience. |
| **Phone** | Optional | Admin Signup form | `users.phone` | Optional mobile contact number. |
| **Password** | Yes | Admin Signup form | `users.password_hash` | Minimum 8 characters. Hashed with Argon2id. |
| **Status** | Auto-set | Server default | `users.status` | Set to `'pending'` until approved by existing admin. |

---

## 2. Eligibility & Validation Rules Checklist

Before creating any account, the system verifies:
1. **Email format:** Must match standard email pattern (`EMAIL_RE`).
2. **Phone format:** Must contain between 7 and 15 digits (`PHONE_RE`).
3. **Audience-based uniqueness:** Due to Migration 0007, an email must be unique within its audience:
   * `users_mobile_email_key`: Unique across `role IN ('student', 'parent')`.
   * `users_admin_email_key`: Unique across `role = 'admin'`.
   * A person can have one mobile account and one admin account with the exact same email without collision.
4. **Password strength:** Admin passwords must be at least 8 characters long and match confirmation.
5. **No passwords for mobile/web signups:** Mobile and Web student/parent accounts are passwordless; login is handled via email OTP or Google.

---

## 3. Four Sections Implementation

```text
┌────────────────────────────────────────────────────────┐
│           Mobile App / Web App / Admin Panel           │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│               NestJS Backend Controller                │
│    POST /api/v1/auth/register (Student/Parent)         │
│    POST /api/v1/admin/auth/register (Admin Staff)      │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│           PostgreSQL Database (Neon DB)                │
│   tables: users, user_exam_preferences                 │
│   indexes: users_mobile_email_key, users_admin_...    │
└────────────────────────────────────────────────────────┘
```

### 3.1 Mobile App (`src/`)
Guides students or parents through a 5-step registration sequence to collect test preferences and contact details for passwordless account creation. Form responses are captured in `SignupDraftContext`, normalized, and submitted to `POST /api/v1/auth/register`. On successful registration, the user is redirected to `/login?registered=1`.

* **Code References:**
  * Draft state context: `src/state/signup-draft.tsx` (lines 19–55)
  * Role selection screen: `src/app/(auth)/signup.tsx` (lines 25–60)
  * Name input screen: `src/app/(auth)/signup-name.tsx` (lines 26–65)
  * Test selection screen: `src/app/(auth)/signup-test.tsx` (lines 26–56)
  * Experience selection screen: `src/app/(auth)/signup-experience.tsx` (lines 12–55)
  * Contact and submit screen: `src/app/(auth)/signup-contact.tsx` (lines 29–47)
  * API caller: `src/services/email-auth-api.ts` (lines 59–67)
* **API:** `POST /api/v1/auth/register`
* **DB Table:** `users`, `user_exam_preferences`

---

### 3.2 Backend (`learning-app-backend`)
Provides registration endpoints that validate incoming payloads with Zod, prevent duplicate signups per audience, and isolate admin privileges. `AuthController.register` delegates to `AuthService.register`, which creates the user and writes exam preferences atomically inside a database transaction. For staff, `AuthAdminController.register` delegates to `AuthService.adminRegister`, which hashes the password with Argon2id and sets the account status to `pending`.

* **Code References:**
  * Mobile register endpoint: `src/modules/auth/auth.controller.ts` (lines 68–94)
  * Admin register endpoint: `src/modules/auth/auth-admin.controller.ts` (lines 57–78)
  * Mobile service method: `src/modules/auth/auth.service.ts` (lines 175–207)
  * Admin service method: `src/modules/auth/auth.service.ts` (lines 396–416)
  * User repository insert: `src/modules/auth/repositories/users.repository.ts` (lines 45–87)
* **API:** `POST /api/v1/auth/register`, `POST /api/v1/admin/auth/register`
* **DB Table:** `users`, `user_exam_preferences`

---

### 3.3 Web App (`learning-web-app`)
Implements a responsive Next.js App Router multi-step signup matching mobile styling and business rules. Registration responses are tracked across steps in a React client context and submitted through the internal BFF proxy route `/api/auth/register`, which forwards the request server-to-server to the NestJS backend.

* **Code References:**
  * Draft context: `src/lib/auth/signup-draft.tsx` (lines 18–54)
  * Step page sequence: `src/app/(auth)/signup/page.tsx`, `name/page.tsx`, `test/page.tsx`, `experience/page.tsx`
  * Contact submit page: `src/app/(auth)/signup/contact/page.tsx` (lines 26–56)
  * Auth client functions: `src/lib/auth/api.ts` (lines 14–22)
  * BFF proxy handler: `src/app/api/[...path]/route.ts` (lines 68–94)
* **API:** `/api/auth/register` (proxies to NestJS `POST /api/v1/auth/register`)
* **DB Table:** `users`, `user_exam_preferences`

---

### 3.4 Admin Panel (`learning-app-admin`)
Provides an administrative staff signup form capturing name, email, phone, and password. The form validates fields with Zod and submits them to `/admin/auth/register`. Upon submission, a "Waiting for approval" card is displayed, and the account remains locked until approved by an existing administrator.

* **Code References:**
  * Form component: `src/features/auth/components/signup-form.tsx` (lines 29–64)
  * Page shell: `src/features/auth/pages/signup-page.tsx` (lines 7–35)
  * API request method: `src/features/auth/api/auth.api.ts` (lines 14–30)
* **API:** `POST /api/v1/admin/auth/register`
* **DB Table:** `users` (status set to `'pending'`, role `'admin'`)

---

## 4. What Already Works (Zero Changes Needed)

1. **Audience-level isolation:** Migration 0007 ensures student and admin registrations using the same email never collide.
2. **Atomic exam preference storage:** Mobile and web signups write test preferences in the same transaction as user creation.
3. **Admin approval requirement:** Admin signups are automatically quarantined in `pending` status.
4. **Input normalization:** Leading/trailing spaces and upper-case email characters are stripped automatically across all frontends.
