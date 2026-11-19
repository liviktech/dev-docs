# Plan: admin panel setup with signup and login

## What we are building

The first version of the Learning App admin panel (a new React web app in `learning-app-admin`) and the backend APIs it needs. Only two things work in this task: **admin signup** and **admin login**. After login the admin lands on an empty dashboard page.

The admin panel talks to the same backend as the mobile app (`learning-app-backend`). It uses its own set of routes under `/admin`, but the same database, the same users table and the same token system.

Nothing is built yet. This file is the plan only.

## Decisions already made

| Question | Answer |
|---|---|
| Who can create an admin account? | **Option A.** Anyone can fill the signup form, but the account is saved as `pending`. It cannot log in until an existing admin approves it. |
| How do admins log in? | **Email and password.** Email OTP (like the mobile app) can be added later if needed. |
| Signup fields | Name, email, phone (optional), password. |
| Admin levels | **One role only:** `admin`. No super admin or editor for now. |
| API naming | `/api/v1/admin/auth/...` (the backend rule is that admin routes carry `/admin` right after the version, and mobile routes carry no audience word). |
| Admin panel rules | Everything is a reusable component (button to table). Use shadcn for UI and TanStack Query for all API calls. |

## Defaults I picked (change them if you disagree)

| Topic | Default | Why |
|---|---|---|
| Where the browser keeps tokens | Access token in memory, refresh token in `localStorage` | The backend expects tokens in the request body and has no cookie support. Cookies would need backend changes. The risk is XSS, so the panel must never render raw HTML from users. |
| "Wrong email or password" message | One generic 401 for unknown email, wrong password, and a non-admin account | Stops people from finding out which emails are admins. |
| Duplicate email at signup | Still answers 409 "email already registered" | Same as mobile. Rate limited. We accept this small leak. |
| Frontend framework | Vite, React, TypeScript, React Router | The panel sits behind a login and needs no SEO, so Next.js is not needed. |
| Forms | react-hook-form with Zod | Zod is already the validation library in the backend, so the rules look the same on both sides. |
| Tailwind | Yes | shadcn is built on Tailwind. |
| Who approves the first admin and later ones | A small backend script (see below). There is no approval screen in this task. | The request was signup and login only. An approval screen is the next task. |

## The flow in simple words

1. **Signup.** A person opens the panel, fills name, email, phone (optional) and password. The backend saves them as an `admin` with status `pending` and answers "Account created, waiting for approval". No tokens are given.
2. **Approval.** An existing admin approves the account (for now with a script run by a developer). The status becomes `active`.
3. **Login.** The admin enters email and password. If the account is `active` and the role is `admin`, the backend returns an access token (15 minutes) and a refresh token (30 days).
4. **Using the panel.** The panel sends the access token with every request. When it expires, the panel quietly gets a new pair using the refresh token and retries.
5. **Logout.** The panel tells the backend to end the session and clears the tokens.

The very first admin cannot sign up and wait for approval (nobody exists to approve), so a developer creates the first one with the script below.

## Part 1: backend (`learning-app-backend`)

### What exists today and what we reuse

| Already there | We reuse it for |
|---|---|
| `users.role` already has `admin` | Admin accounts |
| `users.password_hash` and `PasswordService` (Argon2id) | Password signup and login |
| `auth_sessions` and `auth_refresh_tokens` tables, `TokenService` | Admin sessions and tokens, with the same refresh rotation and reuse detection |
| `AccessTokenGuard` (checks the session is live and the user is active) | The first half of the admin guard |
| Throttling, Zod pipe, response envelope, error filter, Swagger | Everything the same as mobile |

### What is missing and we add

1. **Database migration `0005`.** Add the value `pending` to the `user_status` enum (it has `active` and `disabled` today). PostgreSQL cannot drop an enum value, so the down migration leaves it in place (same as the `parent` role in migration 0003). The value is not used inside the same migration, which PostgreSQL requires.
2. **`AdminRoleGuard`.** Runs `AccessTokenGuard` first, then rejects with 403 unless the role is `admin`. The role is read from the database through the live session check, never trusted from the request.
3. **Admin auth controller** `auth-admin.controller.ts` in the existing auth module, with the routes below. It calls the existing auth service. No copied logic.
4. **Auth service changes.**
   - `adminRegister`: hash the password, insert a user with role `admin` and status `pending`. The role and status are set on the server. Unknown body fields are rejected.
   - `adminLogin`: same steps as the existing `login` (equal timing for unknown emails, generic 401), plus: the role must be `admin`, or it answers the same generic 401. A `pending` account gets 403 `ACCOUNT_PENDING_APPROVAL`. A `disabled` account gets 403 `ACCOUNT_INACTIVE`. The pending and disabled answers are only given after the password is correct, so they do not reveal accounts.
   - Refresh, logout and `me` reuse the existing methods. The refresh must also refuse a `pending` or `disabled` user (it already refuses non-active users).
