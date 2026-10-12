# Contract: Worker Queues and Jobs

All queues use BullMQ on Redis 7. Job payloads contain IDs and safe routing metadata, not message bodies, access tokens, authorization codes, raw webhook bodies, or signed media URLs. Workers reload authoritative state through tenant-scoped repositories.

## Standard envelope

```ts
type JobEnvelope<T extends object> = {
  version: 1;
  jobId: string;
  tenantId: string;
  correlationId: string;
  deduplicationKey: string;
  enqueuedAt: string;
  payload: T;
};
```

Producers and consumers validate this envelope with shared Zod schemas. Each handler writes its idempotency result before acknowledging success.

## Queues

| Queue | Job types | Required idempotency/effect |
|---|---|---|
| `webhook-processing` | `dispatch-delivery`, `normalize-message`, `apply-status`, `apply-echo`, `apply-account-update` | Delivery/event digest and `wamid`; monotonic reducers |
| `history-sync` | `request-sync`, `normalize-history-chunk`, `finalize-sync` | One successful request per connection/type; phase plus chunk order |
| `media-download` | `download-media`, `scan-media`, `expire-media` | Provider media ID plus message; generated object key and checksum |
| `template-sync` | `submit-template`, `apply-template-status`, `reconcile-templates` | Tenant/WABA/name/language and provider event digest |
| `outbound-send` | `send-message`, `reconcile-unknown-send` | Client idempotency key; never blind-resend an ambiguous timeout |
| `outbox-publish` | `publish-outbox`, `sweep-unpublished` | Outbox event ID |
| `retention` | `purge-closed-tenant-content`, `purge-expired-audit` | Tenant plus cutoff/run ID; resumable batches |

## Retry and dead-letter behavior

- Baseline: 8 attempts with exponential backoff from 1 second, jitter, and a 15-minute cap.
- A provider-directed retry delay or local throttle may postpone a job beyond the baseline backoff.
- Permanent validation, missing-resource, or revoked-authorization failures are classified without repeated retries.
- Exhaustion persists a `DeadLetter` row and publishes an operational alert. Failed jobs may remain briefly in BullMQ for diagnosis, but Redis is not the system of record.
- Requeue requires an authenticated, authorized, audited operator action and creates a linked job with a new job ID.
- Worker shutdown pauses intake, finishes or safely abandons the active job within the grace window, and relies on lock recovery for abandoned jobs.

## Outbound rate and ordering

Before each Meta send begins, the worker acquires a token from an atomic Redis script keyed by `meta_phone_number_id`. Coexistence connections are limited to 20 send starts per rolling second across all worker replicas. Provider- and recipient-level constraints may impose a lower effective rate. A short conversation-level mutex preserves intended ordering.

The limiter is fail-closed: if Redis coordination is unavailable, outbound sends wait rather than exceed the limit. An API timeout after request transmission marks the message `DELIVERY_UNKNOWN`; reconciliation checks later webhook/provider evidence rather than automatically sending a duplicate.

## Operational requirements

- Redis production configuration uses persistence appropriate to the host and `noeviction`; PostgreSQL outbox/reconciliation protects against queue-publication gaps.
- Concurrency, attempts, backoff, lock duration, and retention are environment-configured with safe defaults.
- Metrics are labeled with bounded categories only: queue/job type, outcome, attempt bucket, and latency. Tenant IDs, phone numbers, `wamid`, and message content are not metric labels.
- Readiness fails when required Postgres, Redis, or object-storage checks fail; liveness only reflects process health.
