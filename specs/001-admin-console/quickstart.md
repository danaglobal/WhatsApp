# Quickstart: Admin Console Platform

This guide defines the expected development and verification workflow for the implementation described by this feature. Commands become executable as the implementation tasks land.

## Prerequisites

- Docker Engine with Compose v2
- Node.js 22 LTS (minimum version pinned in the root `package.json`)
- Corepack-enabled pnpm at the repository-pinned version
- A Meta developer app, WhatsApp test assets, and an Embedded Signup configuration for real sandbox verification
- Optional: ngrok or Cloudflare Tunnel for public HTTPS webhook testing

Do not install Postgres, Redis, or object storage directly for normal development; Compose supplies the same service versions used by the single-server deployment profile.

## Repository layout

```text
apps/
├── api/        # NestJS/Fastify HTTP, auth, webhook receipt, Socket.IO
├── worker/     # NestJS/BullMQ asynchronous processing
└── web/        # React/Vite static admin console
packages/
├── db/         # Prisma schema, migrations, tenant repositories
└── shared/     # Types, Zod contracts, Meta client, safe utilities
```

The web package imports browser-safe shared subpaths only. Neither the web bundle nor its build-time variables may import server configuration, Meta clients, encryption code, tokens, or secrets.

## Configure the environment

1. Copy `.env.example` to `.env.local` for local use. Never commit the populated file.
2. Generate independent high-entropy values for session signing, CSRF, invitation/reset token signing, and the credential master key.
3. Configure these groups:

| Group | Examples | Notes |
|---|---|---|
| Runtime | `NODE_ENV`, `LOG_LEVEL`, `PUBLIC_ORIGIN` | Production origin must be HTTPS |
| Postgres | `DATABASE_URL`, migration-owner URL | Runtime role must not own tables or bypass RLS |
| Redis | `REDIS_URL` | Production uses persistence and `noeviction` |
| Object storage | endpoint, region, bucket, access credentials | MinIO-compatible local service; maintained S3 provider in production |
| Security | session keys, credential master key/version | Secret manager or injected environment only |
| Meta | app ID, app secret, configuration ID, verify token, `META_GRAPH_API_VERSION=v26.0` | Revalidate before version upgrades |
| Mail | SMTP/provider values | Used for invitation and reset links |

Frontend runtime calls same-origin `/api`, `/webhooks`, and `/socket.io` paths. Do not compile environment-specific backend URLs into the Vite bundle; Caddy routes them.

## Start the local platform

```powershell
corepack enable
corepack pnpm install --frozen-lockfile
docker compose --profile dev up --build
```

Compose starts Postgres 16, Redis 7, local S3-compatible storage, the migration job, API, worker, web server, and Caddy. Dependency health checks gate the migration and application containers. The migration job must complete before API and worker become ready.

Expected local endpoints:

- Admin console: `https://localhost/`
- API readiness: `https://localhost/ready`
- API liveness: `https://localhost/health`
- Meta webhook: `https://localhost/webhooks/meta/whatsapp`
- Object-store console: bound to localhost only, on the port documented by `.env.example`

Local TLS certificates may require explicit trust. No database, Redis, object-store, or internal application port is publicly exposed in the production Compose profile.

## Seed development identities

Run the idempotent development seed through the API package:

```powershell
corepack pnpm --filter @app/api seed:dev
```

The seed creates one SuperAdmin and a sample tenant with exactly one tenant Admin. It must refuse to run in production and must source passwords from local environment variables. It does not create an Agent because Agent provisioning is out of phase-one scope.

## Run quality gates

```powershell
corepack pnpm lint
corepack pnpm format:check
corepack pnpm typecheck
corepack pnpm test:unit
corepack pnpm test:contract
corepack pnpm test:integration
corepack pnpm build
corepack pnpm test:e2e
```

CI treats warnings as failures and enforces at least 90% line coverage globally, with 100% direct test coverage for business rules named in the specification. Integration tests use Testcontainers rather than developer-installed services.

## Validate the core flows

### 1. Tenant lifecycle and isolation

