# Research: Admin Console v1

**Date**: 2026-10-10  
**Scope**: Resolve version-sensitive and architectural decisions before Phase 1 design.

## 1. Runtime and dependency baselines

**Decision**: Use the latest patched Node 22 LTS release in lockfiles and image digests, with a
mandatory Node 24 migration milestone before Node 22 EOL on 2027-04-30. Pin React 18 as explicitly
requested even though it is no longer the current React major.

**Rationale**: This honors the requested stack while acknowledging its short support runway. Vite's
current Node floor is satisfied by a patched Node 22 release, and TanStack Query v5 supports React 18.

**Alternatives considered**: Node 24 and React 19 would be better greenfield defaults but conflict
with the requested baseline; floating major versions would make builds non-reproducible.

## 2. Meta Graph API and Embedded Signup

**Decision**: Pin `META_GRAPH_API_VERSION=v26.0`. Use Embedded Signup v4 and validate
`WA_EMBEDDED_SIGNUP` messages as a forward-compatible discriminated union. Treat the authorization
code as a separate value from `response.authResponse.code` and exchange it only server-side.

For coexistence, accept the documented `FINISH_WHATSAPP_BUSINESS_APP_ONBOARDING` outcome. Do not
hardcode `sessionInfoVersion: 4`: Meta's coexistence page still shows
`featureType='whatsapp_business_app_onboarding'` and `sessionInfoVersion:'3'`, while the newer v4
page describes `feature_type` and omits that setting. A sandbox contract test must capture the
working launch parameters and finish/cancel/error variants before release.

**Rationale**: Meta's v4 page uses v26.0 and v26.0 is the current published Graph version. Tolerant
event parsing prevents a documentation inconsistency or new event subtype from corrupting signup.

**Alternatives considered**: Treating FINISH/CANCEL/ERROR as three fixed schemas was rejected
because current Meta examples report some user-facing errors as `CANCEL` and list additional finish
variants. Hardcoding an unverified session-info version was rejected.

**Primary sources**:

- https://developers.facebook.com/documentation/business-messaging/whatsapp/embedded-signup/version-4/
- https://developers.facebook.com/documentation/business-messaging/whatsapp/embedded-signup/implementation/
- https://developers.facebook.com/documentation/business-messaging/whatsapp/embedded-signup/onboarding-business-app-users/
- https://developers.facebook.com/docs/graph-api/changelog/version26.0/

## 3. Standard and coexistence onboarding

**Decision**: Both signup modes exchange the returned code with `GET /oauth/access_token` on the
server and subscribe the app using `POST /{WABA_ID}/subscribed_apps`. Register only API-only numbers
with `POST /{PHONE_NUMBER_ID}/register`. A coexistence number is already registered and must skip
that call. Start contacts and history sync through `POST /{PHONE_NUMBER_ID}/smb_app_data`, with
`sync_type` values `smb_app_state_sync` and `history`, once each and within 24 hours.

**Rationale**: This is the mode distinction documented by Meta. Re-registering a coexistence number
would break onboarding; sync calls are one-time and become unavailable after the window.

**Alternatives considered**: One generic registration pipeline was rejected because its side
effects differ by mode. Manual Admin-triggered initial sync was rejected because it risks the
24-hour limit.

**Primary sources**:

- https://developers.facebook.com/documentation/business-messaging/whatsapp/embedded-signup/onboarding-customers-as-a-tech-provider/
- https://developers.facebook.com/documentation/business-messaging/whatsapp/embedded-signup/onboarding-business-app-users/
- https://www.postman.com/meta/whatsapp-business-platform/request/c1ai24q/subscribe-to-your-waba

## 4. Webhook contract and durability

**Decision**: Subscribe to and fixture-test `messages` (including `statuses`),
`smb_message_echoes`, `history`, `smb_app_state_sync`, `account_update`,
`message_template_status_update`, and `phone_number_quality_update`. Verify
`X-Hub-Signature-256` against exact raw bytes before parsing. In one PostgreSQL transaction, store
the delivery and outbox record, then acknowledge `200`; process from the durable record through
BullMQ.

`phone_number_quality_update` uses current fields such as
`max_daily_conversations_per_business`; removed `current_limit`/`old_limit` fields remain tolerated
only in historical fixtures. Template-state parsing accepts known states and preserves unknown
future states without treating them as approval.

**Rationale**: Meta retries failed webhook delivery for up to 36 hours. Durable commit plus outbox
prevents loss when Redis is unavailable after acknowledgement; database reducers remain correct
under at-least-once queue delivery.

