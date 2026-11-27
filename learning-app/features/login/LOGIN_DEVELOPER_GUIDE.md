# Login Developer Guide: Required Fields & Implementation

> **Target Audience:** Developers and AI Agents.  
> **Purpose:** Detailed reference for the Login feature across Mobile App, Backend, Web App, and Admin Panel, covering authentication methods, validation rules, code paths, APIs, and database schemas.

---

## 1. Required Fields & Originating Modules

Authentication differs by audience: students and parents use passwordless login (email OTP or Google Sign-In), while administrative staff authenticate with email and password.

### 1.1 Mobile & Web Passwordless Login (Email OTP & Google)
*Captured on the Login and Verify Email screens in Mobile and Web apps.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Email** | Yes | Login screen (`/login`) | `users.email` | Normalized to lower case. Looks up existing account. |
| **OTP Code** | Yes (for OTP) | Verify screen (`/verify-email`) | `email_verification_otps.otp_hash` | 6-digit numeric string. Valid for 5 minutes. |
| **Google ID Token** | Yes (for Google) | Google button | Stored in provider | Verified via Google public keys (`POST /auth/google`). |
| **Refresh Token** | Auto-issued | Background service | `auth_refresh_tokens.token_hash` | Opaque string. Rotated on each refresh request. |

---

### 1.2 Admin Panel Staff Login
*Captured on the Admin Portal login page.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Email** | Yes | Admin Login form | `users.email` | Normalized to lower case. Scoped to admin audience. |
| **Password** | Yes | Admin Login form | `users.password_hash` | Minimum 8 characters. Verified using Argon2id. |
| **Account Status** | Auto-checked | Server check | `users.status` | Must be `'active'`. `'pending'` and `'disabled'` are rejected. |

---

## 2. Eligibility & Validation Rules Checklist

