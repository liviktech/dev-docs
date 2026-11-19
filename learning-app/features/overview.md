# Features Overview

A concise summary of implemented features across all four project parts: Mobile App, Backend, Web App, and Admin Panel.

> **How to maintain:** When a new feature is added, insert a single row into each of the four tables below, and add the full deep-dive explanation into [feature-details.md](feature-details.md).

---

## 1. Mobile App

| Feature | Explanation | API Endpoints | Database Tables |
|---|---|---|---|
| **Sign Up** | Guides students or parents through test choice, experience, name, and contact details. Stores input in a local draft context and submits to create a passwordless mobile account. | `POST /api/v1/auth/register` | `users`<br>`user_exam_preferences` |
| **Login** | Authenticates students and parents using 6-digit email OTP codes or Google Sign-In. Saves JWT tokens in SecureStore and routes incomplete students directly to resume onboarding. | `POST /api/v1/auth/request-otp`<br>`POST /api/v1/auth/verify-email`<br>`POST /api/v1/auth/google`<br>`POST /api/v1/auth/refresh` | `users`<br>`email_verification_otps`<br>`auth_sessions`<br>`auth_refresh_tokens` |
| **Onboarding** | A 6-step questionnaire for subjects, academics, goals, skill ratings, study habits, and learning goals. Checks server state to resume at the first incomplete step and saves answers to backend step endpoints. | `GET /api/v1/student/onboarding`<br>`PUT /api/v1/student/onboarding/*`<br>`POST /api/v1/student/onboarding/complete` | `student_profiles`<br>`student_subjects`<br>`student_exam_goals`<br>`student_learning_goals`<br>`taxonomy_nodes`<br>`exams`<br>`users` |

---

## 2. Backend

| Feature | Explanation | API Endpoints | Database Tables |
|---|---|---|---|
| **Sign Up** | Validates registration data with Zod schemas and prevents duplicate emails per audience with partial indexes. Stores exam preferences for mobile users and creates staff accounts in pending status. | `POST /api/v1/auth/register`<br>`POST /api/v1/admin/auth/register` | `users`<br>`user_exam_preferences` |
| **Login** | Verifies OTP codes, Google identity tokens, or Argon2id passwords with single-use refresh token rotation. Isolates mobile and admin credentials, and blocks pending or disabled accounts. | `POST /api/v1/auth/request-otp`<br>`POST /api/v1/auth/verify-email`<br>`POST /api/v1/auth/google`<br>`POST /api/v1/admin/auth/login`<br>`POST /api/v1/auth/refresh`<br>`POST /api/v1/admin/auth/refresh` | `users`<br>`email_verification_otps`<br>`auth_sessions`<br>`auth_refresh_tokens` |
| **Onboarding** | Enforces ordered step progression by locking profile rows in transactions and rejecting skipped steps. Saves answers into dedicated relational tables and records the completion timestamp on the user account. | `GET /api/v1/student/onboarding`<br>`PUT /api/v1/student/onboarding/subjects`<br>`PUT /api/v1/student/onboarding/academic`<br>`PUT /api/v1/student/onboarding/goals`<br>`PUT /api/v1/student/onboarding/skill-levels`<br>`PUT /api/v1/student/onboarding/study-preferences`<br>`POST /api/v1/student/onboarding/complete` | `student_profiles`<br>`student_subjects`<br>`student_exam_goals`<br>`student_learning_goals`<br>`taxonomy_nodes`<br>`exams`<br>`users` |

---

## 3. Web App

| Feature | Explanation | API Endpoints | Database Tables |
|---|---|---|---|
| **Sign Up** | Multi-step browser registration that matches the mobile app flow for students and parents. Holds step answers in a client draft context and sends to the backend through the web app API handler. | `/api/auth/register`<br>*(proxies to NestJS `POST /api/v1/auth/register`)* | `users`<br>`user_exam_preferences` |
| **Login** | Email OTP and Google login using a Backend-for-Frontend (BFF) proxy route. Transferred session tokens are placed in httpOnly cookies, preventing browser JavaScript from seeing raw tokens. | `/api/auth/request-otp`<br>`/api/auth/verify-email`<br>`/api/auth/google`<br>`/api/auth/logout`<br>*(proxied to NestJS `/api/v1/auth/*`)* | `users`<br>`email_verification_otps`<br>`auth_sessions`<br>`auth_refresh_tokens` |
| **Onboarding** | Desktop web 6-step wizard with resume detection, validation on each step, and route guards. Saves student selections via TanStack Query mutations to onboarding proxy routes. | `/api/student/onboarding/*`<br>`/api/student/onboarding/complete`<br>*(proxied to NestJS `/api/v1/student/onboarding/*`)* | `student_profiles`<br>`student_subjects`<br>`student_exam_goals`<br>`student_learning_goals`<br>`taxonomy_nodes`<br>`exams`<br>`users` |

---

## 4. Admin Panel

| Feature | Explanation | API Endpoints | Database Tables |
|---|---|---|---|
| **Sign Up** | Registration form for staff to submit their name, email, phone, and password for admin privileges. Accounts are marked pending upon creation and cannot log in until approved by an administrator. | `POST /api/v1/admin/auth/register` | `users` |
| **Login** | Email and password sign-in for approved staff, storing access tokens in memory and refresh tokens in localStorage. Displays an informative pending message if unapproved and auto-refreshes tokens on 401. | `POST /api/v1/admin/auth/login`<br>`POST /api/v1/admin/auth/refresh`<br>`POST /api/v1/admin/auth/logout`<br>`GET /api/v1/admin/auth/me` | `users`<br>`auth_sessions`<br>`auth_refresh_tokens` |
| **Onboarding (Taxonomy Data)** | The admin panel has no student onboarding; it manages the underlying taxonomy data that drives it. Administrators create, rename, reorder, and archive exam subjects and categories for mobile and web. | `GET /api/v1/admin/taxonomy`<br>`POST /api/v1/admin/taxonomy/nodes`<br>`PATCH /api/v1/admin/taxonomy/nodes/:id` | `taxonomy_nodes`<br>`exams`<br>`users` |
