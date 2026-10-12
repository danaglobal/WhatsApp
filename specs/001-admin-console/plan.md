# Implementation Plan: Admin Console v1

**Branch**: `001-admin-console` | **Date**: 2026-10-10 | **Spec**: [spec.md](./spec.md)

**Input**: `/specs/001-admin-console/spec.md` plus the prescribed TypeScript, container, Meta,
security, observability, testing, and deployment stack.

## Summary

Build a portable multi-tenant WhatsApp administration platform as a pnpm TypeScript monorepo. A
NestJS/Fastify API owns authentication, tenant-scoped HTTP and Socket.IO contracts, Embedded Signup
completion, and fast webhook ingestion. A separate NestJS/BullMQ worker processes durable webhook
deliveries, coexistence sync, media downloads, template updates, and rate-limited sends. PostgreSQL
16 with Prisma, tenant-context repositories, and forced row-level security is the source of truth;
Redis 7 provides queues, sessions, rate limits, and realtime fan-out; S3-compatible storage freezes
inbound media before Meta URLs expire. React 18 compiles to static assets and calls only the backend.

The webhook durability boundary is one PostgreSQL transaction that stores the verified raw delivery
and an outbox record. The API returns `200` after commit; an outbox dispatcher submits idempotent
BullMQ jobs. Domain processors deduplicate by provider identifiers and monotonic state rules, and
dead-lettered work remains replayable by correlation ID.

## Technical Context

**Language/Version**: TypeScript on the latest patched Node.js 22 LTS line; ECMAScript modules;
strict TypeScript in every workspace. Node 22 reaches end of support on 2027-04-30, so Node 24
migration is a dated follow-up before that point.

**Primary Dependencies**: pnpm workspaces, NestJS with Fastify, BullMQ, Prisma, Zod, Pino,
OpenTelemetry, Socket.IO with Redis adapter, React 18, Vite, TanStack Query, React Router, Tailwind
CSS, and i18next.

**Storage**: PostgreSQL 16 for durable relational/event state; Redis 7 for queues, sessions,
rate-limit state, and realtime fan-out; an S3-compatible provider for media. MinIO is pinned for
local development only because its community distribution is no longer a suitable production
default; production uses a maintained S3-compatible service.

**Testing**: Vitest, Supertest, Testcontainers for PostgreSQL/Redis/S3 integration, sanitized Meta
webhook fixtures for contracts, and Playwright for critical browser journeys.

**Target Platform**: OCI-compatible Linux containers on amd64 and arm64; evergreen desktop
browsers; HTTPS at every non-local ingress.

**Project Type**: Multi-process web platform in one TypeScript monorepo: API, worker, static web
application, shared contracts/Meta adapter, and database package.

**Performance Goals**:

- Commit and acknowledge 95% of valid webhook deliveries within 200 ms and 99% within 1 second.
- Surface 95% of live message/status events in the browser within 5 seconds of receipt.
- Surface 95% of template-state changes within 30 seconds.
- Make 95% of conversation lists and threads usable within 2 seconds at target load.
- Sustain 100 webhook events/second and a 1,000 events/second five-minute burst on the reference
  production profile, scaling API and workers independently.
- Start eligible coexistence sync immediately and always within Meta's 24-hour limit.

**Constraints**:

- Pin `META_GRAPH_API_VERSION=v26.0`; upgrades require official-doc review and refreshed fixtures.
- Coexistence sends are limited to 20 messages/second per number. The adapter also enforces
  recipient-pair limits and provider `Retry-After` guidance; other number limits remain configured.
- Verify `X-Hub-Signature-256` from exact raw request bytes before persistence.
- No token, credential, encryption key, or privileged endpoint is delivered to the browser.
- Encrypt Meta tokens with AES-256-GCM using versioned key IDs, unique nonces, and authenticated
  tenant/connection context; master keys come only from environment or a secret manager.
- Images are immutable and deployment-neutral; hostnames, secrets, backing services, and replica
  counts are runtime inputs. The static web app uses same-origin relative paths.
- Embedded Signup v4 launch parameters remain a release-gate contract check: current Meta pages
  conflict between `featureType`/`feature_type` and `sessionInfoVersion` for coexistence. Sandbox
  fixtures must confirm the exact configuration before production implementation.
- Accepted webhooks have RPO zero at the database commit boundary. Queue recovery/dead-letter
  replay targets 15 minutes after service restoration, with alerts within 5 minutes of exhaustion.

**Scale/Scope**: 250 tenants, 25 linked numbers per tenant, and 1 million imported plus live
messages per tenant per month with cumulative active-tenant retention. Phase one supports one
tenant Admin. `Agent` remains a reserved, non-assignable role value only.

## Constitution Check

*GATE: Passed before research and re-checked after Phase 1 design.*

