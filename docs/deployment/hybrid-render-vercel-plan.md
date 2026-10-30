# Hybrid deployment plan (Vercel + Render)

Status: planning branch only. This file does not provision infrastructure or change production.

## Target topology

- Vercel: Twenty React/Vite frontend, built from the monorepo workspace.
- Render web service: Twenty API, built from the repository's server-only Docker target.
- Render background worker: same server image and runtime environment, command `yarn worker:prod`.
- Render PostgreSQL: persistent primary database.
- Render Key Value: Redis-compatible BullMQ/cache service; configure the no-eviction policy where supported.
- Object storage: S3-compatible storage for production attachments. Do not use ephemeral container filesystem or local storage for durable uploads.

## Repository facts verified

- Monorepo Dockerfile: `packages/twenty-docker/twenty/Dockerfile`.
- Docker build context must be repository root because the Dockerfile copies root workspace files.
- Server-only Docker build target: `twenty-server`.
- Full app target: `twenty`.
- The Dockerfile also defines a later `twenty-app-dev` stage; its final stage is not the production server. A platform build that simply builds the Dockerfile without selecting a target may therefore build the wrong image.
- Compose reference: `packages/twenty-docker/docker-compose.yml`.
- API health check in compose: `GET /healthz` on port 3000.
- API process: `node dist/main`.
- Worker process: `yarn worker:prod`.
- Worker must use the same database, Redis, encryption and storage configuration as the API. Run migrations only from the API deployment, not the worker.
- Frontend package: `packages/twenty-front`; build script is `yarn nx run twenty-front:build` (verify root Nx target before wiring CI).

## Critical build-target gate

Do not create Render services until the platform can reliably select the Docker build target `twenty-server`, or a dedicated, tested production Dockerfile has been added. The existing Dockerfile's last stage is `twenty-app-dev`, so an implicit default build is unsafe.

Preferred resolution: add a small dedicated production Dockerfile whose final stage is explicitly based on the existing `twenty-server` stage, or use a verified Render build configuration that selects `--target twenty-server`. Test the exact image locally/CI before connecting production services.

## Required configuration

API and worker need matching values:
- `PG_DATABASE_URL`: Render PostgreSQL internal connection string.
- `REDIS_URL`: Render Key Value internal Redis URL.
- `SERVER_URL`: public API origin (Render service URL or chosen API domain).
- `ENCRYPTION_KEY`: strong generated secret; preserve it across deployments and backups.
- `FALLBACK_ENCRYPTION_KEY`: only set for planned key rotation.
- `APP_SECRET`: legacy compatibility only if required by this exact Twenty version.
- `STORAGE_TYPE`: configure the supported S3 storage mode and credentials for production object storage.
- `DISABLE_DB_MIGRATIONS`: false/omitted for API migration owner; true for worker.
- `DISABLE_CRON_JOBS_REGISTRATION`: API owns registration; worker should not register duplicate cron jobs.
- Frontend build-time `REACT_APP_SERVER_BASE_URL`: public API origin.

Do not commit secrets. Use Render secret environment variables and Vercel environment variables. Never print secret values in logs or issue reports.

## Rollout sequence

1. Validate the dedicated server image and worker command.
2. Create Render PostgreSQL and Key Value; record internal URLs as secrets.
3. Create API web service and worker from the same immutable image/build revision.
4. Configure S3-compatible object storage and test upload/download before production.
5. Deploy API; verify `/healthz`, migrations, database connectivity and Redis.
6. Deploy worker; verify it starts and consumes a controlled test job.
7. Configure Vercel frontend build with the API origin; verify login and GraphQL/API calls.
8. Run end-to-end tests: signup/first workspace, login/logout, create/read/update records, file upload, background job, restart persistence, backup/restore.
9. Only after all checks pass, assign production domains and retire the broken Vercel monolith deployment.

## Rollback and data safety

- Keep the existing Vercel project and current production alias unchanged until acceptance tests pass.
- Take a PostgreSQL backup before migrations and verify a restore into a separate database.
- Keep encryption keys and object-storage data backed up; a database-only backup is insufficient.
- Pin deployments to a commit/image digest. Roll back application images independently of database schema only when migration compatibility is confirmed.

## Open validation items

- Confirm the exact root Nx target for the frontend build and all Vite environment-variable handling.
- Confirm this Twenty revision's required production env vars from its version-matched docs/config.
- Confirm Render Key Value connection URL format and supported eviction settings.
- Confirm storage provider/bucket and CORS policy.
- Confirm the selected Render plan's persistent disk, worker, database, region and cost limits.

No Render resources are created by this plan. Provisioning should follow only after the Docker target gate and required secrets/storage choices are resolved.
