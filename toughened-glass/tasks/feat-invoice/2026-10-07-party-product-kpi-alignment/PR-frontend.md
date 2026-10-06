# 2026-10-07 - party-product-kpi-alignment (frontend)

## Summary
This PR aligns the KPI cards on the Party and Products pages with the compact sizing, padding, and layout used in Order Booking. It updates each KPI card to use `size="compact"`, reducing outer padding, icon container dimensions, and internal spacing, and sets the grid wrapper to `gap-2.5 shrink-0` to eliminate visual discrepancies across master pages.

## What changed
| File | Change |
|---|---|
| `src/routes/party.tsx` | Added size="compact" to all 4 KPI cards and updated grid container to gap-2.5 shrink-0 to match Order Booking sizing. |
| `src/routes/products.tsx` | Added size="compact" to all 4 KPI cards and updated grid container to gap-2.5 shrink-0 to match Order Booking sizing. |

## Why
The KPI cards on the Party (`/party`) and Products (`/products`) pages used the default card size (`p-3.5` padding, `p-2` icon containers, and wider grid spacing). This made them noticeably taller and vertically disproportionate compared to the compact KPI cards in Order Booking (`/orders`) and other master screens. Standardizing them creates a consistent, cohesive dashboard feel across the ERP.

## How
**Compact card sizing**: Added `size="compact"` to each `<KpiCard />` component across both pages. This reduces card padding from `p-3.5` to `p-2.5`, icon container padding from `p-2` to `p-1.5` with `w-3.5 h-3.5` icons, and internal label/description margins.

**Grid container alignment**: Updated the card grid wrapper classes from `gap-3` to `gap-2.5 shrink-0`, matching the layout container in Order Booking (`header-detail-tab.tsx`).

## Decisions made
**Retained explicit typography class overrides.** Both pages already pass `labelClassName="text-xs"`, `valueClassName="text-2xl"`, and `descriptionClassName="text-xs"`. Retaining these overrides keeps value legibility high while letting `size="compact"` handle the outer padding and icon bounds.

## Notes
- Verified production build live with `npm run build` (`vite build && tsc -b`), passing with exit code 0 and zero TypeScript errors.

## Test plan
**Automated tests (all passing):**
- [x] `npm run build`: Vite build and TypeScript compilation completed with exit code 0.

**Manual checks (local):**
- [x] Verified `/party` page displays all 4 KPI cards with compact padding and matching height.
- [x] Verified `/products` page displays all 4 KPI cards with compact padding and matching height.
- [x] Compared side-by-side with Order Booking (`/orders`) KPI cards to confirm padding, font size, and spacing alignment.
