# Contract: Browser Realtime Events

Socket.IO uses the same authenticated server session as HTTP. Authorization occurs on connection and on every tenant room join. A browser can join only its own tenant room; SuperAdmin receives platform-health events, not message content.

## Transport

- Namespace: `/realtime/v1`
- Preferred transport: WebSocket only. If polling is enabled later, the deployment MUST add sticky sessions.
- Horizontal fan-out: Socket.IO Redis adapter.
- Redis delivery is ephemeral. Every event carries durable entity IDs and a monotonically comparable version; clients invalidate/refetch TanStack Query data after reconnect or a detected gap.
- Logout, session revocation, tenant suspension, or closure disconnects affected sockets and clears browser query state.

## Envelope

```json
{
  "eventId": "019...",
  "eventType": "message.created",
  "eventVersion": 1,
  "tenantId": "019...",
  "entityId": "019...",
  "connectionId": "019...",
  "occurredAt": "2026-10-10T12:00:00Z",
  "correlationId": "019...",
  "data": {}
}
```

The server derives `tenantId`; it never trusts a client-supplied tenant identifier. Payloads contain only the minimum data required to update a visible screen. They never contain tokens, credentials, raw webhook payloads, signed media URLs, or hidden tenant content.

## Event types

| Event | Minimum data | Client behavior |
|---|---|---|
| `message.created` | conversation ID, message ID, direction, status | Insert if current conversation is open; invalidate lists |
| `message.status_changed` | message ID, status, version | Apply only newer version; invalidate on gap |
| `conversation.changed` | conversation ID, last-message time, unread count | Reorder/invalidate tenant conversation page |
| `connection.changed` | connection ID, status, quality | Refresh connection/detail health |
| `sync.progressed` | sync ID, type, state, completed/total units | Update connection sync progress |
| `template.changed` | template ID, status | Refresh template list/detail |
| `tenant.access_changed` | tenant status | Clear protected data and redirect when suspended/closed |
| `platform.health_changed` | tenant/connection IDs and health category | SuperAdmin-only health refresh |

## Client-to-server events

Phase one permits only connection lifecycle events (`client.ready`, heartbeat, disconnect). All domain mutations use versioned HTTP APIs with CSRF and idempotency controls.

