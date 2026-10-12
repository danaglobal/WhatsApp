<!--
Sync Impact Report
- Version change: unratified scaffold -> 1.0.0
- Modified principles:
  - Placeholder Principle 1 -> I. Layered, Replaceable Architecture
  - Placeholder Principle 2 -> II. Test-First and Contract-Verified Behavior
  - Placeholder Principle 3 -> III. Maintainable and Traceable Delivery
  - Placeholder Principle 4 -> IV. Security and Tenant Isolation (NON-NEGOTIABLE)
  - Placeholder Principle 5 -> V. Reliable, Rate-Aware Processing
- Added sections:
  - Engineering Constraints
  - Delivery and Review Gates
- Removed sections: none; placeholder sections were concretized
- Follow-up TODOs: none
-->
# WhatsApp Coexistence CRM Constitution

## Core Principles

### I. Layered, Replaceable Architecture

- The widget MUST contain presentation and user-interaction logic only and MUST NOT call
  WhatsApp, Zoho, databases, queues, or other third-party services directly. Rationale: a
  UI-only widget keeps credentials server-side and prevents business logic from fragmenting.
- The backend API, WhatsApp adapter, Zoho adapter, admin app, and widget MUST have explicit
  module boundaries and MUST communicate only through documented interfaces. Rationale:
  enforceable boundaries prevent cross-layer shortcuts and localize change.
- All WhatsApp and Zoho operations MUST pass through adapter interfaces that can be replaced
  with deterministic test doubles. Rationale: isolation makes external behavior mockable and
  provider changes containable.
- Functions SHOULD be small, single-purpose, and named for one observable responsibility;
  exceptions MUST include a reviewable justification. Rationale: narrow functions are easier
  to reason about, test, and maintain.
- Every new dependency or abstraction MUST include written justification covering the need,
  alternatives considered, security impact, and maintenance cost. Rationale: deliberate
  additions limit supply-chain risk and accidental complexity.
- CI MUST enforce formatting, linting, and static typing with zero warnings before merge.
  Rationale: automated, warning-free gates keep the codebase consistent and defects visible.

### II. Test-First and Contract-Verified Behavior

- Business logic MUST be developed test-first: an automated test MUST express the intended
  behavior and fail for the expected reason before production logic is added. Rationale:
  red-green-refactor proves that each test can detect the behavior it governs.
- Every normative business rule and acceptance criterion in a feature specification MUST map
  to at least one automated test. Rationale: complete traceability prevents specified behavior
  from becoming unverified behavior.
- WhatsApp and Zoho adapters MUST have contract tests validated against the pinned upstream API
  schemas and versions they implement. Rationale: schema verification detects provider drift at
  the integration boundary.
- Every webhook flow MUST have integration tests for normal delivery, duplicate delivery,
  retries, delayed delivery, and out-of-order events. Rationale: webhook networks provide
  at-least-once and non-sequential delivery in practice.
- Webhook handlers MUST be idempotent, and tests MUST prove that replaying an event produces no
  duplicate externally visible effect. Rationale: idempotency makes retries safe.
- CI MUST reject a merge when repository-wide line coverage is below 90% or when a changed
  business rule lacks a directly traceable test. Rationale: a numeric floor controls regression
  while rule-level traceability protects critical behavior beyond aggregate coverage.

### III. Maintainable and Traceable Delivery

- Every feature MUST link its implementation and tests to an approved specification with
  explicit acceptance criteria. Rationale: bidirectional traceability keeps intent, code, and
  verification aligned.
- Runtime configuration MUST come from validated environment configuration and MUST NOT be
  hard-coded, including Zoho data-center URLs, tenant identifiers, endpoints, or feature flags.
  Rationale: environment-driven configuration supports safe deployment across tenants and
  regions.
- Services MUST emit structured logs with a correlation ID propagated across HTTP requests,
  webhook jobs, queue work, and adapter calls. Rationale: consistent correlation enables
  end-to-end diagnosis without exposing payloads.
- Logs MUST NOT contain message content, access tokens, refresh tokens, phone numbers, contact
  details, or other personally identifiable information. Rationale: data minimization reduces
  privacy and breach impact.
- Public APIs MUST use explicit versioning; every breaking change MUST include a migration note,
  compatibility impact, and rollout or rollback path. Rationale: consumers need predictable
  evolution and a safe transition plan.

### IV. Security and Tenant Isolation (NON-NEGOTIABLE)

- Secrets, access tokens, refresh tokens, signing keys, and client credentials MUST NOT appear
  in widget or admin frontend code, browser storage, source control, or client-visible responses.
  Rationale: browser-delivered material cannot be treated as secret.
- All WhatsApp, Zoho, and other privileged third-party calls MUST originate from the backend
  through the approved adapter. Rationale: centralized server-side enforcement protects
  credentials and authorization boundaries.