5. **Zod schemas** for admin register and login in `schemas/`. Register: `fullName` (1 to 100), `email`, `phone` (optional, same rule as mobile), `password` (reuse the existing policy). Strict objects.
6. **Error codes** in `shared/constants/http.constants.ts`: `ACCOUNT_PENDING_APPROVAL`.
7. **Throttling.** Reuse `signup`, `login`, `refresh`, `logout` and `account` policies.
8. **CORS.** Add the admin panel origin (`http://localhost:5173` for development) to `CORS_ALLOWED_ORIGINS` in `.env` and note it in `.env.example`. Local only for now, the production origin is added later.
9. **Admin script** `admin:create` (makes the first admin, already `active`) and `admin:approve <email>` (turns a `pending` admin into `active`). The password is typed at a prompt and never passed on the command line or logged.
10. **Docs.** Update `architecture.md` (new endpoints, the `pending` status, the admin guard). While there, fix the two lines that are out of date (it says `suspended` where the database says `disabled`, and the signup section describes the old flow).

### New endpoints

| Method | Path | Auth | What it does |
|---|---|---|---|
| POST | `/api/v1/admin/auth/register` | none | Creates a `pending` admin. 201, no tokens. 409 if the email exists. |
| POST | `/api/v1/admin/auth/login` | none | Email and password. Returns user and tokens. |
| POST | `/api/v1/admin/auth/refresh` | refresh token in body | Rotates tokens. |
| POST | `/api/v1/admin/auth/logout` | refresh token in body | Ends the session. |
| GET | `/api/v1/admin/auth/me` | admin access token | Current admin. |

Response shape is the usual `{ success, message, data, requestId }`. Swagger tag: `Admin - Auth`.

### Safety points to keep

- A student or parent token must not work on admin routes (403), and a student account must not be able to log in through the admin route (generic 401).
- The mobile `POST /auth/login` and `/auth/signup` are older endpoints that the app does not use. The old login would also let an admin sign in. This is not a new hole (the admin still has to know the password), but flag it. **Left out on purpose:** changing the old endpoints is not part of this task.
- No logging of passwords, tokens or emails.

### Backend tests

Integration tests (new file for admin auth):

- Register: success creates a `pending` admin, validation failure, extra field like `role` rejected, duplicate email 409.
- Login: wrong password and unknown email give the same 401, a `pending` admin gets 403, a `disabled` admin gets 403, a student account gets 401, an `active` admin gets tokens.
- `me`: works with an admin token, 401 without a token, 403 with a student token.
- Refresh and logout work for admin sessions (the rotation and reuse rules are already covered, we only check they work through the admin routes).

Then run, as the backend rules require: `npm run typecheck`, `npm run lint`, `npm test`, `npm run test:integration`, and `migrate:up`, `migrate:down`, `migrate:up` for the new migration.

## Part 2: admin panel (`learning-app-admin`)

### Stack

| Need | Choice |
|---|---|
| App | Vite, React, TypeScript (strict) |
| Routing | React Router |
| UI | shadcn/ui on Tailwind |
| API calls and caching | TanStack Query (every server call goes through a query or mutation hook) |
| Forms | react-hook-form, Zod, `@hookform/resolvers` |
| Lint and tests | ESLint, Prettier, Vitest and Testing Library |
| Package manager | npm (same as the backend) |

We will check the current docs of each library before using it, because versions move fast.

### Folder structure

```text
src/
  components/
    ui/           shadcn primitives, generated, not edited by hand
    common/       OUR reusable components built on ui/ (the only ones pages may use)
  features/
    auth/
      api/        request functions and TanStack Query hooks
      components/ pieces only for auth (for example the signup form)
      pages/      SignupPage, LoginPage
      schemas/    Zod schemas
  layouts/        AuthLayout, DashboardLayout
  lib/            api client, token storage, query client, utils
  routes/         router, ProtectedRoute, PublicOnlyRoute
  pages/          DashboardPage (empty for now)
```

### The reusable component rule

The rule: **a page never uses a raw `<button>`, `<input>`, `<table>` and so on, and never styles one-off.** It uses a component from `components/common`. If a needed component does not exist, we add it to `components/common` first, then use it.

Components built in this task:

| Component | Used for |
|---|---|
| `AppButton` | Every button, with a loading state |
| `FormField` | Label, input, error message and hint as one unit |
| `TextInput`, `PasswordInput` (show and hide) | Text fields |
| `PhoneInput` | Optional phone field |
| `AuthCard` and `AuthLayout` | Centered card shell shared by signup and login |
| `PageHeader` | Page title row |
| `Toast` helper | Success and error messages |
| `LoadingScreen` and `ErrorMessage` | Loading and error states |

Planned but not built yet (so we keep the rule in mind): `DataTable` (the reusable table on top of TanStack Table), `ConfirmDialog`, `EmptyState`, `Pagination`.

