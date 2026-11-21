# 2026-11-21 - user-analytics (backend)

## Summary

There was no usage tracking. The backend knew what students answered and when they logged in, but nothing about how they actually use the apps (which screens, how long, what features, when they are active). This PR adds an `analytics` module with client-side session and event ingestion, a background aggregation job, and admin report endpoints that the admin dashboard and User Analytics pages consume.

The migration (`0016_analytics.sql`) creates all needed tables and indexes. The type check passes. Unit tests for the library functions were written. Integration tests were written but not run against a database in this session.

**Parent analytics are behind a flag (`ANALYTICS_PARENT_ENABLED`)** and default to off. This was a choice, not a miss, because parent accounts and linking already exist but parent usage volume is still zero.

## What changed

| File | Change |
|---|---|
| `src/database/migrations/0016_analytics.sql` (new) | Tables for sessions, events, daily aggregates, feature usage, academic snapshots and purge tracking. Down migration drops everything. |
| `src/config/analytics.config.ts` (new) | Reads the `ANALYTICS_*` env vars into a typed config object. |
| `src/config/env.schema.ts` | Adds the `ANALYTICS_*` variables with Zod validation (timezone, intervals, retention, cache TTL). |
| `.env.example` | Documents every new analytics variable with defaults. |
| `src/modules/analytics/analytics.module.ts` (new) | Wires controllers, services, repositories, the scheduler and the report cache. |
| `src/modules/analytics/analytics-client.controller.ts` (new) | Client routes: session start, heartbeat, end, and event batch. Behind `AnalyticsClientGuard`. |
| `src/modules/analytics/analytics-admin.controller.ts` (new) | Admin routes: overview, growth, engagement, usage, features, academic and parents reports, plus a manual backfill trigger. Behind `AdminRoleGuard`. |
| `src/modules/analytics/guards/analytics-client.guard.ts` (new) | Validates the caller is a signed-in student (or parent when enabled). |
| `src/modules/analytics/dto/analytics.dto.ts` (new) | DTOs for client requests. |
| `src/modules/analytics/schemas/analytics-request.schemas.ts` (new) | Zod schemas for all admin query parameters and client payloads. |
| `src/modules/analytics/schemas/analytics-response.schemas.ts` (new) | Zod schemas for every admin report response shape. |
| `src/modules/analytics/interfaces/analytics.interfaces.ts` (new) | TypeScript interfaces used across the module. |
| `src/modules/analytics/services/analytics-tracking.service.ts` (new) | Handles session start, heartbeat, end and event writes. |
| `src/modules/analytics/services/analytics-session.service.ts` (new) | Session timeout and expiry logic. |
| `src/modules/analytics/services/analytics-aggregation.service.ts` (new) | Rolls raw events and sessions into daily aggregates, feature usage and academic snapshots. |
| `src/modules/analytics/services/analytics-reports.service.ts` (new) | Builds the seven admin report shapes from aggregated data. |
| `src/modules/analytics/services/analytics-report-cache.ts` (new) | In-memory TTL cache for expensive report queries. |
| `src/modules/analytics/services/analytics-scheduler.service.ts` (new) | Interval-based job: syncs events, expires sessions, aggregates, purges old data. Backfills history on first run. |
| `src/modules/analytics/repositories/analytics-sessions.repository.ts` (new) | SQL for session CRUD and timeout queries. |
| `src/modules/analytics/repositories/analytics-events.repository.ts` (new) | SQL for event writes and raw event reads. |
| `src/modules/analytics/repositories/analytics-aggregation.repository.ts` (new) | SQL for daily rollup inserts and updates. |
| `src/modules/analytics/repositories/analytics-reports.repository.ts` (new) | SQL for all admin report queries (with filters, comparisons and date ranges). |
| `src/modules/analytics/repositories/analytics-academic.repository.ts` (new) | SQL for academic performance snapshots (accuracy by subject, mock exam scores). |
| `src/modules/analytics/lib/analytics-range.ts` (new) | Date range calculation from filter parameters (7d, 30d, 90d, custom). |
| `src/modules/analytics/lib/analytics-metrics.ts` (new) | Pure metric computation helpers (percentage change, averages, bucketing). |
| `src/modules/analytics/analytics.constants.ts` (new) | Event names, session states and other module constants. |
| `src/app.module.ts` | Registers `AnalyticsModule` and loads `analyticsConfig`. |
| `architecture.md` | Adds the analytics row to the scope table, a new section 10j describing the module, and updates the parent section reference from 10i to 10j. |
| `test/unit/analytics-lib.spec.ts` (new) | Unit tests for the range and metrics library functions. |
| `test/integration/analytics.int-spec.ts` (new) | Route-level tests for client and admin endpoints. |
| `test/integration/setup-env.ts` | Sets `ANALYTICS_REPORT_CACHE_SECONDS=0` so tests get fresh data. |