- Every database query, cache key, object-storage path, queue message, and background job MUST
  carry and enforce a validated tenant identifier. Rationale: isolation at every storage and
  processing boundary prevents cross-tenant data exposure.
- Tests MUST attempt cross-tenant reads, writes, replay, and identifier substitution and MUST
  prove denial. Rationale: adversarial isolation tests verify the boundary rather than assuming
  it.
- Every webhook request MUST have its provider signature and freshness validated before its
  payload is accepted or queued. Rationale: verification blocks forged and replayed events.
- OAuth grants MUST request only the scopes required by documented product behavior and MUST be
  reviewed whenever capabilities change. Rationale: least privilege limits compromise impact.
- Every admin action that reads sensitive operational data or changes tenant, user, integration,
  credential, or policy state MUST create an immutable audit event with actor, tenant, action,
  target, outcome, correlation ID, and timestamp. Rationale: complete auditability supports
  investigation and accountability.

### V. Reliable, Rate-Aware Processing

- Webhook endpoints MUST authenticate, validate, durably enqueue, and acknowledge requests
  without performing nonessential synchronous work. Rationale: short request paths avoid
  provider timeouts and retry storms.
- Webhook processing MUST be asynchronous and MUST define bounded retries with backoff, jitter,
  idempotency keys, and dead-letter handling. Rationale: explicit failure mechanics prevent data
  loss and uncontrolled retry loops.
- Queue workers MUST record retry exhaustion and dead-letter disposition as structured,
  alertable events without logging message content or PII. Rationale: operators need actionable
  failure signals that preserve privacy.
- WhatsApp and Zoho adapters MUST enforce documented provider rate limits using throttling,
  bounded concurrency, retry-after handling, and backpressure. Rationale: rate-aware clients
  remain reliable under load and avoid amplifying provider failures.
- Plans MUST define measurable latency, throughput, and failure-recovery targets for every new
  externally triggered flow. Rationale: explicit service objectives make performance and
  reliability testable.

## Engineering Constraints

- Boundary interfaces MUST use explicit, statically typed request, response, error, and event
  models. Rationale: typed contracts expose integration mistakes before runtime.
- Input received from browsers, webhooks, Zoho, WhatsApp, queues, and configuration MUST be
  validated at the boundary before business logic uses it. Rationale: untrusted data must not
  cross into trusted application state unchecked.
- Adapter schema versions, relevant API versions, and behavioral assumptions MUST be recorded in
  the plan and contract tests. Rationale: pinned assumptions make upstream changes detectable.
- Secrets MUST come from an approved secret store or protected environment injection and MUST be
  rotatable without rebuilding frontend assets. Rationale: operational rotation must not require
  exposing or recompiling credentials.
- Performance optimizations SHOULD follow measurement; speculative optimization MUST include a
  documented constraint or benchmark that justifies it. Rationale: evidence-led optimization
  protects clarity while addressing real bottlenecks.

## Delivery and Review Gates

- Every implementation plan MUST contain a Constitution Compliance section that evaluates each
  applicable MUST and records justified exceptions to SHOULD rules. Rationale: governance must be
  checked before implementation decisions become costly.
- Pull requests MUST include specification links, acceptance-criterion-to-test mapping, CI
  results, security and tenant-isolation impact, and migration notes when applicable. Rationale:
  reviewers need the evidence required to approve a change responsibly.
- A merge MUST be blocked by formatting, linting, typing, test, contract, integration, coverage,
  or security gate failures and by any warning emitted from those gates. Rationale: mandatory
  controls are ineffective if they can be merged around.
- Changes to webhook behavior MUST demonstrate idempotency, retry, duplicate, and out-of-order
  coverage before release. Rationale: these delivery conditions are part of the supported
  contract, not exceptional cases.
- Any approved SHOULD exception MUST identify an owner, rationale, risk, and review date.
  Rationale: time-bounded exceptions prevent guidance from silently becoming permanent debt.

## Governance

- This constitution MUST override conflicting project practices, plans, conventions, and local
  preferences. Rationale: one hierarchy prevents ambiguous governance.
- Amendments MUST document the rationale, affected principles, migration impact, approval record,
  and an appropriate semantic version bump. Rationale: governance changes require an auditable
  explanation and predictable change signal.
- Constitution versions MUST follow semantic versioning: MAJOR for backward-incompatible removal
  or redefinition of governance, MINOR for a new principle or materially expanded obligation, and
  PATCH for non-semantic clarification. Rationale: version numbers communicate governance impact.
- Every plan and pull-request review MUST include an explicit constitution compliance check, and
  unresolved MUST violations MUST block approval. Rationale: recurring verification turns written
  principles into enforceable practice.
- Compliance evidence SHOULD be automated in CI wherever a deterministic check is possible;
  manual checks MUST identify the reviewer and evidence examined. Rationale: automation improves
  consistency while recorded manual review covers contextual rules.

**Version**: 1.0.0 | **Ratified**: 2026-10-10 | **Last Amended**: 2026-10-10