**Alternatives considered**: Enqueue-before-ack without a database inbox can lose work; processing
synchronously violates the fast-ack requirement; trusting a parsed/re-serialized body invalidates
HMAC verification.

**Primary sources**:

- https://developers.facebook.com/docs/graph-api/webhooks/getting-started/
- https://developers.facebook.com/documentation/business-messaging/whatsapp/webhooks/reference/messages/
- https://developers.facebook.com/documentation/business-messaging/whatsapp/webhooks/reference/message_template_status_update/
- https://developers.facebook.com/documentation/business-messaging/whatsapp/webhooks/reference/phone_number_quality_update/

## 5. Idempotency, order, and ambiguous sends

**Decision**: Enforce database uniqueness and deterministic reducers: connection plus `wamid` for
messages; sync run plus phase plus `chunk_order` for history; provider event key or canonical digest
for other transitions. Status projections never regress from read to delivered/sent. Store provider
and ingestion time separately. Replayed history and live events share one message upsert path.

Automatically retry only known pre-submission failures or explicit retryable responses. A timeout
after a request may have reached Meta becomes `delivery_unknown` and requires reconciliation or an
audited manual retry.

**Rationale**: BullMQ is at-least-once in failure cases, and external exactly-once sending cannot
be guaranteed when the provider outcome is ambiguous.

**Alternatives considered**: BullMQ job IDs alone are not a correctness boundary; unconditional
retry after an ambiguous timeout can send duplicate customer messages.

## 6. Queue topology and rate limits

**Decision**: Separate queues for webhook processing, coexistence sync, media, templates, outbound
sends, maintenance, and dead letters. Transient failures get eight exponential retries from one
second with jitter and a 15-minute cap; `429` honors provider delay; permanent validation/auth errors
fail without retry. Persist a sanitized dead-letter record and audit replay.

Implement a Redis Lua sliding-window limiter per phone-number ID: coexistence is fixed at 20 send
starts/second. Also enforce Meta's recipient-pair limit and dynamic provider cooldown. Use a
per-conversation mutex for send order.

**Rationale**: Open-source BullMQ removed group-key rate limiting; the required per-number limit
must therefore be atomic outside its basic global limiter. Redis uses AOF and `noeviction`.

**Alternatives considered**: BullMQ Pro groups are simpler but add licensing/cost; one global
limiter underutilizes independent numbers and does not meet the contract.

**Primary sources**:

- https://docs.bullmq.io/guide/rate-limiting
- https://docs.bullmq.io/guide/retrying-failing-jobs
- https://docs.bullmq.io/guide/going-to-production
- https://developers.facebook.com/docs/whatsapp/cloud-api/overview/

## 7. Tenant isolation with Prisma and PostgreSQL

**Decision**: Every tenant-owned table has `tenant_id UUID NOT NULL`; tenant-owned foreign keys are
composite. Repositories require `TenantContext` and execute in a Prisma interactive transaction
after parameterized `set_config('app.tenant_id', tenantId, true)`. Policies use `USING` and
`WITH CHECK`; RLS is enabled and forced. The runtime role is not owner/superuser and has no
`BYPASSRLS`. A separate migration role owns schema. SuperAdmin reads use audited metadata-only
views/repositories that cannot select content or ciphertext.

**Rationale**: Application predicates provide clear intent while RLS and composite foreign keys
limit damage from missing predicates or substituted IDs. Transaction-local settings prevent pooled
connection leakage.

**Alternatives considered**: Application scoping alone is too fragile; RLS alone does not replace
authorization and can be silently bypassed by owners or privileged roles.

**Primary sources**:

- https://www.postgresql.org/docs/16/ddl-rowsecurity.html
- https://www.postgresql.org/docs/16/sql-createpolicy.html

## 8. Sessions, CSRF, and password material

**Decision**: Use opaque Redis-backed sessions and a `__Host-session` cookie with `Secure`,
`HttpOnly`, `Path=/`, no `Domain`, and `SameSite=Lax`. Rotate on login, enforce a 30-minute sliding
idle timeout plus configurable absolute lifetime, and revoke on logout, reset, suspension, or
closure. Require a session-bound synchronizer token header plus Origin/Fetch Metadata checks.
Use Argon2id at a minimum of 19 MiB, two iterations, parallelism one, benchmarked upward. Store only
hashes of random, expiring, single-use invite/reset tokens.