| Constitutional obligation | Pre-research | Post-design evidence |
|---|---|---|
| UI-only web layer; no cross-layer shortcuts | PASS | Web imports browser-safe contracts only and calls `/api/v1`/Socket.IO. |
| Provider access through replaceable adapters | PASS | One server-only Meta interface has fake and fixture-backed implementations. |
| Format, lint, strict type checks with zero warnings | PASS | Root CI commands fail on warnings in every workspace. |
| Test-first rules and spec traceability | PASS | Tasks must map all FR/SC IDs to tests before implementation. |
| 90% line coverage | PASS | CI enforces 90% overall plus direct rule tests. |
| Contract and webhook flow tests | PASS | Versioned fixtures cover every subscribed field, duplicate, retry, and order case. |
| Environment config and versioned public APIs | PASS | Zod validates env; API is `/api/v1`; Graph v26.0 is configured. |
| Correlated, privacy-safe observability | PASS | Pino redaction and OpenTelemetry propagate request/message IDs without payloads. |
| Strict tenant isolation | PASS | Guards, tenant repositories, `tenant_id`, forced RLS, scoped jobs/objects/rooms, and adversarial tests. |
| Signature/freshness verification and least privilege | PASS | Exact-byte HMAC, field-aware freshness/replay checks, and reviewed adapter scopes precede trusted processing. |
| Immutable audit coverage | PASS | Append-only audit entries cover specified Admin/platform actions without message content. |
| Async retry, DLQ, and rate limits | PASS | Durable inbox/outbox, bounded BullMQ retries, DLQs, and per-number/pair limiters. |
| Dependency/abstraction justification | PASS | Each workspace is a deployable process or enforced trust boundary. |

No constitutional violation remains. Keeping `Agent` as a disabled enum value is documented because
the requested auth vocabulary and future CRM context name it; tests prove it cannot be provisioned.

## Project Structure

### Documentation (this feature)

```text
specs/001-admin-console/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   ├── openapi.yaml
│   ├── meta-webhooks.md
│   ├── realtime-events.md
│   └── queue-jobs.md
└── tasks.md                 # generated later by $speckit-tasks
```

### Source Code (repository root)

```text
apps/
├── api/
│   ├── src/modules/{auth,tenants,whatsapp-connections,templates,conversations,audit,webhooks,realtime,health}/
│   ├── src/main.ts
│   └── test/{contract,integration,security}/
├── worker/
│   ├── src/processors/{webhook,coexistence-sync,media,templates,outbound,retention}/
│   ├── src/main.ts
│   └── test/{queues,integration}/
└── web/
    ├── src/app/
    ├── src/features/{auth,whatsapp,templates,conversations,audit,platform-tenants}/
    ├── src/{components,i18n,realtime,services,host}/
    └── test/e2e/
packages/
├── shared/
│   └── src/{browser,server,meta}/
└── db/
    ├── prisma/{schema.prisma,migrations/}
    ├── src/{repositories,tenant-context.ts,outbox.ts}
    └── test/rls/
infra/{caddy,docker,tunnels}/
test/{fixtures/meta,contract,integration,performance}/
.github/workflows/
docker-compose.yml
docker-compose.production.yml
pnpm-workspace.yaml
package.json
tsconfig.base.json
eslint.config.js
.env.example
```

**Structure Decision**: Use the requested five workspaces. `apps/api` is the synchronous trust
boundary, `apps/worker` owns asynchronous side effects, and `apps/web` is unprivileged. Only
`packages/db` accesses PostgreSQL. `packages/shared` exposes separate browser-safe, server-only,
and Meta subpaths; package exports and lint rules prevent the web app importing provider clients.

## Design Decisions

### Durable webhook ingestion

1. Fastify captures raw bytes and verifies the HMAC in constant time. A field-aware freshness policy
   rejects stale live events while allowing expected historical timestamps only for an active sync;
   the delivery digest blocks exact replay.
2. A short transaction stores `WebhookDelivery` and `OutboxEvent`; then the API returns `200`.
3. An outbox publisher inserts an idempotent BullMQ job. Redis downtime cannot lose a committed event.
4. Workers validate field-specific Zod contracts and apply domain idempotency keys.
5. Exhausted jobs enter named DLQs containing payload references, not payloads in logs.

### Tenant isolation

- Every tenant-owned row, object key, job, Socket.IO room, and cache key carries `tenant_id`.
- Guards derive tenant context from the server-side session, never the request body.
- Repositories require `TenantContext` and transact with `SET LOCAL app.tenant_id`. Migrations enable
  and force RLS; the application role is not a table owner and has no `BYPASSRLS`.
- Super-admin metadata reads use an audited repository path that cannot select message bodies,
  contacts, media, or token ciphertext.
- Tests cover identifier substitution, missing context, cross-tenant joins, replay, object-key
  guessing, cache access, and Socket.IO room joins.

### Ordering and idempotency

- Messages are unique by connection plus `wamid`; repeated delivery never duplicates content.
- Status progression is monotonic, retaining provider and ingestion times separately.
- History chunks are raw records keyed by request, phase, and `chunk_order`; assembly uses the same
  message idempotency path as live events.
