# Malawi Market Intelligence CRM — Architecture Foundation Plan

**Status:** Proposed architecture; implementation has not started  
**Repository:** rebshogan/twenty-crm-reb  
**Planning branch:** architecture/foundation-plan  
**Baseline commit:** d4fd19e3b9ec05406090510756814aafa44d30f2  
**Prepared:** 2026-10-30

## 1. Product outcome

Build a maintainable fork of Twenty CRM for evidence-led market research in Malawi. The system should support the complete research loop:

1. Discover businesses and suppliers across sectors and regions.
2. Verify and qualify organisations.
3. Identify decision-makers using public, business-relevant contact details.
4. Prepare and approve outreach; never send automatically without explicit human approval.
5. Record interviews and distinguish direct evidence from assumptions.
6. Convert repeated, costly problems into opportunity hypotheses.
7. Rank opportunities using evidence quality, customer spend, urgency, reachable market, delivery feasibility and founder fit.
8. Trace every opportunity back to the businesses, interviews and sources supporting it.

This is an internal research CRM first, not a generic CRM SaaS product. Preserve Twenty's CRM foundation; do not rebuild existing functionality.

## 2. Product principles and scope

### Preserve from Twenty

Keep useful upstream capabilities unless evidence shows they are unnecessary:

- Companies and people, relationships and record pages
- Saved views, filtering, sorting, search and table workflows
- Tasks, notes, activity timelines and reminders
- Custom objects, fields and relationships
- Import/export and bulk editing
- User roles, permissions and workspace administration
- API, webhooks and supported integrations
- Email/calendar capabilities where reliable and useful

### Add the Malawi research layer

- Research qualification fields and verification states
- Outreach history, follow-up dates and approval gates
- Structured interview capture
- Problem evidence and confidence levels
- Opportunity assessment and validation experiments
- Supplier/partner capability and sourcing records
- Source register and evidence traceability
- Research-specific dashboard and reports

### Defer

Do not initially build custom billing, a public multi-tenant SaaS, a custom email delivery platform, a new workflow engine, autonomous AI outreach, or a wholesale visual redesign. First establish a reliable workflow using Twenty's existing UI and platform.

## 3. Architecture decision — Vercel-first, gated by a deployment spike

The checked-out repository is an Nx/Yarn monorepo. The root package specifies Yarn 4.18.1 and Node 26.5.1. The server is NestJS, the frontend is React/Vite, and the server depends on PostgreSQL, Redis and BullMQ. The official Docker Compose topology runs a server, worker, PostgreSQL and Redis. The repository includes a multi-stage production Dockerfile at packages/twenty-docker/twenty/Dockerfile.

### Proposed runtime topology

- **Vercel project:** one Git-connected project for the fork.
- **HTTP application:** Twenty's existing server plus compiled frontend, packaged as an OCI container and deployed as a Vercel container-backed service/function. Keep the server as the single public application origin for the first release.
- **Background worker:** the same pinned Twenty build, started with yarn worker:prod, deployed as a Vercel Daemon if the account has access and the BullMQ worker passes lifecycle tests.
- **PostgreSQL:** managed PostgreSQL outside the application container, provisioned through a supported Vercel Marketplace integration if region, connection limits and price are acceptable.
- **Redis:** managed Redis for BullMQ and cache, configured with production-safe persistence and noeviction behaviour compatible with Twenty.
- **Object storage:** S3-compatible storage for uploads and attachments. Do not rely on container-local disk.
- **Local development:** retain upstream Docker Compose for reproducible development and integration tests.

Vercel Services and Daemons are promising because Services can deploy container-backed HTTP applications and Daemons can run pull-based BullMQ workers. Both are evolving platform capabilities, so this architecture remains a hypothesis until the deployment spike proves it on this Vercel team and plan.

### Required deployment spike — synthetic data only

Before product customisation, prove all of these:

1. Build the upstream production image from the checked-in Dockerfile; record build duration, image size and memory use.
2. Deploy the HTTP container and verify /healthz, login, REST/GraphQL requests and frontend delivery.
3. Verify the realtime features Twenty actually uses, including WebSocket/subscription behaviour, through the Vercel routing/runtime path.
4. Connect to managed PostgreSQL and Redis; verify connection pooling, reconnect behaviour and queue health.
5. Run the BullMQ worker as a Daemon; test job completion, retries, duplicate delivery, graceful SIGTERM and restart recovery.
6. Move file storage to S3-compatible storage; test upload, download, permissions and persistence across container restarts.
7. Run database migrations as a separate controlled release task. Do not let multiple scaled HTTP instances race to migrate the same database.
8. Test backup and restore, and verify a preview deployment cannot accidentally write to production data.
9. Record cold-start, request latency, worker stability, storage cost and database/Redis cost.

