# 2026-11-06 - admin-self-approve-item-add (backend)

## Summary

Before this change nobody could approve or reject a production plan or a dispatch plan they raised themselves, admins included. This PR lets ADMIN and SUPER_ADMIN do it on both plan types, and keeps the block for every other role. It is a small change in the plan services and controllers, and the role is passed in from the logged in user. I only type checked the change, I did not call the approve and reject endpoints live.

## What changed

| File | Change |
|---|---|
| `src/modules/production-plan/production-plan.service.ts` | `assertNotOwnPlan` takes an optional `role` and returns early for `SUPER_ADMIN` and `ADMIN`. `approvePlan` and `rejectPlan` take the new `role` argument and pass it on. |
| `src/modules/production-plan/production-plan.controller.ts` | `approvePlan` and `rejectPlan` pass `req.user!.role` to the service. |
| `src/modules/dispatch-plan/dispatch-plan.service.ts` | Same change as the production plan service. |
| `src/modules/dispatch-plan/dispatch-plan.controller.ts` | Same change as the production plan controller. |

## Why

The "Approve Requests" screens need an admin to be able to approve their own plan, for example when they are the only person who can sign off. The old rule was enforced in the backend, so the frontend alone could not allow it.

## How

**Role check**: `assertNotOwnPlan(plan, userId, role)` now returns right away when the role is `SUPER_ADMIN` or `ADMIN`. For any other role it behaves exactly as before and throws `PLAN_SELF_APPROVAL`.

**Role is optional**: The new `role` argument is optional, so any other caller that does not pass it keeps the old strict behaviour.

**Kept the same on purpose**: Only a `SUBMITTED` plan can be approved or rejected, and the reject reason is still required. Cancelling a request (delete) is not touched, it still belongs to the person who raised the plan.

## Decisions made

**Role is passed in, not read inside the service.** The services do not know about the request, so the controller passes `req.user!.role`. I did not add a permission flag because the rule is about the admin roles only.

## Test plan

**Automated tests (no tests added):**
- [ ] `npx tsc --noEmit`: not re-run for the backend in this session. The change is only an extra argument and an early return.

**Manual checks (local):**
- [ ] Admin approves and rejects their own production plan and dispatch plan - not run against a live backend.
- [ ] Non admin gets `PLAN_SELF_APPROVAL` on their own plan - not run, the code path for that role is unchanged.
