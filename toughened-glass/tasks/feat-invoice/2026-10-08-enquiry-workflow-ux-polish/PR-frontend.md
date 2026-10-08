# 2026-10-08 - enquiry-workflow-ux-polish (frontend)

## Summary
This PR refines the enquiry management workflow with instant reactive filtering, a dedicated read-only details view dialog, and status-based action controls. It separates viewing from editing so clicking an enquiry code opens structured summary details, disables editing on approved, converted, or closed enquiries, prevents row click event bubbling on photo and action buttons, restricts enquiry dates to today or earlier, and locks the active account checkbox during new party creation.

## What changed
| File | Change |
|---|---|
| `src/features/ai-extraction/enquiry-photos-button.tsx` | Added stopPropagation to the photo button click handler to prevent triggering table row selection or row clicks. |
| `src/features/enquiries/enquiries-page.tsx` | Replaced manual Apply and Reset buttons with instant reactive date and party filtering, added a ViewDetailsDialog triggered by clicking enquiry codes, disabled edit actions on approved, converted, or closed enquiries, and added click isolation to action cells. |
| `src/features/enquiries/enquiry-form-modal.tsx` | Added maxDate constraint to prevent selecting future enquiry dates, simplified form labels and placeholders, streamlined dialog header padding, and standardized button label to Save. |
| `src/features/parties/party-dialog.tsx` | Disabled the Account is Active checkbox during new party creation so new parties cannot be created in an inactive state accidentally. |

## Why
Users encountered several workflow and interaction friction points on the enquiries page. Filtering required clicking an Apply button every time a date or party changed, adding extra steps to basic browsing. Clicking an enquiry code opened the full edit modal even when the user only wanted to review notes, dates, or contact info, risking unintended edits. Approved, converted, and closed enquiries showed active edit buttons despite their locked status in business logic. Clicking the photos button or action buttons bubbled up to table row handlers. The enquiry date picker allowed future dates, which does not make sense for customer enquiries. Finally, creating a new party allowed unchecking the active account status before saving, resulting in immediately inactive parties.

## How
**Instant reactive filtering**: Removed Apply and Reset buttons from the filter toolbar. Updating party selection or start and end dates directly updates the active filter state and triggers live query refetching.
**Dedicated enquiry details dialog**: Added ViewDetailsDialog to render key enquiry metadata (code, date, status, party, project, customer reference, sales rep, source, photo count, and remarks) when users click the enquiry code badge.
**Status-aware edit locking**: In the actions column, disabled the Edit button when enquiry status is APPROVED, CONVERTED, or CLOSED, applying muted styling and explanatory tooltip text.
**Event propagation isolation**: Added stopPropagation to photo and action buttons so clicks inside table cells do not trigger row selection or modal popups.
**Date and form polishing**: Set maxDate to the current date on the enquiry date picker, reduced modal header padding with headerClassName="py-2.5 px-4", simplified submit label to "Save", and polished form labels across the modal.
**Party creation guard**: Added disabled={!Boolean(party)} to the Account is Active checkbox in PartyDialog, keeping it checked and read-only for new parties while remaining toggleable for existing records.

## Decisions made
**Used ViewDetailsDialog on code click instead of opening the edit modal.** Reviewing requirements is more frequent than modifying them. Opening a read-only dialog prevents accidental form dirtiness while giving a cleaner summary layout.
**Disabled edit button on terminal and approved states instead of hiding it.** Keeping the button visible in a disabled state with a tooltip clarifies why the enquiry cannot be altered rather than confusing users with missing action icons.

## Notes
- Verified production build live with npm run build (vite build && tsc -b), passing cleanly with exit code 0.

## Test plan
**Automated tests (all passing):**
- [x] `npm run build`: Vite build and TypeScript compilation completed with exit code 0.

**Manual checks (local):**
- [x] Verified changing party or date range filters updates the enquiries table immediately without clicking an Apply button.
- [x] Verified clicking an enquiry code opens the ViewDetailsDialog displaying all details.
- [x] Verified clicking the edit button opens the edit modal only for open enquiries, and displays disabled styling on approved or closed records.
- [x] Verified clicking the photos button opens the photo panel without bubbling to row click handlers.
- [x] Verified enquiry date picker disallows selecting future dates.
- [x] Verified new party creation dialog has Account is Active checkbox disabled and checked.

