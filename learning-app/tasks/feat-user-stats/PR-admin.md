# 2026-11-21 - user-analytics (admin)

## Summary

The admin dashboard was an empty placeholder that only proved the login worked. This PR replaces it with a live analytics dashboard (six KPI cards, overview charts, peak times, top features, recent registrations and recent activity) and adds a full User Analytics module with seven sections (Overview, Growth, Engagement, Usage, Features, Academic, Parents) behind a new sidebar entry. The chart library (Recharts) is loaded on demand so it does not increase the login bundle.

## What changed

| File | Change |
|---|---|
| `src/pages/dashboard-page.tsx` | Replaced the empty placeholder with a live dashboard: period selector (7d, 30d, 90d), overview KPI panel, three highlight cards (peak hour, most active day, top features), and recent registrations and activity lists. |
| `src/layouts/dashboard-layout.tsx` | Added "User Analytics" as the fifth sidebar item with the `BarChart3Icon`. Imported `BarChart3Icon` from lucide. |
| `src/components/common/app-sidebar.tsx` | Added `overflow-x-auto` and `shrink-0` so the sidebar scrolls horizontally on small screens when there are more items. |
| `src/routes/routes.tsx` | Dashboard and User Analytics are now lazy-loaded. Added `analyticsSection` helper and seven child routes under `/user-analytics` (overview, growth, engagement, usage, features, academic, parents) with an index redirect to overview. Removed the static `DashboardPage` import. |
| `src/index.css` | Changed the chart colour palette from grey oklch values to the brand blue, teal, light blue, slate and amber (`--chart-1` through `--chart-5`) for both light and dark themes. |
| `package.json` | Added `recharts`, `date-fns` and `react-day-picker`. |
| `package-lock.json` | Lock file updated for the three new dependencies. |
| `README.md` | Added an Analytics section documenting the dashboard, the User Analytics sidebar entry, the backend dependency (migration 0015), the chart stack, and the first-run empty state. |
| `src/components/analytics/analytics-area-chart.tsx` (new) | Reusable area chart wrapper. |
| `src/components/analytics/analytics-bar-chart.tsx` (new) | Reusable bar chart wrapper. |
| `src/components/analytics/analytics-chart-card.tsx` (new) | Card frame for a chart with title and info tip. |
| `src/components/analytics/analytics-chart-frame.tsx` (new) | Responsive container that sizes charts. |
| `src/components/analytics/analytics-chart-parts.tsx` (new) | Shared axis, grid and legend parts used across charts. |
| `src/components/analytics/analytics-chart-skeleton.tsx` (new) | Skeleton placeholder while chart data loads. |
| `src/components/analytics/analytics-chart-tooltip.tsx` (new) | Custom tooltip for all charts. |
| `src/components/analytics/analytics-chart.types.ts` (new) | Shared TypeScript types for chart components. |
| `src/components/analytics/analytics-composed-chart.tsx` (new) | Composed (multi-series) chart wrapper. |
| `src/components/analytics/analytics-date-filter.tsx` (new) | Date range picker used in the filter bar. |
| `src/components/analytics/analytics-empty-state.tsx` (new) | Empty state shown when a chart has no data. |
| `src/components/analytics/analytics-heatmap.tsx` (new) | Hour-by-day heatmap for usage patterns. |
| `src/components/analytics/analytics-info-tip.tsx` (new) | Small info icon with a tooltip explaining a metric. |
| `src/components/analytics/analytics-kpi-card.tsx` (new) | KPI card with value, comparison and trend arrow. |
| `src/components/analytics/analytics-line-chart.tsx` (new) | Reusable line chart wrapper. |
| `src/components/analytics/analytics-pie-chart.tsx` (new) | Reusable pie chart wrapper. |
| `src/components/analytics/analytics-stacked-bar-chart.tsx` (new) | Stacked bar chart wrapper. |
| `src/components/ui/calendar.tsx` (new) | shadcn/ui Calendar component (needed by the date picker). |
| `src/components/ui/chart.tsx` (new) | shadcn/ui Chart component that wraps Recharts with theme colours. |
| `src/components/ui/popover.tsx` (new) | shadcn/ui Popover component (needed by the date picker). |
| `src/components/ui/tooltip.tsx` (new) | shadcn/ui Tooltip component (needed by chart tooltips). |
| `src/features/analytics/api/analytics.api.ts` (new) | API client functions for all admin analytics endpoints. |
| `src/features/analytics/api/analytics.queries.ts` (new) | React Query hooks (`useOverview`, `useGrowth`, `useEngagement`, `useUsage`, `useFeatures`, `useAcademic`, `useParents`). |
| `src/features/analytics/components/analytics-filter-bar.tsx` (new) | Shared filter bar (range, user type, platform, exam, comparison toggle) used across all analytics sections. |
| `src/features/analytics/components/analytics-report-note.tsx` (new) | Small note showing the report period, timezone and last-updated time. |
| `src/features/analytics/components/overview-panel.tsx` (new) | The six KPI cards (total users, active users, new registrations, avg session duration, students, parents) with comparison to the previous period. Used by both the dashboard and the overview section. |
| `src/features/analytics/components/recent-lists.tsx` (new) | Recent registrations and recent activity card components for the dashboard. |
| `src/features/analytics/hooks/use-analytics-filters.ts` (new) | Hook that reads and writes filter state from the URL search params. |
| `src/features/analytics/lib/definitions.ts` (new) | Label maps and colour assignments for user types, platforms, features and event names. |
| `src/features/analytics/lib/filters.ts` (new) | Filter types (`RangeValue`, `AnalyticsFilters`), defaults and serialisation. |
| `src/features/analytics/lib/format.ts` (new) | Formatting helpers (`formatNumber`, `formatDuration`, `formatHour`, `weekdayName`, `shortDate`). |
| `src/features/analytics/lib/query-error.ts` (new) | Error handling for analytics API responses. |
| `src/features/analytics/lib/analytics-lib.test.ts` (new) | Unit tests for the formatting and filter helpers. |
| `src/features/analytics/schemas/analytics.schemas.ts` (new) | Zod schemas matching the backend response shapes. |
| `src/features/analytics/pages/user-analytics-layout.tsx` (new) | Layout with a tab bar for the seven analytics sections. |
| `src/features/analytics/pages/overview-section.tsx` (new) | Overview section: KPIs, daily active users chart, registrations chart, session duration chart. |
| `src/features/analytics/pages/growth-section.tsx` (new) | Growth section: cumulative users, new vs returning, registrations by source. |
| `src/features/analytics/pages/engagement-section.tsx` (new) | Engagement section: sessions per user, session duration distribution, retention. |
| `src/features/analytics/pages/usage-section.tsx` (new) | Usage section: hourly heatmap, platform breakdown, device types. |
| `src/features/analytics/pages/features-section.tsx` (new) | Features section: feature popularity, event trends, feature comparison. |
| `src/features/analytics/pages/academic-section.tsx` (new) | Academic section: accuracy by subject, mock exam score trends, study time correlation. |
| `src/features/analytics/pages/parents-section.tsx` (new) | Parents section: parent activity, linked children, dashboard views. Shows "not available" when disabled on the backend. |