**Go/no-go:** use the Vercel-first topology only if all critical tests pass and total recurring cost is acceptable. If the worker or server lifecycle is unreliable, keep suitable stateless pieces on Vercel but move the persistent Twenty server/worker runtime to a container host. That is a contingency, not the default plan.

## 4. Data model

Use Twenty's built-in objects before introducing parallel copies.

| Research concept | Twenty implementation | Important fields / relationships |
|---|---|---|
| Business prospect | Extend native Company object | Industry, region, town, size, business model, lead source, verification state, qualification status, website/social links |
| Decision-maker | Native Person object | Role/title, public business email/phone, LinkedIn, contact source, verification state; linked to Company |
| Outreach touch | Custom object linked to Company and Person | Channel, draft text, approval state, sent state, sent date, follow-up date, Gmail draft/message ID, owner |
| Interview | Custom object linked to Company and Person | Date, method, interviewer, consent/permission, structured answers, notes, evidence quality, follow-up |
| Problem signal | Custom object linked to Company and Interview | Problem statement, category, frequency, severity, current workaround, consequences, spend evidence/range, sourcing difficulty, FX exposure, confidence |
| Opportunity hypothesis | Custom object linked to Problem Signals and Evidence | Target segment, proposed offer, business model, alternatives, willingness-to-pay evidence, supplier/delivery constraints, market attractiveness, founder fit, next experiment, decision |
| Supplier/partner | Native Company with supplier/partner role | Capability, products/services, coverage, capacity evidence, reliability, source and verification |
| Source/evidence | Custom object linked to relevant records | URL/file, source type, publisher, publication/collection date, claim, summary, reliability, caveats, evidence owner |

Avoid copying the same organisation into separate Business and Supplier tables. One Company may be a prospect, supplier, buyer, partner or competitor; represent this with roles/tags and relevant fields.

Use explicit evidence categories: **direct interview**, **documented/public source**, **observed behaviour**, **estimate**, and **hypothesis**. Do not turn a founder's assumption into a confirmed market fact.

## 5. Core user workflows

### A. Lead discovery and qualification

Capture source and discovery date, check for duplicates, verify the organisation's existence and location, classify its sector and business model, and record why it is worth contacting. Online directory presence is a lead source, not proof of market size or demand.

### B. Outreach with a human approval gate

A user prepares a message or Gmail draft. The record moves through Draft → Needs approval → Approved → Sent → Replied / No response / Follow-up. Only an explicit user action may send. Store sender, recipient, timestamp and provider message ID. Never auto-send from a scrape or AI-generated list.

### C. Interview capture

Use a structured interview form but allow free-text notes. Capture consent, date, respondent role, business context, outsourced work, recurring cost categories, supply gaps, alternatives, consequences and evidence. Do not require confidential supplier names, contract copies or exact prices.

### D. Evidence-to-opportunity

A problem signal should be supported by one or more businesses or sources. An opportunity should link to the problem signals that justify it. Rank opportunities with visible criteria and confidence, not a mysterious AI score. Track the next cheapest test that could disprove the hypothesis.

### E. Research dashboard

Prioritise operational questions:
- How many businesses are discovered, verified and qualified?
- Which sectors/regions are under-sampled?
- Which outreach items need action?
- How many interviews are completed and with which respondent types?
- Which problems have repeated evidence and verified spend?
- Which opportunity hypotheses have willingness-to-pay evidence?
- Which hypotheses should be tested, paused or rejected?

Do not present vanity counts as market validation.

## 6. Integration boundaries

- **Gmail:** create drafts and, after explicit approval, send through the connected account. Store provider IDs and sync status. Never log OAuth tokens or message bodies unnecessarily.
- **Google Calendar:** create interview events only after the user confirms the schedule; store event IDs and sync status.
- **Prospect research/browser automation:** treat retrieved business details as untrusted input. Store source URLs, timestamps and verification status. Require human review before creating a contact or initiating outreach.
- **Exports:** support CSV export for portability and backups; preserve stable record IDs and relationship mappings.

Keep integrations behind service modules/adapters so provider changes do not leak into domain logic.

## 7. Security, privacy and licensing

