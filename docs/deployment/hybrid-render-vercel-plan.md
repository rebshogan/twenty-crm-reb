# Hybrid deployment plan (Vercel + Render)

Status: implementation in progress on `infra/render-hybrid-plan`. No Render infrastructure has been provisioned and no production domain has been changed.

## Target topology

- Vercel: Twenty React/Vite frontend, rooted at `packages/twenty-front`.
- Render web service: Twenty API, built from `packages/twenty-docker/twenty/Dockerfile.render` with repository-root context and explicit `twenty-server-production` target.
- Render background worker: same server image and runtime environment, command `yarn worker:prod`.
- Render PostgreSQL: persistent primary database.
- Render Key Value: Redis-compatible BullMQ/cache service; use `noeviction` where supported.
- Object storage: S3-compatible storage for production attachments. Cloudflare R2 is the leading candidate because its S3 API is compatible and standard storage currently includes 10 GB-month, 1 million Class A operations and 10 million Class B operations monthly, with no egress charge. Actual usage beyond the free allowance is billed.

## Repository and runtime facts verified

- Upstream monorepo Dockerfile: `packages/twenty-docker/twenty/Dockerfile`.
- Docker build context must be repository root because the build copies root workspace files.
- The upstream Dockerfile contains multiple stages and ends in a development stage; an implicit default build is unsafe for a server deployment.
- Dedicated production-only Dockerfile: `packages/twenty-docker/twenty/Dockerfile.render`.
- Dedicated image target: `twenty-server-production`.
- API process: `node dist/main`.
- Worker process: `yarn worker:prod`, defined by the `twenty-server` workspace package.
- API health check: `GET /healthz` on port 3000.
- The Docker entrypoint runs database setup/migrations and cron registration unless disabled.
- The worker must set `DISABLE_DB_MIGRATIONS=true` and `DISABLE_CRON_JOBS_REGISTRATION=true`; the API remains the single migration and cron-registration owner.
- Frontend package: `packages/twenty-front`; Nx target: `twenty-front:build`.
- Frontend build-time API setting: `REACT_APP_SERVER_BASE_URL`.
- Server environment template: `packages/twenty-server/.env.example`; it actively defines `PG_DATABASE_URL`, `REDIS_URL`, `APP_SECRET`, and `FRONTEND_URL`.
- New installs should set a dedicated `ENCRYPTION_KEY`. Current Twenty documentation confirms legacy fallback to `APP_SECRET`, but that fallback is not the recommended configuration for a new installation. Losing the encryption key can make stored credentials and secrets unrecoverable.
- Production uploads must use `STORAGE_TYPE=S_3` with `STORAGE_S3_REGION`, `STORAGE_S3_NAME`, and the provider credentials/endpoint as required. Do not rely on the container's local filesystem for durable files.

## CI/CD implementation and validation

- `.github/workflows/validate-render-image.yml` builds the explicit production image on relevant pull-request changes, pushes to the implementation branch, and manual dispatch.
- The workflow uses read-only repository permissions, disables checkout credential persistence, caches Docker layers, and verifies the API and worker runtime outputs.
- **Verified success:** GitHub Actions run [41072695451](https://github.com/rebshogan/twenty-crm-reb/actions/runs/41072695451) completed successfully. Both the production image build and API/worker runtime-output validation steps passed.
- Vercel preview deployment reached READY for the implementation branch. This confirms deployment readiness/build completion, not CRM end-to-end functionality. The preview is protected and has not been accepted as an unauthenticated browser/API smoke test.
- The optional upstream preview-environment dispatcher expects GitHub App credentials that forks do not have. The workflow has been guarded on both the PR base and implementation branches so this optional integration is skipped when those upstream credentials are absent. The historical failure predates the base-branch fix; verify a fresh event before considering this resolved operationally.
- No Render database, Key Value, API or worker services have been provisioned.
- No production domain has been changed or promoted.

## Required configuration

API and worker need matching infrastructure values unless noted:

- `PG_DATABASE_URL`: Render PostgreSQL internal connection string.
- `REDIS_URL`: Render Key Value internal Redis URL.
- `SERVER_URL`: public API origin (Render service URL or chosen API domain).
- `FRONTEND_URL`: public Vercel frontend origin.
- `ENCRYPTION_KEY`: independently generated, high-entropy secret; preserve it across deployments and backups.
- `APP_SECRET`: set to a separate strong random secret; do not reuse the encryption key.
- `FALLBACK_ENCRYPTION_KEY`: only set during a planned encryption-key rotation.
- `STORAGE_TYPE=S_3` and supported `STORAGE_S3_*` settings: use durable object storage for production uploads.
- `DISABLE_DB_MIGRATIONS`: leave false/omitted on API; set true on worker.
- `DISABLE_CRON_JOBS_REGISTRATION`: leave false/omitted on API; set true on worker.
- Frontend build-time `REACT_APP_SERVER_BASE_URL`: public API origin. Set this only after the API service URL is known.

Do not commit secrets. Use Render secret environment variables and Vercel environment variables. Never print secret values in logs or issue reports.

## Cost estimate (Render Hobby workspace, USD)

Using current published Render compute prices, a conservative small production starting point is:

| Resource | Suggested plan | Monthly estimate |
| --- | --- | ---: |
| API web service | 1 CPU / 2 GB RAM (`1c-2g`) | $25 |
| Background worker | 1 CPU / 2 GB RAM (`1c-2g`) | $25 |
| PostgreSQL | 0.5 CPU / 1 GB RAM (`0.5c-1g`) | $19 |
| Key Value | 256 MB (`256mb`) | $10 |
| **Estimated Render subtotal** | | **$79/month** |

This is a planning estimate, not a bill quote. It excludes taxes, usage beyond included workspace bandwidth/build allowances, optional extra database storage, and any external services. Render currently includes 1 GB of Postgres storage; additional storage is billed separately. Cloudflare R2 Standard currently includes 10 GB-month storage, 1 million Class A operations and 10 million Class B operations per month, with no egress charge; usage beyond those allowances is billed. The Render Hobby workspace itself has no monthly workspace fee. Do not create any of these resources until the user explicitly approves the estimated recurring spend.

A lower-cost worker configuration (512 MB RAM) would reduce the Render subtotal by $18 to about $61/month, but I do not recommend starting there without observing the worker's real memory use. Render's free database expires after 30 days and free Key Value is non-persistent; those are unsuitable for production data.

## Rollout sequence

1. Pass the production Docker image build and image-content checks in GitHub Actions. **Passed.**
2. Confirm Render plans, persistence, worker, database, region and total recurring cost. Do not create paid resources without explicit approval.
3. Create Render PostgreSQL and Key Value; record internal URLs as secrets.
4. Create API web service and worker from the same source revision and Dockerfile target.
5. Create/configure the S3-compatible bucket and CORS policy; test upload/download before production.
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

- Verify a fresh PR event uses the updated base-branch preview-dispatch guard; old failed runs are historical.
- Complete an authenticated browser/API smoke test against the Vercel preview.
- Confirm Render Key Value connection URL format and supported eviction settings for the selected plan.
- Confirm exact Render service and Postgres plan prices, storage/network allowances, persistence and region before requesting approval.
- Select the final object-storage provider/bucket and configure CORS.
- Confirm the selected Render plan's persistent disk, worker, database, region and recurring cost.

No Render resources have been created. Production rollout remains gated on backend provisioning approval, authenticated integration tests and rollback readiness.
