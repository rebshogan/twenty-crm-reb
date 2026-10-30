# ADR-0001: Vercel-first runtime for the Twenty fork

- **Status:** Proposed; gated by a deployment spike
- **Date:** 2026-10-30

## Context

Twenty is an Nx/Yarn monorepo. Its production topology is an HTTP server with bundled frontend, a separate BullMQ worker, PostgreSQL, Redis and persistent file storage. The repository's official multi-stage Dockerfile exposes an explicit `twenty` production target (server plus frontend) and a separate `twenty-server` target. The last/default stage is `twenty-app-dev`, which bundles local Postgres, Redis and ClickHouse and is not suitable for production.

Vercel documents container-backed Services and long-running Daemons. Daemons entered public beta on 15 October 2026 and explicitly support BullMQ consumer workloads. This makes Vercel-first worth testing, but it does not prove Twenty compatibility.

## Decision

Run a synthetic-data deployment spike before application customisation:

1. HTTP application: use the official Dockerfile's explicit `twenty` production target and keep server/frontend on one public origin initially.
2. Worker: run the same immutable image/build with `yarn worker:prod` as a Vercel Daemon, if enabled for this team.
3. PostgreSQL and Redis: use managed external services, never container-local databases.
4. Files: use S3-compatible object storage; do not rely on ephemeral disk.
5. Migrations: run once as a controlled release task, not concurrently on scaled server instances.
6. Keep official Docker Compose as the local development and integration-test baseline.

## Required proof

- Build the production image and record build time, image size and peak memory.
- Verify HTTP port, health, frontend, login/session cookies, REST, GraphQL and the pinned version's realtime transport.
- Verify S3 upload/download, access control and persistence after instance replacement.
- Verify database pooling/reconnects and Redis reconnects with BullMQ load.
- Verify worker completion, retry/backoff, duplicate delivery, SIGTERM and restart recovery.
- Verify migrations are single-run and preview data stores are isolated from production.
- Perform a backup restore into a separate environment.
- Measure cold starts, latency, worker stability and total monthly cost.

## Consequences and gates

This preserves the literal Twenty fork and the requested Vercel-first direction, but relies on evolving platform features. The team currently reports Vercel Hobby, which is restricted to personal/non-commercial use. Use synthetic data only until a suitable commercial plan and data-handling terms are in place. Do not import real business contacts or interviews to Hobby.

If realtime, worker lifecycle, persistence, networking or cost fails acceptance, retain Vercel for compatible components and move only the incompatible persistent runtime to a container host. This is a contingency, not the default.

## Alternatives rejected

- Static frontend-only deployment: cannot run Twenty's server or worker.
- Docker Compose unchanged: Vercel does not deploy Compose as a stack.
- Replacing Twenty with a new Next.js CRM: violates the literal-fork requirement.
- Moving everything off Vercel immediately: premature before the spike.

## References

- https://vercel.com/docs/services
- https://vercel.com/docs/daemons
- https://vercel.com/changelog/vercel-daemons-is-now-in-public-beta
- https://vercel.com/docs/functions/runtimes/container
- https://vercel.com/legal/terms
- https://github.com/twentyhq/twenty
