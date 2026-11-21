# Dashboard Developer Guide: Required Fields & Implementation

> **Target Audience:** Developers and AI Agents.  
> **Purpose:** Detailed reference for the Dashboard feature across Mobile App, Backend, Web App, and Admin Panel, covering administrative portal layouts, student navigation shells, authentication guards, user menus, code paths, APIs, and database schemas.

---

## 1. Required Fields & Originating Modules

In the Learning App ecosystem, the Dashboard architecture encompasses two key environments: the **Admin Portal Dashboard** (administrative management frame and operations hub) and the **Student Application Dashboard Shell** (the central navigation shell and Hub landing experience).

### 1.1 Admin Portal Dashboard
*Captured in the Admin Portal root (`/`) and framed by `DashboardLayout`.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Admin User ID** | Auto-derived | Admin JWT access token | `users.id` | Authenticated administrator identifier. |
| **Email** | Auto-derived | Admin profile | `users.email` | Normalized lower-case email in admin audience. |
| **Full Name** | Optional | Admin signup form | `users.full_name` | Display name of the administrator. |
| **Role** | Auto-derived | System assignment | `users.role` | Must be `'admin'`. Enforced by `AdminRoleGuard`. |
| **Account Status** | Auto-derived | Server status check | `users.status` | Must be `'active'`. `'pending'` accounts are restricted. |
| **Navigation Menu** | Static | System routing | Hardcoded in client | Four primary sections: Dashboard, Questions, Exams, Subjects. |

---

### 1.2 Student Application Dashboard Shell
*Framed by `AppShell` in the Web App and `AppTabs` in the Mobile App.*

| Field | Required | Where to Fill in App | Database Column | Notes |
|---|---|---|---|---|
| **Student User ID** | Auto-derived | Student JWT access token | `users.id` | Authenticated student identifier. |
| **Role** | Auto-derived | Registration role | `users.role` | Must be `'student'` (or `'parent'`). |
| **Onboarded At** | Auto-checked | Server completion check | `users.onboarded_at` | Non-null timestamp. Incomplete users redirected to onboarding. |
| **Navigation State** | Auto-derived | Client router | URL / Tab state | Web: Home, Practice, Mock exams, Stats, Profile.<br>Mobile: Hub, Tests, Cards, Stats, Profile. |
| **Theme Mode** | Optional | Theme toggle button | Client state / storage | Toggle between light and dark visual themes. |

---

## 2. Eligibility & Validation Rules Checklist

