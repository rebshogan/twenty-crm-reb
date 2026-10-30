# Hybrid deployment plan (Vercel + Render)

Status: implementation in progress on `infra/render-hybrid-plan`. No Render infrastructure has been provisioned and no production domain has been changed.

## Target topology

- Vercel: Twenty React/Vite frontend, rooted at `packages/twenty-front`.
- Render web service: Twenty API, built from `packages/twenty-docker/twenty/Dockerfile.render` with repository-root context and explicit `twenty-server-production` target.
- Render background worker: same server image and runtime environment, command `yarn worker:prod`.
- Render PostgreSQL: persistent primary database.
- Render Key Value: Redis-compatible BullMQ/cache service; use `noeviction` where supported.
- Object storage: S3-compatible storage for production attachments. Do not use an ephemeral container filesystem or local storage for durable uploads.

## Repository facts verified

- Upstream monorepo Dockerfile: `packages/twenty-docker/twenty/Dockerfile`.
- Docker build context must be repository root because the build copies root workspace files.
- The upstream Dockerfile contains multiple stages and ends in a development stage; an implicit default build is unsafe for a server deployment.
- Dedicated production-only Dockerfile: `packages/twenty-docker/twenty/Dockerfile.render`.
- Dedicated image target: `twenty-server-production`.
- API process: `node dist/main`.
- Worker process: `yarn worker:prod`, defined by the `twenty-server` workspace package.
- API health check in the upstream compose file: `GET /healthz` on port 3000.
- The Docker entrypoint runs database setup/migrations and cron registration unless disabled.
- The compose reference explicitly disables both migrations and cron registration on the worker. The API remains the single migration and cron-registration owner.
- Frontend package: `packages/twenty-front`; Nx target: `twenty-front:build`.
- Vite uses `REACT_APP_SERVER_BASE_URL` for the development API proxy and exposes `REACT_APP_` variables to the client build.

## CI/CD implementation

- `.github/workflows/validate-render-image.yml` builds the explicit production image on relevant pull-request changes, pushes to the implementation branch, and manual dispatch.
- The workflow uses read-only repository permissions, disables checkout credential persistence, caches Docker layers, and verifies the API and worker runtime outputs.
- The workflow has been committed, but its latest run must complete successfully before the image is considered validated. A committed workflow is not proof of a successful build.
- Vercel Git integration creates preview deployments for branch pushes. The Vercel project is configured with framework `vite`, root directory `packages/twenty-front`, source files outside the root enabled, the repository-pinned Yarn install/build commands, and output directory `build`.
- The latest Vercel preview build is still in progress. It is not accepted until build completion and browser/API smoke tests.
- Do not merge this PR or promote a production deployment while required checks are pending or failing.

## Required configuration

API and worker need matching values:

- `PG_DATABASE_URL`: Render PostgreSQL internal connection string.
- `REDIS_URL`: Render Key Value internal Redis URL.
- `SERVER_URL`: public API origin (Render service URL or chosen API domain).
- `ENCRYPTION_KEY`: strong generated secret; preserve it across deployments and backups.
- `FALLBACK_ENCRYPTION_KEY`: only set for a planned key rotation.
- `APP_SECRET`: present in the version's local env template; confirm exact runtime usage before deciding whether to set it.
- `STORAGE_TYPE` and the version-supported S3 settings: use durable object storage for production uploads.
- `DISABLE_DB_MIGRATIONS`: leave false/omitted on API; set true on worker.
- `DISABLE_CRON_JOBS_REGISTRATION`: leave false/omitted on API; set true on worker.
- Frontend build-time `REACT_APP_SERVER_BASE_URL`: public API origin. Set this only after the API service URL is known.

Do not commit secrets. Use Render secret environment variables and Vercel environment variables. Never print secret values in logs or issue reports.

## Rollout sequence

1. Pass the production Docker image build and image-content checks in GitHub Actions.
2. Confirm the current Render plan, persistence, worker, database, region and total recurring cost. Do not create paid resources without explicit approval.
3. Create Render PostgreSQL and Key Value; record internal URLs as secrets.
4. Create API web service and worker from the same source revision and Dockerfile target.
5. Configure S3-compatible object storage and test upload/download before production.
6. Deploy API; verify `/healthz`, migrations, database connectivity and Redis.
7. Deploy worker with migrations and cron registration disabled; verify it starts and consumes a controlled test job.
8. Set Vercel's `REACT_APP_SERVER_BASE_URL` to the API origin; verify login and GraphQL/API calls.
9. Run end-to-end tests: signup/first workspace, login/logout, create/read/update records, file upload, background job, restart persistence, backup/restore.
10. Only after all checks pass, assign production domains and retire the broken Vercel monolith deployment.

## Rollback and data safety

- Keep existing production aliases unchanged until acceptance tests pass.
- Take a PostgreSQL backup before migrations and verify a restore into a separate database.
- Keep encryption keys and object-storage data backed up; a database-only backup is insufficient.
- Pin deployments to a commit/image digest. Roll back application images independently of database schema only when migration compatibility is confirmed.

## Open validation items

- Complete the GitHub Actions image build and verify its API/worker output checks.
- Complete the Vercel frontend preview build; verify the output serves the SPA and API URL configuration is correct.
- Confirm this exact Twenty revision's runtime-required environment variables, especially encryption and S3 configuration.
- Confirm Render Key Value connection URL format and supported eviction settings.
- Select the object-storage provider/bucket and configure CORS.
- Confirm the selected Render plan's persistent disk, worker, database, region and recurring cost.

No Render resources have been created. Production rollout remains gated on successful CI, preview and integration checks.
