# Vercel deployment spike — execution checklist

**Branch:** architecture/foundation-plan  
**Environment:** isolated preview, synthetic data only  
**Status:** Not executed

## Gate 0 — project and source integrity

- [ ] Confirm Git integration points to `rebshogan/twenty-crm-reb`.
- [ ] Confirm repository root is the build root.
- [ ] Record the exact deployment commit and branch.
- [ ] Ensure no secrets or .env files are committed or printed in logs.
- [ ] Replace the accidental static/default deployment configuration with an explicit container/service configuration only after platform capability is verified.

## Gate 1 — build the production image

- [ ] Build from repository root with `packages/twenty-docker/twenty/Dockerfile`.
- [ ] Use explicit `--target twenty` for the server plus compiled frontend. Do not use the final `twenty-app-dev` stage.
- [ ] Record Node/Yarn/Nx versions, build time, image size, memory peak and build cache behaviour.
- [ ] Confirm the image contains the compiled frontend, server and production dependencies.
- [ ] Confirm the worker can run from the same image with the official `yarn worker:prod` script.
- [ ] Confirm server and worker use the same pinned image digest.

## Gate 2 — HTTP runtime

- [ ] Provision disposable PostgreSQL and Redis services.
- [ ] Configure S3-compatible object storage.
- [ ] Start server with platform-provided port and production environment.
- [ ] Verify /healthz, readiness, login, cookies/session, GraphQL and REST.
- [ ] Verify frontend assets and all deep links.
- [ ] Identify and test the exact realtime transport (WebSocket/subscription/SSE) used by this pinned version.
- [ ] Verify upload/download permissions and persistence after instance replacement.
- [ ] Verify graceful SIGTERM and clean restart.

## Gate 3 — background worker

- [ ] Start `yarn worker:prod` from the same image.
- [ ] Enqueue representative jobs and verify completion.
- [ ] Verify retry/backoff and duplicate delivery behaviour.
- [ ] Verify graceful SIGTERM and restart recovery.
- [ ] Verify worker scaling does not produce duplicate side effects.
- [ ] Verify queue depth, failures and stuck jobs are observable.

## Gate 4 — database, Redis and release safety

- [ ] Run migrations as a separate, controlled release task.
- [ ] Prove multiple HTTP instances cannot race migrations.
- [ ] Verify PostgreSQL pool sizing and reconnect behaviour.
- [ ] Verify Redis authentication, TLS, ACL and noeviction/queue-safe settings.
- [ ] Ensure preview uses isolated DB, Redis and object bucket.
- [ ] Create backup and perform restore to an isolated environment.
- [ ] Confirm all durable data is outside ephemeral containers.

## Gate 5 — acceptance record

| Metric | Result |
|---|---|
| Production image build time | Pending |
| Production image size | Pending |
| Server memory peak | Pending |
| Worker memory peak | Pending |
| Health/readiness | Pending |
| Login + API | Pending |
| Realtime | Pending |
| Upload persistence | Pending |
| Queue retries/restarts | Pending |
| Database restore | Pending |
| Vercel monthly estimate | Pending |
| PostgreSQL monthly estimate | Pending |
| Redis monthly estimate | Pending |
| Object storage monthly estimate | Pending |

**Pass:** critical HTTP, realtime, queue, persistence, security and recovery checks pass at an acceptable recurring cost.

**Fail:** a required stateful or realtime behaviour cannot be supported reliably. Do not mask a failed runtime test by publishing a static page. Record the failing behaviour and evaluate the smallest alternative runtime change.

## Current constraints

- Vercel Services document container-backed HTTP runtimes.
- Vercel Daemons entered public beta on 15 October 2026 and explicitly document BullMQ consumers.
- Daemons have no persistent filesystem; durable state must live in backing services.
- Vercel Hobby is for personal/non-commercial use only. Keep real business contacts and interview records off Hobby.
- A deployment being marked READY is not a successful CRM test; each acceptance item needs a captured result.
