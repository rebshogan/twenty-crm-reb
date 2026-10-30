# Vercel deployment spike — execution record

**Branch:** `architecture/foundation-plan`  
**Environment:** preview / synthetic data only  
**Status:** In progress — no application runtime test has passed yet

## Verified so far

- [x] Git-connected Vercel project exists and is linked to `rebshogan/twenty-crm-reb`.
- [x] A Git deployment was built and reached Vercel's `READY` state.
- [x] The public project alias was inspected in a live browser.
- [x] The alias returns Vercel `404 NOT_FOUND`, not a Twenty login or application.
- [x] The inspected deployment was created from branch `architecture/foundation-plan`, commit `6b75d90c23e3d163e71a3aa403c2443a2df7427e`.
- [x] Its build log reports a short generic Vercel build and no container/server startup. The deployment is a platform build artifact, not proof of a working CRM.
- [x] Project settings report `framework: null`; the repository root has no `vercel.json` or `vercel.toml`.
- [x] The upstream Dockerfile was inspected. It has named `twenty-server`, `twenty`, and `twenty-app-dev` stages. The last/default stage is the development all-in-one image.
- [x] The official Compose file confirms separate `server` and `worker` processes, plus external Postgres and Redis dependencies.
- [x] The server package defines `worker:prod` as `node dist/queue-worker/queue-worker`.
- [ ] Container image successfully built.
- [ ] HTTP server started and health/login/API verified.
- [ ] Realtime transport verified.
- [ ] Worker/queue retry/restart tests passed.
- [ ] Managed Postgres/Redis and S3 persistence verified.
- [ ] Backup/restore verified.
- [ ] Cost and commercial-plan gate resolved.

## Findings and immediate blockers

1. **The current 404 is expected from the current repository/project configuration.** The project has no Vercel container/service deployment configuration. A successful Vercel Git build did not package and launch Twenty's production Docker image.
2. **The Dockerfile default target must not be used blindly.** Its last stage is `twenty-app-dev`, which bundles local stateful dependencies and development settings. Production requires explicit `twenty` (server plus frontend) for HTTP.
3. **The worker needs its own process configuration.** It should use the same immutable production image and run `yarn worker:prod`, with migrations and cron registration disabled in the worker environment because the server/release task owns those duties.
4. **No durable application state may live in a Vercel container.** PostgreSQL, Redis and S3-compatible storage must be separate services.
5. **Sandbox creation was rejected by the connected Vercel API with HTTP 400.** No sandbox was provisioned; the exact API rejection did not expose a diagnostic reason in the v4 attempt. A v3 retry also returned 400 after the required project ID was supplied. No infrastructure resource was created by those attempts.
6. **Repository clone from the execution container was unavailable because outbound DNS/network access to GitHub failed.** This prevents a local Docker build in this execution environment. It is not a failure of the Dockerfile itself.

## Next test sequence

- [ ] Confirm the team has access to Vercel Services and Daemons on its current plan; do not assume beta availability from public documentation alone.
- [ ] Use a runner with GitHub access and Docker/BuildKit to build `packages/twenty-docker/twenty/Dockerfile` from repository root with `--target twenty`.
- [ ] Build the same immutable image for the worker and override its command to `yarn worker:prod`.
- [ ] Do not write a Vercel config until the actual Service/Daemon config schema and Dockerfile build context/target behaviour are proven. In particular, the documented Service schema exposes an entrypoint path but does not show a Docker build-target option.
- [ ] Do not push production images, create paid managed stores, or deploy a live worker until the plan/price and runtime tests are reviewed.

## Acceptance table

| Test | Result | Evidence |
|---|---|---|
| Vercel project Git link | Pass | Project settings |
| Current app route | Fail | Live browser: 404 NOT_FOUND |
| Correct production Docker target identified | Pass | Dockerfile target declarations |
| HTTP runtime | Not run | No image built |
| Worker runtime | Not run | No image built |
| PostgreSQL / Redis | Not provisioned | No external stores created |
| S3 uploads | Not run | No bucket created |
| Backup restore | Not run | No database created |
| Commercial data suitability | Blocked | Team plan is Hobby |