### API layer

- One API client in `lib/` that adds the `Authorization` header, reads the backend's `{ success, data }` envelope, and turns error responses into one `ApiError` type with `statusCode`, `errorCode`, `message`, `requestId` and field `errors`.
- Request functions return typed data. The types come from Zod schemas.
- TanStack Query hooks per action: `useAdminSignup`, `useAdminLogin`, `useAdminLogout`, `useCurrentAdmin`.
- **Refresh handling.** On a 401 `INVALID_ACCESS_TOKEN`, refresh once and retry the request. Refresh tokens work only once, so refreshes must be queued: if several requests fail at the same moment they all wait for one single refresh call. On 401 `INVALID_REFRESH_TOKEN`, clear tokens and go to login.
- Base URL comes from `VITE_API_BASE_URL` (for example `http://localhost:3000/api/v1`). Add a `.env.example`.

### Pages

- **Signup page.** Name, email, phone (optional), password, confirm password. Client side rules match the backend (shown inline). On success, show "Account created. An admin must approve it before you can log in." with a link to login.
- **Login page.** Email and password. Clear messages for wrong details, pending approval and disabled account. Redirects to the dashboard on success.
- **Dashboard page.** Empty shell with the header and a logout button. Proves the protected route and the `me` call work.
- **Route guards.** Logged-out users are sent to login. Logged-in users who open login or signup are sent to the dashboard.

### Admin panel checks

- `npm run typecheck`, `npm run lint`, `npm run build`, `npm test`.
- Component tests for the form components and the signup and login pages (success, validation errors, server errors).
- A manual run in the browser against the real backend: signup, try login while pending, approve with the script, login, reload the page (stays logged in), wait for or force an expired access token (refresh works), logout.

## Order of work

1. Backend: migration, guard, service, controller, schemas, errors, CORS, script, tests, docs.
2. Admin panel: create the Vite app, Tailwind, shadcn, TanStack Query, router, lint and test setup.
3. Admin panel: reusable components, then the API client and token handling.
4. Admin panel: signup, login and dashboard pages, route guards.
5. End to end check in the browser, then fix and note anything that differs from this plan.

Both repos use the branch `feat-admin-auth`. The agent does not commit or push. Branch creation, commits and PRs are done by the developer.

## Out of scope on purpose

- **Email OTP login for admins.** Password only for now. Can be added later using the existing OTP service.
- **Approval screen and admin list in the panel.** Approval is done with the script in this task.
- **Forgot password and change password.** Until this exists, an admin who forgets the password needs a developer to reset it.
- **Admin levels and permissions.** One role only.
- **Cookie based tokens.** Needs backend changes.
- **Changing the mobile app.** Its login screen still shows email and password and does not call the API yet. That is a separate task.
- **Changing the older `/auth/signup` and `/auth/login` endpoints.**

## Risks and unknowns

- **First admin.** Without the script nobody can ever log in. The script must be easy to run and written in the setup notes.
- **Enum change.** `ALTER TYPE ... ADD VALUE` cannot be rolled back. We must run `migrate:up` on a copy first.
- **Parallel refresh.** If the panel sends two refreshes with the same token, the backend will treat it as theft and end the session. The queue in the API client prevents this and must be tested.
- **localStorage token.** Open to XSS if we ever render unsafe content. Revisit cookies before production.
- **Library versions.** shadcn, Tailwind and TanStack Query change often. Check the real docs when we set up, and write down any difference from this plan.
- **Docs drift.** The backend rules say `architecture.md` must be updated in the same change.

## Confirmed details

- **Local only for now.** The admin panel runs at `http://localhost:5173`, and that is the only origin added to CORS. There is no production URL yet. When a real https URL exists, add it to `CORS_ALLOWED_ORIGINS` (the backend rejects non-https origins in production).
- **Phone rule is the same as mobile.** Optional, 7 to 15 digits, optional leading `+`.
- **Password rule is the same as mobile.** 8 to 128 characters, no stricter admin rule for now.

## Update 2026-11-19: one email, one admin account and one mobile account

Found while testing: an admin could sign in to the mobile app, because the mobile login code never checked the role. Decision: admins use the admin panel only, students and parents use the mobile app only.

- The same email may have one admin account and one mobile account. They are separate accounts with separate passwords, sessions and codes.
- On mobile one email is either a parent or a student, never both.
- Migration `0007_email_unique_per_audience` replaces the single unique email index with two partial unique indexes (admin, and student or parent). It is not run yet; the user applies it.
- Every email lookup now takes an audience. Mobile endpoints only see student and parent accounts. Admin endpoints only see admin accounts. Refresh tokens of the other audience are rejected.
- An admin who wants the mobile app signs up there as a student or parent with the same email.
- Tests: `test/integration/auth-audience.int-spec.ts` (written, not run until the migration is applied).