- Template, connection, and sync changes use event-specific unique keys and optimistic versions.
- A transactional outbox drives realtime publications and outbound provider calls.

### Authentication and browser security

- Argon2id hashes carry benchmarked parameters. Opaque Redis sessions use a 30-minute sliding
  inactivity timeout, absolute lifetime, rotation after authentication, and sign-out revocation.
- Cookies are `HttpOnly`, `Secure`, host-only, and `SameSite=Lax`; state changes require a
  session-bound CSRF token header. Login/reset responses do not enumerate accounts.
- Invite/reset tokens are random, hashed, purpose-bound, expiring, and single-use. Only the initial
  Admin invitation is available in phase one.
- Socket.IO authenticates the same session, validates origin, rechecks tenant state, and authorizes
  every room join.

### Meta adapter and media

- One server-only adapter owns Graph v26.0 calls, scopes, retries, backoff, `Retry-After`, versioning,
  recipient-pair limits, and per-number throughput.
- Embedded Signup attempts use a signed single-use state nonce. The browser allow-lists Meta origins,
  validates forward-compatible `WA_EMBEDDED_SIGNUP` events, and sends the code to the backend for
  exchange. `FINISH_WHATSAPP_BUSINESS_APP_ONBOARDING` identifies coexistence completion.
- Both modes exchange the code and subscribe `POST /{WABA_ID}/subscribed_apps`. Coexistence skips
  phone registration and requests contacts/history via `POST /{PHONE_NUMBER_ID}/smb_app_data` once
  within 24 hours; API-only numbers call `POST /{PHONE_NUMBER_ID}/register`.
- Token ciphertext records algorithm, key ID, nonce, tag, and ciphertext; authenticated data binds
  tenant, connection, purpose, and schema version.
- Media workers obtain a fresh URL and immediately stream authenticated bytes to tenant-scoped S3.
  URLs expire after five minutes and are never treated as durable storage.

### Realtime and reads

- Socket.IO rooms are server-derived by tenant, connection, and conversation. Events carry safe
  IDs/status only; TanStack Query refetches authoritative HTTP state after invalidation or reconnect.
- Conversation cursors use last activity plus ID; message cursors use provider timestamp plus ID.
- Indexes start with `tenant_id` and query order columns. Time partitioning is added only if the
  benchmark justifies it.
- Vite uses relative assets and same-origin API/socket paths. A small host seam permits a future
  Zoho build, but no Zoho SDK, iframe auth, or CRM call is implemented now.

### Deployment and operations

- Multi-stage, non-root Debian slim images are pinned by digest for releases and published for
  amd64/arm64. The web image contains only static files and Caddy.
- Caddy terminates TLS and routes `/api`, `/socket.io`, and SPA fallback. A platform ingress may
  replace it without changing images.
- Compose supplies laptop and single-VPS profiles. Target scale/HA requires appropriately sized or
  managed PostgreSQL, Redis, S3, and independent API/worker replicas; image portability does not
  promise equal capacity on every VPS.
- A one-shot migration job runs `prisma migrate deploy`; application replicas never race migrations.
- `/health` checks liveness; `/ready` checks required dependencies and migration compatibility.
- GitHub Actions runs frozen install, formatting, zero-warning lint, strict typecheck, migration
  validation, unit/contract/integration/E2E tests, 90% coverage, build, scan, SBOM/attestation, and
  multi-architecture image push only after all gates pass.

## Delivery Sequence

1. Workspace tooling, strict CI, environment validation, and container skeletons.
2. Schema, migrations, tenant context, forced RLS, repositories, and adversarial isolation tests.
3. Auth, initial invitation, session/CSRF, tenant lifecycle, and audit.
4. Durable webhook inbox/outbox, queues, correlation, DLQ/replay, and fixture contracts.
5. Meta adapter, Embedded Signup, token encryption, WABA subscription, and onboarding branching.
6. Coexistence sync, messages/statuses/echoes, media freezing, and rate-limited sends.
7. Template/conversation APIs, read models, Socket.IO invalidations, and static web UI.
8. Super-admin operations, disconnect/reconnect, retention cleanup, and operational views.
9. App Review Playwright journeys, load tests, portability checks, runbooks, and final compliance.

## Release Gates

- All 62 FRs and 12 SCs have direct test or review evidence.
- Official Meta docs/app dashboard confirm Graph v26.0, Embedded Signup v4 parameters, scopes,
  endpoints, fields, and limits; sanitized fixtures are refreshed.
- Cross-tenant tests pass for HTTP, repository/RLS, queue, object, cache, and realtime boundaries.
- Duplicate, retry, delayed, and out-of-order fixtures converge to one outcome.
- Coverage is at least 90%; all quality/build/security gates emit no warnings.
- Both Meta App Review recordings complete against the release candidate using only the console.
- Backup/restore, DLQ replay, key rotation, token revocation, suspension/restoration, and retention
  deletion are rehearsed outside production.
