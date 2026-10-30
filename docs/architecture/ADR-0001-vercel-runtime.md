# ADR-0001: Vercel-first runtime for the Twenty fork

- **Status:** Proposed; gated by a deployment spike
- **Date:** 2026-10-30
- **Scope:** Hosting architecture for the Malawi Market Intelligence fork

## Context

Twenty is an Nx/Yarn monorepo with a NestJS HTTP server, a React/Vite frontend served by the application, a BullMQ worker, PostgreSQL, Redis, and file storage. Its official Compose stack runs server and worker as separate processes. The production Dockerfile provides explicit `twenty-server` and `twenty` targets; `twenty` adds the compiled frontend to the server image. The final Dockerfile stage is `twenty-app-dev`, which bundles local Postgres, Redis and ClickHouse and is not the production target.

Vercel now documents container-backed HTTP services and Daemons for long-running pull-based workers. Daemons entered public beta on 15 October 2026 and explicitly include BullMQ as a supported use case. This makes a Vercel-first spike technically plausible, but it does not prove Twenty compatibility.

## Decision

Test this topology with synthetic data before customising the CRM:

1. **HTTP application:** build the repository's official Dockerfile using the explicit `twenty` target. Run the existing server/frontend bundle as one public HTTP origin first.
2. **Worker:** use the same immutable image/build with the upstream `yarn worker:prod` command as a Daemon, if the user's Vercel team has access.
3. **Database:** managed PostgreSQL, external to containers.
4. **Queue/cache:** managed Redis, external to containers.
5. **Files:** S3-compatible object storage; never rely on container-local filesystem persistence.
6. **Migrations:** a separate, controlled release task. Do not let scaled server instances race migrations.
7. **Development:** retain official Docker Compose locally for reproducible integration tests.

Do not split the frontend from the backend until tests show that Twenty's bundled single-origin server cannot be used. Keep the public application origin stable for authentication cookies, API requests and realtime functionality.

## Required proof

- Build the explicit production image and record image size, build duration and memory use.
- Confirm the container serves HTTP using the platform-provided port and passes health checks.
- Test login, sessions/cookies, REST, GraphQL, frontend assets and the exact realtime transport used by this pinned Twenty revision.
- Test uploads/downloads and prove objects survive container replacement.
- Test PostgreSQL pool sizing/reconnects and Redis reconnects with BullMQ workload.
- Test worker success, retry/backoff, duplicate delivery, SIGTERM and restart recovery.
- Prove migrations run once and preview environments cannot mutate production.
- Perform a real backup restore into an isolated environment.
- Measure cold start, request latency, worker stability and total monthly cost.

## Consequences

**Benefits:** preserves the literal Twenty fork; honours the requested Vercel-first direction; keeps durable state in managed services; allows one pinned application build to be used by server and worker.

**Risks:** Vercel Services/Daemons are evolving platform features; realtime behaviour, process lifecycle and region/network paths must be verified. Managed Postgres, Redis and object storage may cost more than the application compute.

**Plan/data gate:** the connected team currently reports Vercel Hobby. Hobby is restricted to personal/non-commercial use. Use synthetic data only until the project is on a plan and terms suitable for business operations. Do not import real contacts, interviews or commercial research data to Hobby.

## Alternatives rejected

- Static frontend-only deployment: does not run the Twenty server or worker.
- Deploying Docker Compose unchanged: Vercel does not provide a Compose deployment target.
- Replacing Twenty with a new Next.js CRM: violates the literal-fork requirement.
- Moving everything off Vercel immediately: premature until a deployment spike identifies a concrete incompatibility.

## Revisit conditions

If realtime, worker lifecycle, persistence, networking or cost fails acceptance, retain Vercel for compatible components and move only the incompatible server/worker runtime to a persistent container host. This is a contingency, not the default.

## References

- https://vercel.com/docs/services
- https://vercel.com/docs/daemons
- https://vercel.com/changelog/vercel-daemons-is-now-in-public-beta
- https://vercel.com/kb/guide/does-vercel-support-docker-deployments
- https://vercel.com/i/can-you-run-docker-compose-on-vercel
- https://vercel.com/legal/terms
- https://github.com/twentyhq/twenty