**Rationale**: Server-side revocation is necessary for tenant suspension and credential response;
SameSite is defense-in-depth, not a CSRF substitute.

**Alternatives considered**: Browser-stored JWTs weaken immediate revocation and increase token
exposure; double-submit without session binding is less robust for this same-origin design.

**Primary sources**:

- https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html
- https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html

## 9. Meta token encryption

**Decision**: Use envelope encryption. Generate a random 32-byte data key per credential, encrypt
the token with AES-256-GCM using a fresh 12-byte IV and AAD binding tenant, credential ID, provider,
purpose, and envelope version, then wrap that data key with a versioned master key from environment
or a secret manager. Persist ciphertext metadata but never the master key.

**Rationale**: Envelope encryption supports master-key rotation by rewrapping small data keys and
limits plaintext to the Meta adapter process.

**Alternatives considered**: Direct encryption under one environment key is simpler but forces a
full credential rewrite for rotation and expands blast radius.

**Primary sources**:

- https://nodejs.org/api/crypto.html
- https://docs.aws.amazon.com/kms/latest/developerguide/data-keys.html

## 10. Media ownership and storage

**Decision**: Fetch a fresh Meta media URL, then immediately stream authenticated bytes to a
server-derived tenant object key. Store checksum, detected MIME, size, and scan/quarantine state;
serve via short-lived backend-authorized signed URLs. Never store Meta's URL as the durable link.
Use MinIO only as a pinned local-development service; production uses a maintained S3-compatible
provider.

**Rationale**: Meta media URLs expire after five minutes. Server-derived keys and authorization
prevent tenant path substitution. The MinIO OSS repository was archived in 2026, making it an
unacceptable production default.

**Alternatives considered**: Proxying expiring Meta URLs fails after expiry; public buckets violate
tenant isolation; packaging storage in application images is not portable or durable.

**Primary sources**:

- https://www.postman.com/meta/whatsapp-business-platform/request/fpj02x0/retrieve-media-url
- https://github.com/minio/minio

## 11. Realtime and frontend portability

**Decision**: Socket.IO events are versioned invalidation/upsert envelopes containing safe IDs and
revisions. PostgreSQL remains authoritative; TanStack Query refetches after events and after
reconnect because the Redis adapter does not recover missed history. Use same-origin `/api/v1` and
`/socket.io`, relative Vite assets, `BrowserRouter` for standalone deployment, and a small host seam
for a future Zoho router/auth adapter. Arabic template preview uses `i18next.dir()` and CSS logical
properties; the whole console is not localized in v1.

**Rationale**: Static, relative assets and same-origin routes keep one web image portable. Refetch
on reconnect avoids treating Pub/Sub as durable data.

**Alternatives considered**: Baking `VITE_*` host URLs makes images environment-specific; sending
full message bodies through Pub/Sub increases privacy risk; implementing the Zoho SDK now violates
scope.

**Primary sources**:

- https://vite.dev/guide/build
- https://vite.dev/guide/env-and-mode
- https://socket.io/docs/v4/redis-adapter
- https://www.i18next.com/overview/api

## 12. Containers, CI, and deployment

**Decision**: Produce immutable non-root `api`, `worker`, and `web` OCI images for amd64/arm64,
using Debian slim for Node native dependencies. Caddy serves static assets and proxies same-origin
API/socket paths. Compose covers laptop and single VPS; cloud targets use platform-specific service
definitions with the same images. Migrations run as a one-shot job. CI uses frozen pnpm installs,
zero-warning quality gates, 90% coverage, real PostgreSQL/Redis/S3 integration, production-image
Playwright, image scans, SBOMs, attestations, and SHA/semver tags.

**Rationale**: Image portability is achievable across hosts; one universal deployment manifest and
equal performance on every VPS are not. A separate migration job avoids replica races.

**Alternatives considered**: Alpine increases Prisma/Argon2 native compatibility risk; startup
migrations race under replicas; floating tags undermine reproducibility.

**Primary sources**:

- https://docs.docker.com/build/building/multi-platform/
- https://docs.docker.com/compose/how-tos/startup-order
- https://caddyserver.com/docs/automatic-https
- https://docs.github.com/en/actions/how-tos/secure-your-work/use-artifact-attestations/use-artifact-attestations

## Resolved uncertainties

All research questions are resolved. The only version-sensitive implementation gate
is empirical confirmation of current Embedded Signup v4 coexistence launch parameters in a Meta
sandbox because Meta's official pages conflict; the design explicitly handles that through a
fixture-backed release gate rather than guessing.
