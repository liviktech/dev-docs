# 2026-10-01 - user-company-id-fix

## Summary

Creating a new user was throwing a 500 Internal Server Error because the `INSERT` into `core.app_user` did not include `company_id`, which is a `NOT NULL` column in the database. The user module was the only module missing this. Every other module (enquiry, order booking, work order, proforma invoice) already follows the pattern of reading `req.user.companyId` from the JWT session and passing it down through the service to the repository. This PR applies that same pattern to the user creation flow in three files.

## What changed

### rubine-toughened-glass-backend
| File | Change |
|---|---|
| `src/modules/master/user/user.controller.ts` | Reads `req.user!.companyId` from the JWT session and passes it to the service on user creation. |
| `src/modules/master/user/user.service.ts` | Accepts `companyId` as the first argument to `createUser` and passes it down to the repository. |
| `src/modules/master/user/user.repository.ts` | Adds `company_id` to the `INSERT INTO core.app_user` column list, updates `VALUES` placeholders from `$1-$6` to `$1-$7`, and passes `companyId` as `$1`. |

## Why

`core.app_user.company_id` is `NOT NULL` in the database schema (multi-tenant design). The column was never included in the `INSERT` query, so every attempt to create a user hit a PostgreSQL `23502` (not-null violation), which was not caught by the existing error handlers (they only catch `23505` unique and `23503` foreign key violations), so it surfaced as an unhandled 500.

## How

**Pattern already in use**: Every other module (enquiry, order booking, work order, proforma invoice) reads `companyId` from `req.user!.companyId` in the controller and threads it through the service and into the repository query. The user module simply had not done this.

**Three-file plumbing fix**: Controller extracts `companyId`, passes it to service, service passes it to repository, repository adds it as `$1` in the `INSERT` statement. No schema changes, no new infrastructure.

**Company row already exists**: The seed `001_company.sql` inserts the company row on first setup. The `company_id` is already in every active user's JWT via the auth middleware, so it is always available on authenticated requests.

## Decisions made

**Did not add `companyId` to the create user request body schema.** The company a new user belongs to is always the company of the person creating them (taken from their JWT session), not a value the caller supplies. Accepting it in the request body would be a security risk in a multi-tenant system.


## Notes

- Verified the enquiry module already has full company-level isolation across list, get, create, update, approve, and status-change operations. No changes needed there.
- Checked build with `npx tsc --noEmit`, exit code 0, zero TypeScript errors.

## Next steps

- The same `company_id` threading should be verified for any other modules that do INSERT operations to company-scoped tables (parties, products, etc.) to make sure none have the same miss.

## Test plan

**Automated tests (1 passing):**
- `npx tsc --noEmit`: Type check completed with exit code 0.

**Manual checks (local):**
- Logged in as an admin user and created a new user through the UI. Previously returned 500. After this fix the user is created and the response returns the new user record with status 201.
- Could not verify cross-company isolation in the current single-company setup. The schema and query are correct (company_id is threaded from JWT and used in every WHERE clause), but live cross-tenant testing requires a second company to be seeded.