Before issuing a session, the system verifies:
1. **Registered account existence:** The email must belong to an existing user in the requested audience (`users_mobile_email_key` for students/parents, `users_admin_email_key` for admins). Unknown emails return 404 on mobile OTP or 401 on admin login.
2. **Account status:** The account status must be `'active'`. Pending admin accounts return 403 (`ACCOUNT_PENDING_APPROVAL`). Inactive or disabled accounts return 403 (`ACCOUNT_INACTIVE`).
3. **OTP validity and lifetime:** The OTP code must match the SHA-256 hash stored in `email_verification_otps` and must be consumed within 5 minutes (`ttlMs`).
4. **Resend rate limit cooldown:** A new OTP can only be requested once every 60 seconds per account.
5. **Brute-force protection:** After 5 incorrect OTP guesses (`maxAttempts`), the OTP code is invalidated and rejected with `OTP_ATTEMPTS_EXCEEDED`.
6. **Refresh token rotation:** Each refresh token is single-use. Replaying an already-consumed refresh token triggers reuse detection and immediately revokes the entire user session.
7. **Single-flight concurrency lock:** Both frontend clients and backend service serialize refresh requests using in-flight locks to prevent race conditions during simultaneous API calls.

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
│    POST /api/v1/auth/request-otp (Step 1: Send OTP)    │
│    POST /api/v1/auth/verify-email (Step 2: Sign In)    │
│    POST /api/v1/auth/google (Google OAuth)             │
│    POST /api/v1/admin/auth/login (Admin Password)      │
│    POST /api/v1/auth/refresh (Token Rotation)          │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│           PostgreSQL Database (Neon DB)                │
│   tables: users, email_verification_otps,              │
│           auth_sessions, auth_refresh_tokens           │
└────────────────────────────────────────────────────────┘
```

### 3.1 Mobile App (`src/`)
Provides passwordless authentication using a 6-digit email OTP and Google Sign-In, storing access and refresh tokens securely on the device. The user enters their email on `login.tsx` to request an OTP code, which is then submitted on `verify-email.tsx`. Upon verification, tokens are stored in `SecureStore` via `token-storage.ts`, and `useAfterAuth` checks `user.onboardingCompleted` to route unfinished students to `/resume` or completed students to `/hub`. Background API calls automatically rotate expired access tokens via `api-client.ts` with a single-flight mutex.

* **Code References:**
  * Login screen: `src/app/(auth)/login.tsx` (lines 28–46)
  * OTP verify screen: `src/app/(auth)/verify-email.tsx` (lines 34–54)
  * Google sign-in hook: `src/hooks/use-google-sign-in.ts` (lines 10–40)
  * SecureStore token management: `src/services/token-storage.ts` (lines 15–38)
  * Post-auth routing hook: `src/hooks/use-after-auth.ts` (lines 13–22)
  * API client with refresh interceptor: `src/services/api-client.ts` (lines 105–140)
* **API:** `POST /api/v1/auth/request-otp`, `POST /api/v1/auth/verify-email`, `POST /api/v1/auth/google`, `POST /api/v1/auth/refresh`
* **DB Table:** `users`, `email_verification_otps`, `auth_sessions`, `auth_refresh_tokens`

---

### 3.2 Backend (`learning-app-backend`)
Provides a dual-audience authentication controller supporting passwordless OTP, Google OAuth2 tokens, admin password login, and rotating refresh tokens. `AuthController` processes student and parent requests: `requestCode` generates a cryptographically random 6-digit code with a 5-minute expiry stored in `email_verification_otps` and sends it via email. `verifyEmail` checks the code inside a database row transaction (`lockById`), marks the user verified, and issues a JWT token pair. `AuthAdminController.login` validates admin credentials using Argon2id and rejects non-active accounts. Token rotation consumes the presented refresh token and replaces it atomically.

* **Code References:**
  * Mobile auth controller: `src/modules/auth/auth.controller.ts` (lines 96–148, 230–283)
  * Admin auth controller: `src/modules/auth/auth-admin.controller.ts` (lines 80–135)
  * OTP request service: `src/modules/auth/auth.service.ts` (lines 222–285)
  * OTP verify service: `src/modules/auth/auth.service.ts` (lines 293–348)
  * Admin password service: `src/modules/auth/auth.service.ts` (lines 431–480)
  * Refresh rotation service: `src/modules/auth/auth.service.ts` (lines 496–565)
  * OTP database repository: `src/modules/auth/repositories/email-verification-otps.repository.ts` (lines 20–85)
  * Session and refresh repositories: `src/modules/auth/repositories/auth-sessions.repository.ts` (lines 15–60), `src/modules/auth/repositories/auth-refresh-tokens.repository.ts` (lines 20–70)
* **API:** `POST /api/v1/auth/request-otp`, `POST /api/v1/auth/verify-email`, `POST /api/v1/auth/google`, `POST /api/v1/admin/auth/login`, `POST /api/v1/auth/refresh`, `POST /api/v1/admin/auth/refresh`
* **DB Table:** `users`, `email_verification_otps`, `auth_sessions`, `auth_refresh_tokens`

---

### 3.3 Web App (`learning-web-app`)
Implements a web login interface using OTP verification and Google login with secure httpOnly cookie session management to protect tokens against client-side script access. Users input their email on `/login` and submit the OTP on `/verify-email`. The internal BFF proxy route `/api/[...path]/route.ts` intercepts token responses, writes secure `prepify_at` and `prepify_rt` cookies, and returns only user profile data to the browser, while automatically handling token refresh on 401 errors.

* **Code References:**
  * Login form component: `src/components/auth/login-form.tsx` (lines 23–45)
  * OTP verification form: `src/components/auth/verify-email-form.tsx` (lines 39–57)
  * BFF cookie proxy handler: `src/app/api/[...path]/route.ts` (lines 84–121)
  * Server session validation: `src/lib/server/session.ts` (lines 48–78)
  * Upstream fetch proxy: `src/proxy.ts` (lines 10–17)
* **API:** `/api/auth/request-otp`, `/api/auth/verify-email`, `/api/auth/google`, `/api/auth/logout`
* **DB Table:** `users`, `email_verification_otps`, `auth_sessions`, `auth_refresh_tokens`

---

### 3.4 Admin Panel (`learning-app-admin`)
Provides an administrative login portal requiring email and password credentials with automatic session refresh and role authorization. The operator enters their credentials on `login-page.tsx`, submitting to `POST /api/v1/admin/auth/login`. On success, tokens are held in session storage and user details are retrieved via `GET /admin/auth/me`. The API client wraps requests in an automatic refresh handler that deduplicates concurrent 401 refresh calls through a shared promise lock.

* **Code References:**
  * Form component: `src/features/auth/components/login-form.tsx` (lines 18–62)
  * Page shell: `src/features/auth/pages/login-page.tsx` (lines 10–34)
  * Session token manager: `src/lib/auth/session.ts` (lines 18–44)
  * Refresh lock in API client: `src/lib/api/client.ts` (lines 111–142)
  * Auth API endpoints: `src/features/auth/api/auth.api.ts` (lines 32–44)
* **API:** `POST /api/v1/admin/auth/login`, `POST /api/v1/admin/auth/refresh`, `POST /api/v1/admin/auth/logout`, `GET /admin/auth/me`
* **DB Table:** `users`, `auth_sessions`, `auth_refresh_tokens`

---

## 4. What Already Works (Zero Changes Needed)

1. **Hardware-backed mobile token storage:** Tokens in the mobile app are saved directly into iOS Keychain and Android KeyStore via `expo-secure-store`.
2. **Tamper-proof web sessions:** Next.js BFF proxy stores tokens in encrypted, httpOnly, same-site cookies, preventing script access.
3. **Single-use refresh token safety:** Reusing an old refresh token immediately invalidates all active sessions for that user.
4. **Rate limits and brute-force defenses:** 60-second cooldown timer on OTP requests and a strict 5-attempt limit on OTP code submissions.
5. **Audience separation:** Admin credentials and student credentials cannot authenticate against the opposing portal.
