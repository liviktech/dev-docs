# 2026-10-07 - enquiry-datepicker-refinements (frontend)

## Summary
This PR adds custom enquiry sources and fast-create flows for projects and sales reps in the enquiry modal, implements date range bounds and validation across date pickers, and refines calendar styling. It also defaults the enquiries page filter to open records without an automatic current-month restriction and optimizes the photo review layout for enquiries.

## What changed
| File | Change |
|---|---|
| `src/components/ui/calendar.tsx` | Added dark highlight for current day when no date is selected, updated selected date styling to slate-900, and integrated selection context state. |
| `src/components/ui/date-picker.tsx` | Added minDate and maxDate prop support with matcher constraints and selection guards, adjusted layout to justify-between, and added text truncation. |
| `src/features/ai-extraction/photo-review-panel.tsx` | Switched enquiry screen photo preview to a centered single-column dialog (max-w-[780px]) instead of a side-by-side split grid with empty reading fields. |
| `src/features/enquiries/api.ts` | Relaxed enquirySource type in CreateEnquiryInput from enum literal to string. |
| `src/features/enquiries/enquiries-page.tsx` | Changed default status filter to OPEN, removed default month range filter to avoid hiding records, added start and end date range validation, added enquiry code to status toasts, and adjusted filter tab order. |
| `src/features/enquiries/enquiry-form-modal.tsx` | Added custom enquiry source creation dialog with WhatsApp default, enabled adding projects directly on existing parties with auto-tabbing, added sales rep creation dialog trigger, and updated form labels. |
| `src/features/order-booking/header-detail-tab.tsx` | Added minDate and maxDate bounds to booking date filters, widened date picker inputs to 132px, and added start and stop date validation toast. |
| `src/features/parties/party-dialog.tsx` | Added initialTab prop, defaulting to basic and opening directly to projects with an empty project row pre-populated when opened from enquiry project selection. |
| `src/features/pi-status/pi-status-page.tsx` | Added minDate and maxDate bounds to PI status date filters, widened date picker inputs to 132px, and added validation toast preventing stop date from preceding start date. |

## Why
Users faced several usability friction points across enquiry management and date filtering. When entering an enquiry, users could not add custom enquiry sources or add a new project for an already selected party without navigating away to the party master page. Date pickers allowed users to select an end date earlier than a start date without visual constraints or feedback, leading to confusing empty filter results. Additionally, the default date filter on the enquiries list restricted the view to the current month, giving the impression that older open enquiries were missing, and the photo preview modal for enquiries showed an empty right column intended for order line item extraction.

## How
**Custom sources in enquiry modal**: Added an inline dialog to create custom sources on the fly, storing them in component state alongside WhatsApp, Phone, and Email, and passing the selected key to the enquiry payload.
**Direct project addition for parties**: Updated PartyDialog to accept initialTab="projects" and automatically append a new blank project item. The enquiry modal now passes the currently selected party and sets the initial tab to projects when "+ Add New Project" is clicked, updating the party via mutation and auto-selecting the newly created project upon save.
**Date bounds and validation**: Added minDate and maxDate props to DatePicker, parsing ISO or formatted date strings and passing a matcher to Calendar so out-of-range dates cannot be clicked. Added validation toasts and auto-adjustment in EnquiriesPage, HeaderDetailTab, and PIStatusPage so stop dates cannot precede start dates.
**Enquiry list defaults and toasts**: Initialized fromDate and toDate filters to empty strings so all open enquiries are visible by default, and updated toast notifications on status update and approval to include the specific enquiry code.
**Dedicated photo review modal layout**: Checked screen === 'enquiry' in PhotoReviewPanel to render a single-pane photo view with zoom toggling inside a 780px modal, bypassing the two-column grid.

## Decisions made
**Passed selected party into PartyDialog instead of a detached project dialog.** Reusing PartyDialog with initialTab="projects" preserves consistent party validation and backend payload handling without having to create an isolated project-only API and form.
**Widened date picker containers to 132px.** Increasing width from 114px prevents formatted dates (e.g. dd/MM/yyyy) from overflowing or wrapping inside the input buttons.

## Notes
- Verified production build live with npm run build (vite build && tsc -b), passing with exit code 0.

## Test plan
**Automated tests (all passing):**
- [x] `npm run build`: Vite build and TypeScript compilation completed with exit code 0.

**Manual checks (local):**
- [x] Verified enquiry form modal opens and allows adding custom sources (e.g. WhatsApp, Walk-in) and selecting them.
- [x] Verified clicking "+ Add New Project" with a party selected opens the party dialog directly on the Projects tab with a new blank project row.
- [x] Verified date picker calendar disables dates before minDate and after maxDate, preventing invalid date range selections in enquiries, order booking, and PI status.
- [x] Verified photo review dialog displays single photo pane on enquiry screen instead of two-column layout.

