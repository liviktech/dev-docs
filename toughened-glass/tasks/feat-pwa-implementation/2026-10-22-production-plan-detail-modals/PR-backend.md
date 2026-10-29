    # 2026-10-22 - production-plan-detail-modals (backend)

    ## Summary

    The app had no production plan, and the work order list had no way to show whether a work order was planned. This PR adds a Production Plan module (create, edit, submit, approve, reject, delete, list) with its own permissions, and adds the plan status to the work order list.

    The plan rules are enforced here, not only in the UI: only the person who raised a plan can change or delete it, and nobody can approve or reject their own plan. I checked live that `master.work_center` has no `id` column (its key is company and code) and built the plan around that. The module compiles, but **the endpoints were not called end to end** (see "Test plan").

    **Notifications are not included on purpose** - see "Next steps" below. This was a choice, not a miss.

    ## What changed

    | File | Change |
    |---|---|
    | `src/database/migrations/077_production_plan.sql` (new) | Adds `production.production_plan` (one per work order, with status and who submitted, approved or rejected it) and `production.production_plan_line`. |
    | `src/database/seeds/026_production_plan_rbac.sql` (new) | Adds a `production_plan` tab to the Work Order module with VIEW, ADD, EDIT and APPROVE rights. Planning Team gets view, add and edit. Production Manager gets view and approve. Customer Care gets view. Admin bypasses rights. |
    | `src/modules/production-plan/production-plan.types.ts` (new) | Row and API types. |
    | `src/modules/production-plan/production-plan.schema.ts` (new) | Zod schemas for the work order id, the plan body and the reject reason. |
    | `src/modules/production-plan/production-plan.repository.ts` (new) | Raw SQL: list, find, lock, replace lines, status updates, delete. |
    | `src/modules/production-plan/production-plan.service.ts` (new) | Business rules and transitions (see "How"). Has marked comments where notifications will be added later. |
    | `src/modules/production-plan/production-plan.controller.ts` (new) | Thin HTTP layer. |
    | `src/modules/production-plan/production-plan.routes.ts` (new) | `GET /production-plans` (list) and, under `/work-orders/:workOrderId/plan`, get, save, submit, approve, reject and delete. All behind `requirePermission('work_order', ..., 'production_plan')`. |
    | `src/api/route-mapper.ts` | Registers the two new routers. The plan router is mounted before the work order router. |
    | `src/modules/work-order/work-order.repository.ts` | The work order list also returns `plan_status` (a subselect, so the existing GROUP BY is untouched). |
    | `src/modules/work-order/work-order.types.ts` | `WorkOrderListItem` gets an optional `planStatus`. |

    ## Why

    Production planners and the Production Manager need to agree a plan per work order before production starts, and the Production Manager or Admin needs to approve it. The rules about who can change or approve a plan have to live on the server so they cannot be skipped from the UI.

    ## How

    **Status flow**: DRAFT, SUBMITTED, then APPROVED or REJECTED. A rejected plan can be edited and submitted again.

    **Ownership**: save, submit and delete are allowed only for the user who created the plan, and only while it is not approved. Anyone else gets a 403 (`PLAN_NOT_OWNER`). An approved plan cannot be changed or deleted (`PLAN_LOCKED`).

    **No self approval**: approve and reject refuse the user who created the plan (`PLAN_SELF_APPROVAL`), admins included. Only a SUBMITTED plan can be approved or rejected.

    **Locking**: every change locks the plan row (`FOR UPDATE`) inside a transaction. The first save creates the plan row with `INSERT ... ON CONFLICT DO NOTHING`, so two saves at once cannot create two plans.

    **Plan body**: a save replaces all lines in one go. The order of lines in the request is the sequence. The service also checks that an end date is not before the start date.

    **Empty plan**: `GET` for a work order with no plan returns an empty DRAFT so the UI can start editing without a separate create call.

    **Work centre**: stored as `work_center_code` and joined to `master.work_center` by company and code for the name. The API still calls it `workCenterId` (it carries the code), matching the work centre API.

    ## Decisions made

    **No foreign key on work centre.** `master.work_center` has no `id` column and its primary key is `(company_id, code)`. The first version of the migration used an `id` foreign key and failed (`column "id" referenced in foreign key constraint does not exist`). The line table has no `company_id`, so I stored the code without a foreign key, like other tables here do.

    **Approval is a tab right.** I used `work_order_production_plan_approve` rather than the module level `work_order_approve`, so approving plans can be granted separately from approving work orders.

    **Pending is a UI idea.** The list endpoint returns all plans and the frontend splits them into Pending, Approved and All. I did not add a status filter to the API.

    **Plan approval does not change the work order status.**

    ## Notes

    - `GET /production-plans` returns only plans that exist. A work order with no plan is not in the list.
    - Run `npm run migrate` and then `npm run seed` to get the new tables and rights.
    - I checked live in the dev database that `master.work_center` has no `id` column, and that the Order Charges master has Admin Charge and Wooden Box Packing but no Insurance entry.

    ## Next steps

    - Notifications: tell the Production Manager and Admin when a plan is submitted, tell Executive and Customer Care when it is approved, and tell the planner when it is rejected. Comments in `production-plan.service.ts` mark the spots.
    - Decide whether an approved plan should gate or start work order production.

    ## Test plan

    **Automated tests (no new tests added):**
    - [x] `npx tsc --noEmit -p .`: passes with no errors.
    - [ ] No test files exist for this module, none were added.

    **Manual checks (local):**
    - [x] Looked up `master.work_center` in the dev database to confirm its columns and primary key, then fixed the migration to match.
    - [ ] Running `077_production_plan.sql` and `026_production_plan_rbac.sql` after the fix - **not re-run by me**. The first run failed on the work centre foreign key, and the migration was changed after that.
    - [ ] Calling each endpoint (save, submit, approve, reject, delete, list) as planner, manager and admin - **not checked live**. Only the code and types were checked.
    - [ ] Self approval and non owner edit returning 403 - **not checked live**.