Before rendering or interacting with the Dashboard shells, the system verifies:
1. **Admin portal authorization:** Access to the Admin Dashboard requires an authenticated session with `role = 'admin'` and `status = 'active'`. Pending admin accounts are locked out with a warning screen.
2. **Student portal authorization:** Access to student dashboards requires a valid student session where `onboarded_at IS NOT NULL`. Unauthenticated requests redirect to `/login`; incomplete profiles redirect to `/resume`.
3. **Single-flight token refresh:** Both Admin and Student dashboard clients employ in-flight mutex locks on 401 responses, ensuring concurrent widget requests do not initiate duplicate token rotations.
4. **Audience isolation:** Admin credentials cannot access student dashboard endpoints, and student credentials cannot authenticate against admin dashboard routes.
5. **Responsive navigation breakpoints:**
   * On desktop viewports (>= 1024px), the shell displays a persistent left sidebar with brand identity, nav links, and user menu.
   * On mobile viewports (< 1024px), the shell collapses into a minimal top header and fixed bottom navigation tab bar.

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
│    GET /api/v1/admin/auth/me (Admin Identity)          │
│    GET /api/v1/student/home (Student Hub Payload)      │
│    POST /api/v1/admin/auth/logout                      │
│    POST /api/v1/auth/refresh (Token Rotation)          │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│           PostgreSQL Database (Neon DB)                │
│   tables: users, auth_sessions, auth_refresh_tokens,   │
│   student_profiles, assessment_templates               │
└────────────────────────────────────────────────────────┘
```

### 3.1 Mobile App (`src/`)
Implements the student dashboard shell via `AppTabs` (`components/app-tabs.tsx`) and `(tabs)/_layout.tsx`. Bottom tabs provide direct access to the Hub (`hub.tsx`), Tests (`tests.tsx`), Cards (`cards.tsx`), Stats (`stats.tsx`), and Profile (`profile.tsx`). The Hub acts as the dashboard landing view, organizing widgets, active test prompts, and recent practice sessions. Shared dashboard cards in `components/dashboard/` provide reusable charts and history rows.

* **Code References:**
  * Native tabs navigator: `src/components/app-tabs.tsx` (lines 8–46)
  * Tabs layout shell: `src/app/(tabs)/_layout.tsx` (lines 5–12)
  * Student Hub screen: `src/app/(tabs)/hub.tsx` (lines 35–225)
  * Shared dashboard components: `src/components/dashboard/` (`summary-card.tsx`, `subject-performance.tsx`, `test-history.tsx`, `question-lists.tsx`)
  * Post-auth router: `src/hooks/use-after-auth.ts` (lines 12–25)
* **API:** `GET /student/home`, `GET /student/stats`
* **DB Table:** `users`, `student_profiles`, `assessment_attempts`

---

### 3.2 Backend (`learning-app-backend`)
Provides profile resolution and session management powering both dashboards. `AuthAdminController.getMe` validates the admin JWT against `UsersRepository.findAdminById`, ensuring active status before returning staff details. `HomeController.getHome` consolidates student data into a single response, while token rotation endpoints (`/auth/refresh`, `/admin/auth/refresh`) keep dashboard sessions alive seamlessly.

* **Code References:**
  * Admin identity controller: `src/modules/auth/auth-admin.controller.ts` (lines 80–135)
  * Student home aggregator: `src/modules/home/home.controller.ts` (lines 20–50)
  * Access token & role guards: `src/modules/auth/guards/access-token.guard.ts` (lines 10–45), `src/modules/auth/guards/admin-role.guard.ts` (lines 10–35)
  * Session token service: `src/modules/auth/services/token.service.ts` (lines 40–95)
* **API:** `GET /api/v1/admin/auth/me`, `POST /api/v1/admin/auth/logout`, `GET /api/v1/student/home`, `POST /api/v1/auth/refresh`
* **DB Table:** `users`, `auth_sessions`, `auth_refresh_tokens`, `student_profiles`

---

### 3.3 Web App (`learning-web-app`)
Implements `AppShell` (`components/app/app-shell.tsx`), wrapping all signed-in pages in `src/app/(app)/layout.tsx`. Features a responsive layout with a 64-column desktop sidebar, brand header, user profile badge with sign-out dropdown (`user-menu.tsx`), theme switcher, and mobile bottom tab bar. Navigation links are wrapped in a Suspense boundary to support fast URL streaming in Next.js 16.

* **Code References:**
  * App shell frame: `src/components/app/app-shell.tsx` (lines 65–101)
  * Navigation links & suspense: `src/components/app/app-shell.tsx` (lines 13–63)
  * User menu & sign-out: `src/components/app/user-menu.tsx` (lines 15–55)
  * Signed-in layout wrapper: `src/app/(app)/layout.tsx` (lines 10–25)
  * Web student hub: `src/app/(app)/hub/page.tsx` (lines 20–95)
* **API:** `/api/student/home`, `/api/auth/logout`
* **DB Table:** `users`, `auth_sessions`

---

### 3.4 Admin Panel (`learning-app-admin`)
Provides the administrative portal shell via `DashboardLayout` (`layouts/dashboard-layout.tsx`). Displays a fixed desktop sidebar (`AppSidebar.tsx`) with links to Dashboard (`/`), Questions (`/questions`), Exams (`/exams`), and Subjects (`/subjects`), a top header with the admin profile name and a sign-out button, and an animated loader (`PageLoader.tsx`) inside a Suspense boundary. `DashboardPage` acts as the landing greeting and verifies session health.

* **Code References:**
  * Dashboard layout frame: `src/layouts/dashboard-layout.tsx` (lines 33–85)
  * Dashboard landing page: `src/pages/dashboard-page.tsx` (lines 6–21)
  * App sidebar component: `src/components/common/app-sidebar.tsx` (lines 15–75)
  * Protected route wrapper: `src/routes/protected-route.tsx` (lines 10–35)
  * Admin query hooks: `src/features/auth/api/auth.queries.ts` (lines 15–45)
* **API:** `GET /api/v1/admin/auth/me`, `POST /api/v1/admin/auth/logout`, `GET /api/v1/admin/mock-exams`
* **DB Table:** `users`, `auth_sessions`, `assessment_templates`, `taxonomy_nodes`

---

## 4. What Already Works (Zero Changes Needed)

1. **Audience-isolated dashboard barriers:** Strict database indexes and role guards prevent student sessions from accessing admin dashboard routes and vice versa.
2. **Seamless token refresh deduplication:** Both frontend dashboards queue concurrent 401 requests behind a shared refresh promise, preventing race conditions or premature logouts.
3. **Fluid cross-device responsiveness:** Desktop sidebar navigation effortlessly transitions to compact mobile headers and bottom tabs across screen sizes.
4. **Instant session revocation:** Clicking sign out instantly purges local storage tokens / httpOnly cookies and revokes server-side session records in the background.
5. **Route protection and deep linking:** Authenticated users trying to open auth pages are redirected to their respective dashboards, while expired sessions redirect cleanly to login.
