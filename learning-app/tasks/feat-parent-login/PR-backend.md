# 2026-11-24 - docker-deploy (backend)

## Summary

The backend had no way to be packaged and run on a server. This PR adds a Dockerfile, a compose file with PostgreSQL and migrations, and a `.dockerignore`. No application code changed. The image builds and starts, and the app stops with a clear config error when `DATABASE_URL` is missing. The full compose stack was not run against a database.

## What changed

| File | Change |
|---|---|
| `Dockerfile` (new) | Multi-stage build on `node:22-slim`. Build stage compiles TypeScript, a deps stage installs production packages only, and the runtime stage copies `dist/`, the SQL migrations and seeds, and runs as the non-root `node` user. Health check calls `/api/v1/health/live`. |
| `docker-compose.yml` (new) | `db` (PostgreSQL 17 with a volume), `migrate` (one-shot `npm run migrate:up`), and `api` (starts after migrations finish). `DATABASE_URL` is built from `POSTGRES_PASSWORD`. |
| `.dockerignore` (new) | Keeps `.env`, `node_modules`, tests, docs and `.git` out of the image. |

## Why

The API needs to be deployed on a server. Migrations are SQL files run by `node-pg-migrate`, so the image has to carry them and the stack needs a step that runs them before the API starts.

## How

**Debian base, not Alpine**: argon2 is a native module, and the Debian image is the safer choice. argon2 loads inside the built image.

**No secrets in the image**: `.env` is excluded and read by compose through `env_file` at run time.

**Migrations before start**: the `api` service waits for `migrate` to finish successfully, which waits for the database health check.

## Notes

- Set `NODE_ENV=production`, the JWT and OTP secrets, SMTP, `GOOGLE_WEB_CLIENT_ID` and https `CORS_ALLOWED_ORIGINS` in `.env`, plus `POSTGRES_PASSWORD`.
- Set `TRUST_PROXY_HOPS` to the number of reverse proxies in front of the app. The new rate limits count per client IP, so without it every user shares the proxy's IP.
- Put HTTPS in front of port 3000. To use a managed database, remove the `db` service and set `DATABASE_URL` directly.
- The README still says "No Docker is used or required". It was not changed in this PR.

## Next steps

- Update the README with a short deploy section.

## Test plan

**Automated tests (none added):**
- [ ] No unit or integration tests, since there is no application code change.

**Manual checks (local):**
- [x] `docker build`: the image builds.
- [x] `require('argon2')` inside the image: loads.
- [x] `dist/main.js` and `src/database/migrations` are present in the image.
- [x] Start without environment variables: exits with "Invalid environment configuration: DATABASE_URL".
- [ ] `docker compose up` with a real `.env` and database - not run. Migrations, the health check and a live request are not checked.