- Keep the repository public only if intentional; never commit credentials, .env files, OAuth tokens, database URLs or interview data.
- Use least-privilege roles, secure session settings, rate limits, audit events and tested backups.
- Collect only public, business-relevant contact details; record source and purpose. Respect opt-outs and deletion requests.
- Use synthetic data for deployment experiments. Do not import real contacts/interviews until the hosting plan and data-handling terms are suitable.
- The fork is predominantly AGPLv3, with some MIT packages and files marked @license Enterprise. Do not copy or modify Enterprise-marked files without appropriate commercial rights. Preserve upstream notices, visibly mark modifications and dates, publish the corresponding source for the modified network service as AGPLv3 requires, and use original branding rather than Twenty trademarks. This plan is not legal advice.

**Vercel plan gate:** the connected Vercel team currently reports Hobby. Vercel's terms restrict Hobby to personal/non-commercial use; its terms also describe different content/model-training treatment for Hobby/trial versus paid Pro. Use only synthetic, non-sensitive data for a prototype on Hobby. Before using this for business operations or real interview/contact records, review current terms and upgrade to an appropriate paid plan or choose hosting whose terms meet the project's needs.

## 8. Delivery phases and exit criteria

### Phase 0 — Repository and compliance baseline
- Pin the upstream baseline commit and keep a clear upstream remote/rebase strategy.
- Confirm license boundaries and branding plan.
- Record current build/test commands and existing CI status.
- Add architecture decision records before structural changes.

**Exit:** reproducible baseline, no accidental Enterprise-only code use, documented upstream sync strategy.

### Phase 1 — Vercel deployment spike
- Validate the proposed HTTP container + worker Daemon + managed Postgres/Redis + S3 topology with synthetic data.
- Keep migrations separate and test backup/restore.
- Document actual Vercel plan availability, regional placement and cost.

**Exit:** all critical deployment spike checks pass, or a documented alternative runtime is selected.

### Phase 2 — Research data model
- Configure native Company and Person objects.
- Add only required research custom objects, relationships, fields, views and validation.
- Seed development fixtures and test imports.

**Exit:** a user can create a business, link a decision-maker, record outreach and an interview, attach problem evidence and create a traceable opportunity.

### Phase 3 — Workflow and UI adaptation
- Add research navigation, list views, record layouts, interview forms, opportunity assessment and dashboard.
- Reuse Twenty components and patterns; avoid a wholesale rewrite.
- Check responsive layouts, keyboard access, empty states, loading, error and permission states.

**Exit:** a new researcher can complete the core workflow without training or workarounds.

### Phase 4 — Integrations and data migration
- Add Gmail draft/send approval flow and Calendar scheduling.
- Import existing Airtable prospect records only after deduplication and field mapping.
- Preserve source provenance and report skipped/merged rows.

**Exit:** integrations are idempotent and auditable; migration counts reconcile.

### Phase 5 — Hardening and controlled launch
- Add tests, observability, backup/restore drills, security review, rate limits and access review.
- Run a small pilot with a limited set of businesses and synthetic/test accounts first.
- Confirm no automated outbound messages can bypass approval.

**Exit:** end-to-end workflows pass and an operator can recover from failed deployment, queue backlog and database restore.

## 9. Engineering standards

- Work on short-lived feature branches and use pull requests; do not make large unreviewed changes directly on main.
- Keep upstream changes isolated from Malawi-specific domain code where practical.
- Prefer small adapters and extension points over patching Twenty core internals.
- Add tests with every behaviour change: unit tests for domain rules, integration tests for persistence/queues, and browser tests for critical user journeys.
- Keep migrations forward-only where possible, with rollback through tested backup/restore rather than destructive scripts.
- No secrets in commits, CI logs, browser traces or screenshots.
- No production deploy or real-data migration until the user has reviewed the preview and the required plan/privacy gates are resolved.

## 10. Architecture references

- Twenty source: https://github.com/twentyhq/twenty
- Twenty production Dockerfile: https://github.com/twentyhq/twenty/blob/main/packages/twenty-docker/twenty/Dockerfile
- Twenty Docker Compose: https://github.com/twentyhq/twenty/blob/main/packages/twenty-docker/docker-compose.yml
- Twenty self-hosting: https://docs.twenty.com/developers/self-host/self-host
- Vercel Services: https://vercel.com/docs/services
- Vercel container images: https://vercel.com/docs/functions/container-images
- Vercel Daemons: https://vercel.com/docs/daemons
- Vercel Hobby plan: https://vercel.com/docs/plans/hobby
- Vercel terms: https://vercel.com/legal/terms
- AGPLv3: https://www.gnu.org/licenses/agpl-3.0.html

---

**Implementation status:** planning branch only. No application code, infrastructure resources, secrets, database, worker, or production deployment has been changed by this document.