## Why

The admin had no way to see how students use the apps. The placeholder dashboard showed nothing after login. The request was for a full analytics dashboard and a dedicated User Analytics area so the admin can track usage, growth, engagement, feature popularity and academic progress.

## How

**Lazy loading**: both the dashboard page and the entire User Analytics module are loaded with `lazy` imports in the router. The 17 chart components and Recharts itself are only downloaded when the admin opens the dashboard or a User Analytics section. The login bundle stays small.

**Reusable chart building blocks**: `src/components/analytics/` has one wrapper per chart type (area, bar, line, pie, stacked bar, composed, heatmap) plus shared parts (axes, grid, legend, tooltip, skeleton, empty state, card frame). Every analytics section composes from these.

**Filters in the URL**: the `useAnalyticsFilters` hook reads range, user type, platform, exam and comparison from `URLSearchParams` and writes them back on change. Filters are shareable by copying the URL.

**Overview panel shared between dashboard and User Analytics**: the six KPI cards are in `OverviewPanel` with a `variant` prop. The dashboard uses `variant="dashboard"` (compact) and the overview section uses the full layout.

**Kept the same on purpose**: no existing component, page, route or test was changed in behaviour outside of the dashboard page (which was an empty placeholder). The sidebar component only got a scroll fix for small screens.

## Decisions made

**Charts loaded on demand.** Recharts is large. Making the dashboard and analytics routes lazy keeps the login and question management bundles fast.

**One chart component per type, not one per use.** Each section just passes data and config to the same area/bar/line/pie wrapper, so adding a new chart is a few lines, not a new component.

**Filters in the URL, not in React state.** The admin can bookmark or share a filtered view. The hook handles serialisation both ways.

## Notes

- The analytics data comes from the backend's `/admin/analytics/*` endpoints (see `PR-backend.md`). Migration `0016` must be applied before anything loads.
- A fresh deployment shows empty charts until the backend scheduler has run its first cycle (about a minute after startup).
- The chart palette was changed from grey to the brand colours (blue, teal, light blue, slate, amber) for both light and dark themes.
- Three new shadcn/ui components were added (Calendar, Popover, Tooltip) because the date picker and chart tooltips need them.

## Test plan

**Automated tests (passing):**
- [x] `src/features/analytics/lib/analytics-lib.test.ts`: formatting helpers, filter serialisation, range defaults.
- [x] `npx tsc --noEmit`: no errors.
- [ ] `npm run lint` and `npm run format:check`: not run.

**Manual checks (local):**
- [x] Dashboard loads with KPI cards, overview charts, highlight cards and recent lists. Period selector (7d, 30d, 90d) switches data. Checked live at `localhost:5173`.
- [x] "User Analytics" appears as the fifth sidebar item. Clicking it opens the tab-bar layout with all seven sections. Checked live.
- [x] Filters (range, user type, platform) update the URL and persist on page refresh. Checked live.
- [ ] Dark mode charts: not checked. The CSS variables are set but dark mode was not toggled.