1. Sign in as SuperAdmin and create a tenant plus its single Admin invitation.
2. Accept the invitation and verify the tenant Admin cannot list/create users or access platform routes.
3. Create a second tenant and seed distinguishable messages.
4. Exercise list, detail, aggregate, nested include, websocket, media access, worker, and export paths; confirm neither tenant can observe the other.
5. Suspend the first tenant, ingest a signed webhook, and confirm processing continues while the tenant session/API/socket cannot access operational data.
6. Restore it and confirm accumulated events appear once.
7. Close a disposable tenant and verify retention schedules content deletion by 30 days while audit retention remains 24 months.

### 2. Webhook durability and ordering

Replay the sanitized fixtures from `apps/worker/test/fixtures/meta/<graph-version>/`:

- valid and invalid signatures;
- duplicated delivery and duplicated `wamid`;
- status before message;
- history chunks in reverse arrival order;
- SMB echo followed by the corresponding live/history record;
- all supported webhook fields plus unknown fields/status values;
- media URL retrieval followed by durable object-store download.

The API must acknowledge only after the webhook and outbox transaction commits. Kill the publisher between commit and enqueue, restart it, and verify the reconciliation sweep publishes exactly one logical result. No logs may contain fixture message bodies, phone numbers, tokens, or media URLs.

### 3. Outbound reliability and rate limit

Queue messages for two coexistence phone numbers while multiple workers run. Assert that each phone number begins no more than 20 sends in any rolling second and that one busy number does not consume another number's allowance. Simulate provider `429`, transient failure, permanent failure, and a socket timeout after request transmission. The ambiguous timeout must become `DELIVERY_UNKNOWN`, not an automatic duplicate send.

### 4. Admin UI and realtime

Run Playwright flows for:

- mock Embedded Signup finish/cancel/error and server-side completion;
- multi-number connection list and number-specific health;
- conversation browsing and sending;
- template creation and status update;
- reconnect after Socket.IO interruption, followed by query refetch;
- logout/cache clearing, tenant suspension, and session revocation;
- English and Arabic/RTL rendering at supported breakpoints.

Inspect the production web bundle and source maps to prove that it contains no app secret, access token, credential master key, server environment value, or server-only package.

## Test real Meta Embedded Signup

Mock fixtures are necessary but not sufficient because the Embedded Signup documentation and payload contract can change independently.

1. Start a public HTTPS tunnel:

   ```powershell
   ngrok http https://localhost:443
   ```

   or:

   ```powershell
   cloudflared tunnel --url https://localhost:443 --no-tls-verify
   ```

2. Set `PUBLIC_ORIGIN` to the stable tunnel origin, restart the relevant services, and configure `<origin>/webhooks/meta/whatsapp` in the Meta app.
3. Complete both a coexistence sandbox onboarding and a new-number sandbox onboarding.
4. Capture only sanitized, consented fixture shapes. Record capture date and `META_GRAPH_API_VERSION`; never commit authorization codes, access tokens, phone numbers, names, or message content.
5. Verify the coexistence flow skips registration and exposes the one-time contacts/history sync choices within the required 24-hour window; verify the new-number flow registers the number.

The release gate fails if the actual JS SDK parameter names, session-info event shape, subscribed webhook fields, or API response schemas differ from the pinned adapters and fixtures.

## Build portable images

```powershell
docker buildx bake --pull
docker compose --profile production config
```

The build produces three non-root OCI images: API, worker, and web/Caddy. Application images contain no environment-specific configuration. Multi-platform manifests may contain architecture-specific layers, but each target uses the same source, Dockerfile, configuration contract, and immutable version tag.

For a single VPS, provide secrets and run the production Compose profile behind Caddy automatic HTTPS. For Render, Railway, Fly.io, DigitalOcean App Platform, ECS, or Kubernetes, deploy the same image tags, supply managed Postgres/Redis/S3 endpoints, run migrations as a one-shot release job, and map health/readiness probes. Do not run schema migration concurrently in every API replica.

## Release checklist

- All quality gates pass with zero warnings.
- Constitution compliance is reviewed in the plan and pull request.
- Database restore, Redis loss/reconciliation, object-store access, and rolling shutdown are exercised.
- Tenant isolation and log-redaction negative tests pass.
- The current Meta sandbox contract is verified and its date/version recorded.
- Load tests cover the target of 250 tenants, up to 25 numbers each, and the specified message volume on a documented hardware profile.
- Images are vulnerability-scanned, dependency/SBOM artifacts are published, and deployment uses immutable digests.