## Why

The admin panel had no insight into how students use the apps. Practice counts and mock exam scores exist, but nothing about sessions, screen views, feature popularity, peak times or growth trends. The admin dashboard and User Analytics pages in the admin panel need this data.

## How

**Client-side tracking**: the web and mobile apps call four endpoints: session start (on page load or app open), heartbeat (every few minutes while active), session end (on logout or tab close), and event batch (screen views and feature taps). The guard rejects calls from non-student users unless parent tracking is enabled.

**Background aggregation**: a scheduler runs every `ANALYTICS_JOB_INTERVAL_SECONDS` (default 60s). Each cycle it expires stale sessions, rolls raw events and sessions into daily aggregate rows, snapshots academic data, and purges raw data older than `ANALYTICS_RETENTION_DAYS`. On the first cycle after startup it backfills any missing past days so a fresh deployment catches up.

**Admin reports**: seven report endpoints (overview, growth, engagement, usage, features, academic, parents) read from the aggregate tables. Each accepts range, user type, platform, exam and comparison filters. Reports are cached in memory for `ANALYTICS_REPORT_CACHE_SECONDS` to avoid repeated heavy queries.

**Kept the same on purpose**: no existing table, route, migration, module or test was changed in behaviour. The only touched files outside the new module are the app module (to register it), the env schema (to validate new vars), architecture.md (to document it) and the test setup (to disable caching in tests).

## Decisions made

**Aggregate tables instead of querying raw events.** Raw events can grow fast. The scheduler pre-computes daily rollups so admin reports are always fast, even over 90 days. Raw data is purged after the retention window.

**In-memory report cache.** A simple TTL map avoids hitting the database on every dashboard load. It resets on deploy, which is fine because the data only changes once per aggregation cycle.

**Parent analytics behind a flag.** Parent accounts exist but have no real usage yet, so the parent report returns "not available" until `ANALYTICS_PARENT_ENABLED` is set to true.

## Notes

- Migration `0016` has to be run (`npm run migrate:up`) before the routes or the scheduler work. There is a known ordering issue if migration `0015_parent-child-links` was already applied: `0016_analytics` must sort after it alphabetically.
- The scheduler is disabled in tests (`ANALYTICS_SCHEDULER_ENABLED` defaults to true but integration tests do not set it, and setup-env sets the cache to 0).
- A new deployment shows empty charts until the first scheduler cycle completes (about a minute), which backfills history.
- `POST /admin/analytics/backfill` lets an admin manually rebuild past aggregates if needed.

## Next steps

- Run `npm run test:integration` against a local database and fix anything it finds.
- Run `npm run lint` and `npm run format:check`.
- Apply migration `0016` to the shared database.

## Test plan

**Automated tests (unit passing, integration not run):**
- [x] `test/unit/analytics-lib.spec.ts`: range calculation, metric helpers, percentage change, bucketing. Passed.
- [x] `npx tsc --noEmit`: no errors.
- [ ] `test/integration/analytics.int-spec.ts`: written for client session lifecycle, event batching, admin report endpoints with filters, guard rejections, and the backfill trigger. Not run, because no database was set up in this session.
- [ ] `npm run lint` and `npm run format:check`: not run.

**Manual checks (local):**
- [ ] Migration `0016` up and down against a database: not run. The integration tests cover it once they are run.
- [ ] Admin report endpoints called by hand: not run. The admin panel calls them (see `PR-admin.md`).
