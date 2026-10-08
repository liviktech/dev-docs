# 2026-10-07 - master-code-cascade-updates (frontend)

## Summary
This PR fixes product editing by preventing sub-products from being wiped on update, normalizes child entity IDs, and preserves sub-product associations in rate rows. It also adjusts table column alignment for product, spec, location, representative, and territory columns across the admin inventory and sales rep pages.

## What changed
| File | Change |
|---|---|
| `src/features/products/product-dialog.tsx` | Conditionally omits empty subProducts array when editing an existing product, sanitizes empty string IDs to undefined for routing, rates, and charges, and preserves sub_product_id on rate items. |
| `src/features/products/types.ts` | Made subProducts optional on CompositeProductPayload so update payloads do not require sending sub-product arrays. |
| `src/routes/admin.inventory.tsx` | Centered Product / Spec and Location table columns using meta align center and replaced left margins with centered flex layouts. |
| `src/routes/admin.sales-reps.tsx` | Centered Representative Name and Assigned Territory table columns using meta align center and removed hardcoded left margins. |

## Why
When editing an existing composite product, the dialog previously submitted subProducts: []. This caused backend update handlers to delete existing sub-products attached to that product. Furthermore, empty strings for child entity IDs triggered validation errors, and rate sub-product bindings were stripped. In the admin tables, several column headers and cell values used manual left margins (ml-6, ml-12) that broke vertical alignment with their headers.

## How
**Safe product payload construction**: In ProductDialog, subProducts is only initialized as [] for newly created products. When editing (currentProduct is truthy), subProducts is omitted from the update payload.
**ID normalization and rate preservation**: Child items in routing, rates, and charges map id: item.id || undefined so empty strings are not passed to the API. subProductId: rt.sub_product_id || null preserves specific sub-product rate links.
**Payload type definition update**: Made subProducts?: Array<...> optional in CompositeProductPayload to align with the conditional payload.
**Table layout centering**: Added meta: { align: "center" } to column definitions and changed content wrappers to inline-flex centered containers in admin.inventory.tsx and admin.sales-reps.tsx.

## Decisions made
**Omitted subProducts key entirely on updates rather than passing null.** The backend interprets an omitted key as no change to sub-products, preventing unintentional cascade deletions.

## Notes
- Verified production build live with npm run build (vite build && tsc -b), passing with exit code 0.

## Test plan
**Automated tests (all passing):**
- [x] `npm run build`: Vite build and TypeScript compilation completed with exit code 0.

**Manual checks (local):**
- [x] Verified updating a product preserves existing sub-products and does not overwrite them with an empty list.
- [x] Verified inventory and sales reps admin tables display aligned headers and cell contents.

