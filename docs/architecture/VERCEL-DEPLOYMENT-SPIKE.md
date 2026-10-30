# Vercel deployment spike — execution record

**Planning branch:** `architecture/foundation-plan`  
**Runtime wiring:** draft PR [#1](https://github.com/rebshogan/twenty-crm-reb/pull/1), branch `deployment/vercel-runtime`  
**Environment:** preview / synthetic data only  
**Status:** Production image build verified; HTTP + worker integration and live runtime checks still pending.

## Verified so far

- [x] Git-connected Vercel project exists and is linked to `rebshogan/twenty-crm-reb`.
- [x] Initial project alias was inspected in a live browser and returned Vercel `404 NOT_FOUND`. That was a generic Git deployment, not a running Twenty app.
- [x] The upstream Dockerfile's last stage was identified as the development all-in-one image. The planning branch now appends an explicit `twenty-vercel` stage based on the production `twenty` server-plus-frontend stage; the development image remains selectable explicitly.
- [x] GitHub Actions run [41032093355](https://github.com/rebshogan/twenty-crm-reb/actions/runs/41032093355) completed successfully at commit `ce3cb9f556f7c992cc7814578a13ba3e4e43d3d4`.
  - Built the default production image without pushing it.
  - Verified `NODE_ENV=production`, `NODE_PORT=3000`, working directory `/app/packages/twenty-server`, `/app/entrypoint.sh`, Node 26.12.1 and Yarn 4.18.1.
  - Reported image size: 242,689,703 bytes (approximately 231.5 MiB).
- [x] A temporary Compose smoke stack was added for PostgreSQL 16, Redis 7, Twenty server and worker. It uses throwaway credentials and disposable volumes only.
- [ ] The newer workflow run [41034297293](https://github.com/rebshogan/twenty-crm-reb/actions/runs/41034297293) is still in progress. It includes the HTTP `/healthz`, frontend HTML and worker-process smoke checks; those checks have not yet been reported as passing.
- [x] Draft PR [#1](https://github.com/rebshogan/twenty-crm-reb/pull/1) adds Vercel wiring:
  - Root `Dockerfile.vercel` symlink preserves repository-root Docker build context.
  - `vercel.json` defines the Twenty container service and a BullMQ worker Daemon.
  - The worker is deliberately configured with zero instances, and no public rewrite is present. This prevents an unconfigured service from being exposed or starting against nonexistent backing stores.
- [x] Vercel build [7ei3aGvcWpaYzAdBN9P3vQEnDg57](https://vercel.com/rebshogan-3286/twenty-crm-reb/7ei3aGvcWpaYzAdBN9P3vQEnDg57) reached the container-image build and reported `built in 1294.1s`; the image was then pushed to the Vercel Container Registry.
- [ ] That Vercel deployment was still in `BUILDING` state at the last check; deployment readiness and runtime behaviour are not yet verified.

## Findings and remaining blockers

1. **The original 404 was a configuration problem.** The first Git deployment had no container/service config and did not start Twenty.
2. **The Vercel builder initially used the Dockerfile directory as its context.** This failed because the Dockerfile's COPY instructions expect monorepo-root files such as `package.json`. The root symlink is the current fix; the latest build has passed this stage and built the image.
3. **No public route is intentionally configured yet.** Services are private until a top-level rewrite exposes them. Do not add a public rewrite until server environment variables and backing services exist.
4. **No worker is running yet.** The Daemon is configured with `iad1: 0` intentionally; the process command is `yarn worker:prod`.
5. **Migrations need explicit ownership.** Twenty's server entrypoint runs database setup/migrations and cron registration unless disabled. Before scaling HTTP instances, set `DISABLE_DB_MIGRATIONS=true` on the service and arrange one controlled migration task. The worker command overrides the image entrypoint, so it does not run the server entrypoint migration routine.
6. **No durable backing services have been provisioned.** PostgreSQL, Redis and S3-compatible storage are not configured. No external database, Redis, object bucket, or production secrets have been created.
7. **The connected Vercel team is on Hobby.** Keep this synthetic-only. Do not import real business contacts/interviews or use it for commercial operations until the plan and data-handling terms are suitable.
8. **Cost still needs review.** The image has been pushed to Vercel Container Registry. Confirm registry/service/daemon usage pricing and plan eligibility before enabling a public service or nonzero worker count.

## Next test sequence

- [ ] Confirm the latest GitHub smoke workflow completes successfully; review server and worker logs if it fails.
- [ ] Confirm Vercel deployment `7ei3aGvcWpaYzAdBN9P3vQEnDg57` reaches a terminal successful state.
- [ ] Verify the image's effective entrypoint, command, working directory and exposed port in Vercel's build/runtime output.
- [ ] Confirm the team can use Vercel Services and Daemons on its current plan.
- [ ] Decide whether to provision managed PostgreSQL, Redis and S3-compatible storage after reviewing their costs. Do not create billable stores without approval.
- [ ] Configure environment variables in Vercel's encrypted environment-variable store; never commit values.
- [ ] Test login/session cookies, REST, GraphQL, frontend assets, realtime transport, uploads and file persistence with synthetic data.
- [ ] Test worker success, retry/backoff, duplicate delivery, SIGTERM and restart recovery before setting daemon instances above zero.
- [ ] Test a controlled migration and backup restore; isolate preview data from production.
- [ ] Add a public rewrite only after the application and its backing services are configured and verified.

## Acceptance table

| Test | Result | Evidence |
|---|---|---|
| Vercel Git link | Pass | Project settings |
| Original generic deployment route | Fail as expected | Live browser: 404 NOT_FOUND |
| Production Docker image build | Pass | GitHub Actions run 41032093355 |
| Production image size/runtime metadata | Pass | CI image inspect; 242,689,703 bytes |
| Vercel container image build | Built; deployment pending | Vercel build logs: 1294.1s, image push started/completed |
| HTTP health + frontend | Pending | GitHub smoke run 41034297293 |
| Worker process / queue behaviour | Pending | GitHub smoke run 41034297293 |
| Realtime transport | Not run | Requires running app |
| Managed PostgreSQL / Redis | Not provisioned | No external stores created |
| S3 upload persistence | Not run | No bucket created |
| Backup restore | Not run | No database created |
| Commercial data suitability | Blocked | Team plan is Hobby |

