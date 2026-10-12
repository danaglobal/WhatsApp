# Contract: Meta Webhook Boundary

## Public endpoints

`GET /webhooks/meta/whatsapp` implements the verification challenge using configured verify-token material. `POST /webhooks/meta/whatsapp` is the single delivery endpoint for all tenants and numbers.

The POST endpoint MUST:

1. Capture the exact request bytes before JSON parsing.
2. Compare `X-Hub-Signature-256` against an HMAC-SHA256 computed with the app secret using constant-time comparison.
3. Apply a configured, field-aware freshness policy: stale live events are rejected, while old provider timestamps are accepted only for a currently authorized history sync; a deterministic delivery digest blocks exact replay.
4. Reject a missing/invalid signature or invalid freshness context without publishing a job.
5. In one database transaction, store a `WebhookDelivery` and corresponding `OutboxEvent` using the deterministic payload/event digest.
6. Return `200` only after commit, without waiting for domain processing. A database failure produces a non-2xx response so Meta can retry.

No request header, raw payload, message text, contact name, phone number, token, or media URL may be written to ordinary logs. Logs contain correlation ID, redacted delivery ID, field type, outcome, latency, and safe error category.

## Supported field dispatch

| Meta field | Worker outcome |
|---|---|
| `messages` | Normalize inbound messages and delivery statuses; schedule media download when applicable |
| `smb_message_echoes` | Reconcile WhatsApp Business App outbound messages into the same conversation/message model |
| `history` | Store raw chunk first, then normalize in phase and `chunk_order` order |
| `smb_app_state_sync` | Apply coexistence contact/state sync idempotently |
| `account_update` | Update connection/WABA state and raise an operational event |
| `message_template_status_update` | Upsert template status using a forward-compatible status string |
| `phone_number_quality_update` | Update quality and current messaging limit fields without assuming removed legacy names |

Unknown fields and unknown enum values are retained in redacted/raw storage, counted, and acknowledged; they do not crash the dispatcher. A fixture review is required whenever `META_GRAPH_API_VERSION` changes.

## Identity and idempotency

- Resolve the tenant from trusted WABA/phone-number mappings, never from a tenant value in the incoming payload.
- Live messages, history messages, and SMB echoes converge by provider `wamid` plus connection identity.
- Status observations deduplicate by a canonical event digest and advance message status monotonically.
- History chunks deduplicate by sync run, phase, chunk order, and payload hash.
- Every handler may run more than once and in any order without duplicating user-visible messages or regressing state.

## Processing and failure contract

- Default asynchronous attempts: 8, exponential backoff beginning at 1 second with jitter, capped at 15 minutes.
- Provider `429` responses honor retry guidance and shared per-number/provider throttles.
- Schema or permanent authorization errors fail without blind retry and create a redacted `DeadLetter` record.
- Exhausted transient errors create a `DeadLetter` record and an operator-visible health signal.
- Reprocessing a dead letter is an audited action and creates a new job linked to the original failure.
- Suspended tenants continue ingestion and processing, but tenant UI/API access stays blocked until restoration.
- Closed tenants are not operationally processed; closure retention rules govern deletion.

## Embedded Signup boundary

The browser receives only public configuration values and an opaque state. It sends the short-lived authorization code and session information to the backend. The backend exchanges the code, subscribes the WABA, encrypts the resulting token, and:

- for `COEXISTENCE`, does not call phone registration and offers one-time contacts/history sync within 24 hours;
- for `CLOUD_API_ONLY`, registers the selected phone number after signup;
- records finish, cancel, error, and timeout outcomes without persisting codes or tokens in logs.

Because Meta's current pages differ on coexistence parameter naming and session-info version, the exact JS SDK options and event payload MUST pass a real sandbox fixture gate before release. Version-sensitive values remain environment configuration, not scattered constants.

## Contract fixtures

Sanitized fixtures MUST cover every supported field, multiple entries/changes, absent optional properties, unknown fields/enums, invalid signatures, duplicate delivery, duplicate `wamid`, status-before-message, history out of arrival order, echo/live convergence, retry, rate limiting, and media metadata/download expiry. Fixture provenance records the Graph API version and capture date.

