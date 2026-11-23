# 2026-11-21 - admin-header-tabs-inter-font (admin)

## Summary

The admin panel had a large page title with a description under it, pill style tab switches, the Geist font, and a lot of empty space around every page. This PR gives every page one slim header bar with an icon and the title (no description), adds one reusable folder style tab component and uses it for every tab switch, moves the whole app to the Inter font from one place, and reduces the page margins. The sidebar is not changed.

I only checked that the type check passes and that the production build works. I did not open the pages in the browser, so the look in light and dark mode is not checked yet.

**The login and signup card descriptions are not included on purpose** - see "Decisions made" below. This was a choice, not a miss.

## What changed

| File | Change |
|---|---|
| `src/components/common/page-header.tsx` | Rebuilt as a slim white bar with a border. Props are now `title`, `icon` and `actions`. The `description` prop is removed. |
| `src/components/common/section-tabs.tsx` (new) | Reusable row of folder style tabs. Props: `items`, `value`, `onChange`, `label`. Scrolls sideways on narrow screens. |
| `src/components/common/section-tab.tsx` (new) | One tab. The active tab is filled and bold with rounded top corners, inactive tabs are pale and turn accent on hover. At least 44px tall. |
| `src/components/common/exam-tabs.tsx` | The SAT/ACT switch now uses `SectionTabs`. Same props as before, so the two pages that use it did not need changes. |
| `src/features/analytics/pages/user-analytics-layout.tsx` | Header description removed, icon added, pill tabs replaced with `SectionTabs`. |
| `src/features/analytics/pages/growth-section.tsx` | The "Group by" switch now uses `SectionTabs`. |
| `src/pages/dashboard-page.tsx` | Header description removed and icon added. The 7/30/90 days switch now uses `SectionTabs`. The welcome text is gone, so the unused `useCurrentAdmin` call and import are removed. |
| `src/features/content/pages/questions-page.tsx` | Description removed, icon added. |
| `src/features/content/pages/subjects-page.tsx` | Description removed, icon added. |
| `src/features/content/pages/import-page.tsx` | Description removed, icon added. |
| `src/features/content/pages/question-form-page.tsx` | Descriptions removed on both headers (add and edit), icon added. The edit page's "published or draft" text is gone with it. |
| `src/features/exams/pages/exams-page.tsx` | Description removed, icon added. |
| `src/features/exams/pages/exam-editor-page.tsx` | Description removed, icon added. |
| `src/layouts/dashboard-layout.tsx` | Page padding reduced from `p-4 md:p-8` to `p-3 md:p-4`, and the gap between blocks from `space-y-6` to `space-y-4`. The sidebar part of this file is not touched. |
| `src/index.css` | Font import and `--font-sans` switched from Geist to Inter. |
| `package.json`, `package-lock.json` | Added `@fontsource-variable/inter`, removed `@fontsource-variable/geist`. |
| `src/components/ui/tabs.tsx` | No real change. It is no longer used by any page but I left the shadcn file in place. |

## Why

The pages did not look the same and each one made its own title and tab style. One header and one tab component means a change in one file shows everywhere. The descriptions under the titles were not needed.

The easy part was swapping the call sites. The real work was making the tabs look right using only the existing theme colours, so light and dark mode keep working with no new tokens.

## How

**Header**: a white card colour bar with a border, 64px minimum height. The icon uses the primary colour and the title uses the normal text colour. Actions sit on the right and wrap on small screens.

**Tabs**: the strip has a line under it in the primary colour and the active tab sits on that line. Colours come from existing tokens only (`primary`, `primary-foreground`, `secondary`, `secondary-foreground`, `accent`, `ring`). The strip uses `role="tablist"` and each tab uses `role="tab"` with `aria-selected`.

**Font**: the font is set in one place, `src/index.css`. The import and `--font-sans` are there, and `--font-heading` follows `--font-sans`, so every page and every shadcn component picks it up. To change the font again, edit those two lines only.

**Same on purpose**: the sidebar, `ui/tabs.tsx`, the analytics URL handling (the filters still carry over between sections) and the `ExamTabs` props.

## Decisions made

**Login and signup descriptions are left as they are.** The request was about the page header text. The auth cards are a different component, so I did not touch them.

**Dashboard welcome text was dropped, not moved.** It was part of the description under the header, so it went with it.

**Dark mode active tab.** The dark `--primary` is a light grey, so the active tab shows as a light pill with dark text. It is readable, so I did not change the dark theme.

## Notes

- `ui/tabs.tsx` is now unused. It can be deleted later if nobody wants it back.
- The page padding change affects every page that uses `DashboardLayout`. If it feels too tight, it is one line to loosen.

## Test plan

**Automated tests (no test files in this project):**
- [x] `npx tsc -b`: passes with no errors
- [x] `npx vite build`: builds successfully

**Manual checks (local):**
- [ ] Open each page (Dashboard, Questions, Add and Edit question, Import, Subjects, Exams, Exam editor, User Analytics) and check the new header and spacing - not done yet, I only ran the type check and build
- [ ] Switch tabs on User Analytics, Growth "Group by", Dashboard days, and SAT/ACT on Subjects and Import - not done yet
- [ ] Check light and dark mode and a phone width - not done yet
- [ ] Check the Inter font loads - not done yet, the build includes the font package
