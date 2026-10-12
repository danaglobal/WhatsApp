# Feature Specification: Admin Console v1

**Feature Branch**: `N/A - no branch hook configured`

**Created**: 2026-10-10

**Status**: Draft

**Input**: User description: "Build the first standalone Admin Console for tenant onboarding,
WhatsApp number connection and coexistence sync, template management, conversations, operational
health, audit history, and platform-level tenant oversight."

## Clarifications

### Session 2026-10-10

- Q: Should tenant Admins be able to intentionally disconnect a WhatsApp number in this first
  release? → A: Admins can disconnect after confirmation; historical data remains read-only and
  the number can later reconnect.
- Q: While a tenant is suspended, how should incoming WhatsApp messages and status events be
  handled? → A: Retain and process inbound events; hide operational access until the tenant is
  restored.
- Q: Should tenant Admins be able to revoke an existing user's access in this first release? → A:
  Phase one has exactly one tenant Admin; tenant user invitations, additional users, the Agent
  role, and tenant-level user management are out of scope.
- Q: What retention policy should phase one apply to tenant contacts, messages, media, and audit
  records? → A: Active-tenant history is retained; tenant content is deleted within 30 days of
  closure; non-sensitive audit records are retained for 24 months.
- Q: What maximum pilot workload should phase one support without user-visible degradation? → A:
  Up to 250 tenants, 25 linked numbers per tenant, and 1 million messages per tenant per month;
  message history retains tenant and number identity for future cross-number CRM search.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Establish Tenant Access (Priority: P1)

A platform super admin creates a business tenant and invites its first administrator. The invited
administrator sets a password, signs in, and signs out. Each phase-one tenant has exactly one
Admin, who sees only that tenant's features and data, and an inactive session expires automatically.

**Why this priority**: Secure tenant access is the entry point for every other console capability
and is required to onboard a pilot business.

**Independent Test**: Create one tenant, redeem its first-Admin invitation, sign in and out, verify
that no tenant user-management or invitation action is available, and verify tenant isolation and
inactivity expiry without connecting a WhatsApp number.

**Acceptance Scenarios**:

1. **Given** an authenticated platform super admin, **When** the super admin creates a tenant with
   an initial administrator email, **Then** the tenant is created and a single-use invitation is
   issued to that email.
2. **Given** a valid unused invitation, **When** the recipient sets a compliant password, **Then**
   the account is activated and the recipient can sign in to the correct tenant.
3. **Given** an authenticated tenant Admin, **When** the Admin uses the phase-one console,
   **Then** no control or endpoint permits inviting, adding, removing, deactivating, reactivating,
   or changing the role of another tenant user.
4. **Given** a signed-in user whose session has been inactive for the configured timeout, **When**
   the user next interacts with the console, **Then** the user must sign in again.
5. **Given** Admins from two tenants, **When** either Admin searches, browses, or directly requests
   the other tenant's data, **Then** no cross-tenant data is returned or changed.

---

### User Story 2 - Connect WhatsApp Numbers (Priority: P1)

A tenant Admin opens the WhatsApp page and connects either an existing WhatsApp Business App
number in coexistence mode or a new API-only number. The Admin can connect multiple numbers and
inspect the business-visible identity, quality, capacity, type, connection date, and health of
each connection.

**Why this priority**: A working number connection is required for synchronization, templates,
messaging, pilot onboarding, and Meta App Review demonstrations.

**Independent Test**: Starting from a tenant with no connected numbers, complete one coexistence
connection and one API-only connection, verify their displayed metadata, then exercise cancellation,
failure, retry, and duplicate-number handling.

**Acceptance Scenarios**:

1. **Given** a tenant with no connection, **When** an Admin opens the WhatsApp page, **Then** the
   page shows "No account connected" and an available "Connect WhatsApp" action.
2. **Given** an Admin starts connection, **When** the Admin completes provider signup for an
   existing Business App number, **Then** the number is saved as a coexistence connection and is
   visible in that tenant's number list.
3. **Given** an Admin starts connection, **When** the Admin registers a new number for API-only
   use, **Then** the number is saved as an API-only connection and is visible in the list.
4. **Given** a connected number, **When** its summary or details are viewed, **Then** the display
   number, display name, display-name status, quality rating, messaging-limit tier, connection
   type, connection date, and connection status are shown.
5. **Given** signup is cancelled or fails, **When** the Admin returns to the console, **Then** a
   clear outcome and retry action are shown without creating a usable duplicate connection.
6. **Given** a number is already connected to a tenant, **When** any Admin attempts to connect it
   again, **Then** the duplicate is rejected safely and the existing tenant's identity is not
   disclosed to another tenant.
7. **Given** any completed, cancelled, or failed connection attempt, **When** support history is
   reviewed, **Then** the attempt, tenant, actor, time, outcome, and non-sensitive reason are present.
8. **Given** a connected number with historical data, **When** an Admin confirms disconnection,
   **Then** active messaging stops, retained contacts and conversations become read-only, and the
   number remains eligible for a later reconnection.

---

### User Story 3 - Synchronize Coexistence Data (Priority: P1)

Immediately after a coexistence number is connected, the Admin can see automatic contact and
message-history synchronization progress. When synchronization completes, imported contacts and
past messages are available in Conversations. If the business declines history sharing, the
console clearly distinguishes that choice from a failure.

**Why this priority**: Coexistence customers need their existing context in the console quickly,
and the provider permits the onboarding history request only once and within a limited period.

**Independent Test**: Connect a coexistence number in a controlled account, verify that both sync
requests are initiated automatically once, observe each progress state, and confirm imported data
or the declined-history state in Conversations.

**Acceptance Scenarios**:

1. **Given** a newly completed coexistence connection, **When** connection finalization succeeds,
   **Then** contact sync and all eligible history-sync phases are requested automatically.
2. **Given** an eligible coexistence connection, **When** its details are viewed during sync,
   **Then** separate contact and per-history-phase progress states are visible.
3. **Given** contacts and history have synchronized, **When** an authorized user opens
   Conversations, **Then** the imported contacts and messages appear under the connected number.
4. **Given** the business declines to share history, **When** the number details are viewed,
   **Then** the console shows "History sharing declined" rather than failed or pending.
5. **Given** a sync request was already accepted, declined, or completed, **When** a retry or
   duplicate event is received, **Then** the one-time request is not submitted again.
6. **Given** the 24-hour onboarding eligibility period is approaching or has expired without a
   successful request, **When** an Admin views the number, **Then** the risk or expired state and
   available recovery guidance are clear.

---

### User Story 4 - Manage Message Templates (Priority: P1)

An Admin chooses a connected number and reviews its account's templates. The Admin can compose a
Marketing, Utility, or Authentication template in English or Arabic with supported optional
content, preview it in the correct reading direction, submit it, and see provider approval or
rejection updates without refreshing the page.

**Why this priority**: Approved templates are required to initiate conversations and to complete
the template-creation Meta App Review video.

**Independent Test**: For a selected number, list existing templates, create valid English and
Arabic examples, verify preview and variable validation, submit one, and observe an automatic
approved or rejected status update.

**Acceptance Scenarios**:

1. **Given** a selected connected number, **When** an Admin opens Templates, **Then** all templates
   for that number's business account show name, category, language, current status, and any
   rejection reason.
2. **Given** an Admin creates a template, **When** required fields and valid numbered variables
   with sample values are entered, **Then** the console permits submission.
3. **Given** a template contains an optional header, footer, or buttons, **When** the Admin edits
   the template, **Then** the preview represents the chosen header type, footer, and no more than
   three supported buttons.
4. **Given** Arabic is selected, **When** the preview is displayed, **Then** message content and
   layout use right-to-left presentation while controls remain understandable.
5. **Given** any template variable lacks a sample value, **When** submission is attempted, **Then**
   submission is blocked and each incomplete variable is identified.
6. **Given** a template was submitted, **When** its review state changes, **Then** the list updates
   to pending, approved, rejected, paused, or disabled without manual refresh and shows the
   rejection reason when supplied.

---

### User Story 5 - Exchange Messages (Priority: P1)

The tenant Admin selects a connected number, browses recent conversations, opens a chronological
thread, receives live updates, and sends an allowed response. Free text is available only within
the active customer-service window; otherwise the user selects an approved template and supplies
all variables. A new conversation starts with an approved template.

**Why this priority**: Two-way messaging is the core pilot workflow and is required for the
send-message Meta App Review video.

**Independent Test**: Receive a customer message, exchange messages through the console and phone
app, verify ordering, source and delivery statuses, let the service window close, and start a new
thread with an approved template.

**Acceptance Scenarios**:

1. **Given** a selected connected number with conversations, **When** Conversations opens,
   **Then** threads appear newest first with contact name or number, last-message preview, and
   unread state.
2. **Given** a conversation is opened, **When** its messages load, **Then** the complete available
   thread appears chronologically and each message shows direction, time, delivery status, source,
   and a failure reason when applicable.
3. **Given** a new message or status event arrives, **When** the user remains on the relevant list
   or thread, **Then** the visible content updates without a manual refresh.
4. **Given** a live customer message opened the 24-hour service window, **When** the user composes
   a reply before expiry, **Then** free-text sending is enabled and the remaining window time is
   shown.
5. **Given** no service window is open, **When** the user opens the composer, **Then** free text is
   disabled and only approved templates valid for the selected number can be sent.
6. **Given** an approved template with variables, **When** the user leaves any required value
   empty, **Then** sending is blocked and the missing values are identified.
7. **Given** a phone number with no existing thread, **When** the user supplies a valid number and
   sends an approved template, **Then** a new conversation is created and displayed.
8. **Given** inbound image, document, audio, or video content, **When** the thread is viewed,
   **Then** the media type, safe preview or download action, sender, time, and status are displayed.
9. **Given** a business-originated phone-app message is mirrored, **When** it appears in the
   thread, **Then** its source is labeled "Phone app"; console messages are labeled "Console" and
   inbound customer messages are labeled "Customer".

---

### User Story 6 - Respond to Connection Health (Priority: P2)

A tenant Admin can recognize when a number is disconnected, understand the reported reason, and
follow guidance to reconnect it. All tenant Admins see a prominent warning while any applicable
number remains disconnected.

**Why this priority**: Clear health information reduces message disruption and support effort,
though the primary review and pilot flows can be demonstrated before advanced health handling.

**Independent Test**: Simulate disconnection for a connected number and verify its status, reason,
tenant-wide Admin banner, reconnect guidance, and recovery when the number reconnects.

**Acceptance Scenarios**:

1. **Given** a number becomes disconnected, **When** an Admin views any console page, **Then** a
   tenant-scoped banner identifies the affected number without exposing it to other tenants.
2. **Given** a disconnected number, **When** its details are opened, **Then** the reported reason
   and actionable reconnect instructions are shown.
3. **Given** the connection returns to a healthy state, **When** status is refreshed, **Then** the
   disconnected indication and banner clear automatically.

---

### User Story 7 - Review Administrative Activity (Priority: P2)

A tenant Admin reviews an immutable history of security-sensitive and business-significant
actions performed in that tenant, including access, invitations, number changes, template
creation, and message sending.

**Why this priority**: Pilot support and accountability require a reliable record of who performed
administrative and messaging actions.

**Independent Test**: Perform the applicable auditable actions as the platform super admin and the
tenant Admin, then verify that the tenant Admin sees accurate actor, action, target, outcome, and
time while another tenant does not gain access.

**Acceptance Scenarios**:

1. **Given** an auditable action occurs, **When** it completes or fails, **Then** an audit entry
   records who, what, when, tenant, target, and outcome without recording message content or secrets.
2. **Given** a tenant Admin opens the audit log, **When** entries are listed, **Then** only that
   tenant's entries are visible in newest-first order.
3. **Given** an Admin from another tenant requests an audit entry, **When** authorization is
   evaluated, **Then** access is denied without confirming whether the entry exists.

---

### User Story 8 - Oversee Tenants (Priority: P2)

A platform super admin lists tenants, reviews each tenant's connected-number statuses, and suspends
a tenant when necessary without accessing tenant message content.

**Why this priority**: Platform operations need basic onboarding and risk controls for pilot
customers while respecting tenant privacy.

**Independent Test**: Create two tenants, connect numbers, verify their summaries as a super admin,
suspend one tenant, and confirm its users lose operational access while the other tenant is
unaffected.

**Acceptance Scenarios**:

1. **Given** a platform super admin, **When** the tenant list is opened, **Then** all tenants and
   their lifecycle status are visible without message or contact content.
2. **Given** a tenant is selected, **When** its operational summary is viewed, **Then** its
   connected numbers and connection statuses are shown.
3. **Given** an active tenant, **When** the super admin confirms suspension, **Then** tenant users
   can no longer perform operational actions, inbound events continue to be validated, retained,
   and processed without tenant-Admin access, and the suspension is audited.
4. **Given** inbound events accumulated while a tenant was suspended, **When** the super admin
   restores that tenant, **Then** the tenant Admin can access the resulting current history
   without duplicate messages or status regressions.

### Edge Cases

- A phone number already connected to the same tenant is not duplicated; a number connected to a
  different tenant is rejected without revealing the owning tenant.
- Duplicate webhook events do not duplicate messages, status transitions, contacts, templates,
  audit records, or user-visible side effects.
- Out-of-order events are reconciled into a valid final state; an older event does not overwrite a
  newer message, template, sync, or connection state.
- A message received for a disconnected or formerly connected number is retained for safe
  reconciliation, is not exposed to the wrong tenant, and raises an operationally visible state.
- Disconnecting and later reconnecting the same number preserves one continuous tenant-owned
  history rather than creating a duplicate connection or conversation set.
- Incoming image, document, audio, and video messages display a safe representation and a clear
  unavailable state when media retrieval fails or expires.
- Template submission and sending are blocked when a required variable or sample value is empty.
- Imported historical customer messages and messages sent before the business completed connection
  do not open a current customer-service window; only an eligible live inbound message does.
- Messages sent from WhatsApp for Windows may not mirror into coexistence history; contextual help
  communicates this known limitation without implying the console lost a received event.
- A user opening the same conversation in multiple tabs sees consistent unread and message states.
- A status or message event received during tenant suspension is validated, retained, and processed
  into the correct tenant history, but remains inaccessible to the tenant Admin until restoration.
- Suspending a tenant does not start content deletion; only tenant closure starts the 30-day
  deletion window, and restoring a suspended tenant exposes the continuously retained history.
- The same contact may message multiple linked numbers; each message remains associated with its
  receiving or sending number so future tenant-wide search can distinguish those histories.

## Requirements *(mandatory)*

### Functional Requirements

#### Tenant Access and Roles

- **FR-001**: The platform MUST allow a super admin to create a tenant with a unique identity,
  display name, lifecycle status, and first Admin email.
- **FR-002**: The system MUST issue single-use, expiring invitations bound to one tenant, email,
  and role.
- **FR-003**: An invited user MUST be able to set a compliant password and activate the account
  only while the invitation is valid and unused.
- **FR-004**: Activated users MUST be able to sign in and sign out.
- **FR-005**: Authenticated sessions MUST expire after the configured period of inactivity.
- **FR-006**: Each tenant MUST have exactly one active tenant Admin in phase one, created through
  the platform super admin's initial invitation flow.
- **FR-007**: Tenant Admins MUST NOT be able to invite, add, remove, deactivate, reactivate, or
  change the role of tenant users; additional tenant users and the Agent role are not available in
  this release.
- **FR-008**: Every tenant-scoped read, write, search, list, event, and export MUST enforce the
  authenticated tenant boundary.
- **FR-009**: Suspended tenants MUST be prevented from initiating operational actions until
  restored by a super admin; inbound events MUST continue to be validated, retained, and processed
  but MUST remain inaccessible to the tenant Admin during suspension.

#### WhatsApp Connections

- **FR-010**: The WhatsApp page MUST show an explicit empty state and connection action when a
  tenant has no connected account.
- **FR-011**: Tenant Admins MUST be able to launch and complete the provider's embedded signup
  journey without entering provider credentials into the console itself.
- **FR-012**: The connection journey MUST support both an existing Business App number in
  coexistence mode and a new API-only number.
- **FR-013**: A tenant MUST be able to connect and select among multiple numbers.
- **FR-014**: Each connected number MUST display its phone number, display name, display-name
  status, quality rating, messaging-limit tier, connection type, connection date, and connection
  status.
- **FR-015**: Cancelled and failed signup outcomes MUST show a non-sensitive reason, preserve a
  recoverable state, and offer retry.
- **FR-016**: Every connection attempt MUST record tenant, actor, number when safely available,
  start time, completion time, outcome, and non-sensitive failure reason for support.
- **FR-017**: The system MUST prevent the same provider phone number from becoming an active
  duplicate connection, including across tenant boundaries.
- **FR-018**: A cross-tenant duplicate rejection MUST NOT reveal the identity or data of the tenant
  that owns the existing connection.

#### Coexistence Synchronization

- **FR-019**: A completed coexistence connection MUST automatically initiate eligible contact sync
  and message-history sync without requiring a second Admin action.
- **FR-020**: The initial sync requests MUST be made within the provider's 24-hour onboarding
  eligibility window and MUST NOT be repeated after a terminal accepted, declined, or completed
  outcome.
- **FR-021**: The number detail view MUST show separate contact-sync status and each message-history
  phase with pending, in-progress, completed, failed, unavailable, or declined states as applicable.
- **FR-022**: A business decision not to share history MUST be shown as declined, distinct from a
  system failure.
- **FR-023**: Successfully synchronized contacts and messages MUST become visible in Conversations
  for the owning tenant's Admin.
- **FR-024**: Duplicate, retried, or out-of-order sync events MUST converge on one correct state
  without duplicating contacts or messages.
- **FR-025**: The console MUST warn Admins when the one-time sync eligibility window is at risk or
  has expired and MUST state any available recovery path.

#### Templates

- **FR-026**: For the selected number, Templates MUST list every available template with name,
  category, language, status, and provider-supplied rejection reason.
- **FR-027**: Template status MUST distinguish pending, approved, rejected, paused, and disabled.
- **FR-028**: Tenant Admins MUST be able to create Marketing, Utility, and Authentication templates
  in at least English and Arabic.
- **FR-029**: Template composition MUST support an optional text or image header, required body,
  optional footer, and up to three quick-reply, URL, or phone buttons.
- **FR-030**: Body variables MUST be numbered consistently and MUST each have a non-empty sample
  value before submission.
- **FR-031**: The console MUST provide a live preview of composed content and MUST render Arabic
  content right-to-left.
- **FR-032**: Invalid names, unsupported combinations, excess buttons, missing required content,
  and incomplete variables MUST be identified before submission.
- **FR-033**: Submitted template status and rejection information MUST update without manual page
  refresh when the provider reports a change.
- **FR-034**: Only templates associated with the selected number's business account MUST be listed
  or offered for sending.

#### Conversations and Messaging

- **FR-035**: Conversations for the selected number MUST be listed newest first with contact name
  or number, last-message preview, last activity time, and unread indication.
- **FR-036**: Opening a conversation MUST show its available messages in chronological order.
- **FR-037**: Every message MUST show direction, time, current delivery status, source, and a
  non-sensitive failure reason when failed.
- **FR-038**: Message source MUST distinguish Console, Phone app, and Customer.
- **FR-039**: New messages, conversation order, unread state, and delivery-status changes MUST
  update without a manual refresh.
- **FR-040**: The tenant Admin MUST be able to send free text only while an eligible
  24-hour customer-service window is open.
- **FR-041**: The composer MUST show the remaining service-window time and MUST disable free text
  when the window closes.
- **FR-042**: When no service window is open, the composer MUST permit only an approved template
  available to the selected number.
- **FR-043**: Sending a template MUST require a non-empty value for every required variable.
- **FR-044**: The tenant Admin MUST be able to start a new conversation by entering a valid phone
  number and sending an approved template.
- **FR-045**: Incoming image, document, audio, and video messages MUST display their media type,
  relevant metadata, and a safe view or download action when the media is available.
- **FR-046**: Historical or pre-connection customer messages MUST NOT establish a current
  customer-service window.
- **FR-047**: Duplicate message or status events MUST NOT create duplicate messages or regress a
  message from a later delivery state to an earlier state.
- **FR-048**: Messages received for a disconnected number MUST remain associated with the correct
  tenant and number and MUST surface a support-visible reconciliation state.
- **FR-049**: Contextual help MUST state that messages sent through WhatsApp for Windows may not
  mirror into coexistence history.

#### Health, Audit, and Platform Oversight

- **FR-050**: A disconnected number MUST show Disconnected, the available reason, and actionable
  reconnection guidance.
- **FR-051**: Tenant Admins MUST see a persistent tenant-scoped banner while any connected number
  requiring attention remains disconnected.
- **FR-052**: Connection-health indications MUST clear automatically after confirmed recovery.
- **FR-053**: Sign-in, invitation, number connection or disconnection, template creation, message
  send, tenant creation, and tenant suspension attempts MUST produce immutable audit entries for
  both success and failure.
- **FR-054**: Each audit entry MUST identify tenant, actor, action, target, timestamp, outcome, and
  a correlation reference while excluding message content, credentials, tokens, and unnecessary PII.
- **FR-055**: Tenant Admins MUST be able to view only their tenant's audit entries; Agents MUST NOT
  exist as tenant users in this release.
- **FR-056**: Platform super admins MUST be able to list tenants and view tenant lifecycle status
  and connected-number health without access to message or contact content.
- **FR-057**: Platform super admins MUST be able to suspend a tenant through an explicit confirmed
  action, and the action MUST be audited.
- **FR-058**: All user-facing failures MUST provide a clear outcome and safe next action without
  exposing secrets, provider credentials, internal identifiers, or another tenant's existence.
- **FR-059**: Tenant Admins MUST be able to disconnect a connected number through an explicit
  confirmation; the disconnected number's retained contacts and conversations MUST become
  read-only, and reconnecting that number MUST restore the same tenant-owned history.
- **FR-060**: Contacts, messages, and media MUST be retained while a tenant is active or suspended;
  those tenant-content records MUST be deleted within 30 days after tenant closure, while
  non-sensitive audit records MUST be retained for 24 months from each audit event.
- **FR-061**: The phase-one service MUST support up to 250 tenants, 25 linked numbers per tenant,
  and 1 million messages per tenant per month without violating the measurable response and event
  visibility outcomes in this specification.
- **FR-062**: Every contact, conversation, and message MUST retain its owning tenant and linked
  number association so a future CRM experience can search history across all of a tenant's linked
  numbers without crossing tenant boundaries.

### Key Entities *(include if feature involves data)*

- **Tenant**: A business boundary containing its single phase-one Admin, numbers, contacts,
  conversations, templates, connection attempts, and audit entries; has an active, suspended, or
  closed lifecycle state, with closure starting the content-deletion window.
- **User**: A platform super admin or the single Admin identity for one tenant, with email, account
  state, role, tenant membership where applicable, invitation history, and session activity.
- **Invitation**: A single-use, expiring offer for one email to join one tenant with one role.
- **WhatsApp Connection**: A tenant-owned provider number and business account relationship with
  identity metadata, connection type, health, quality, messaging tier, connection timestamps, and
  a lifecycle that preserves tenant history across disconnect and reconnect.
- **Connection Attempt**: A support record of an attempted signup journey and its outcome without
  stored provider credentials.
- **Sync Run**: The one-time coexistence contact or history operation with phase, progress, consent
  outcome, eligibility window, and terminal result.
- **Contact**: A tenant- and number-scoped conversation participant identified by phone number and
  available display information; the same person may have distinct conversation context across
  multiple linked numbers.
- **Conversation**: A tenant-, number-, and contact-scoped thread with last activity, unread state,
  and customer-service window state.
- **Message**: A conversation item with provider identity, direction, source, time, content type,
  delivery progression, failure information, optional media reference, and immutable owning-tenant
  and linked-number associations.
- **Template**: A business-account message definition with name, category, language, components,
  variables, samples, buttons, review state, and optional rejection reason.
- **Audit Entry**: An immutable tenant-scoped record of actor, action, target, time, outcome, and
  correlation reference with sensitive payload data excluded.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A first-time tenant Admin can redeem an invitation, sign in, and reach the empty
  WhatsApp connection state in under 5 minutes without support assistance.
- **SC-002**: In normal provider conditions, a coexistence Admin sees synchronized contacts and
  available history begin appearing within 15 minutes of completing signup, with visible progress
  throughout the wait.
- **SC-003**: At least 95% of normal-condition messages sent from the console reach the WhatsApp
  recipient, and phone-app messages appear in the console with the correct source label, within
  5 seconds of the corresponding provider event.
- **SC-004**: At least 95% of provider-reported template status changes appear in the open template
  list within 30 seconds without manual refresh.
- **SC-005**: Automated isolation tests block 100% of attempted cross-tenant reads, writes,
  identifier substitutions, event replays, and direct record requests.
- **SC-006**: Replaying any supported webhook event at least three times produces exactly one
  user-visible message, template transition, sync transition, or connection transition.
- **SC-007**: The send-message and create-template Meta App Review videos can each be recorded from
  start to finish entirely within the Admin Console without database edits, hidden tools, or
  manual refresh.
- **SC-008**: At least 90% of pilot users complete connection, template submission, and permitted
  message sending on their first attempt using only in-product guidance.
- **SC-009**: Every supported Admin action can be located in the correct tenant's audit history by
  actor and time, and no audit entry contains message content or credentials.
- **SC-010**: A disconnected-number test alerts all tenant Admins within 30 seconds of the reported
  health change and removes the alert within 30 seconds of confirmed recovery.
- **SC-011**: Retention verification confirms that active and suspended tenants keep their history,
  closed-tenant content is absent no later than 30 days after closure, and audit records remain
  available for 24 months without message content or credentials.
- **SC-012**: At the target workload of 250 tenants, 25 linked numbers per tenant, and 1 million
  messages per tenant per month, at least 95% of conversation lists and threads become usable
  within 2 seconds and the message and status visibility targets in SC-003, SC-004, and SC-010
  remain satisfied.

## Assumptions

- Invitation links expire after 72 hours and cannot be reused; a replacement initial-Admin
  invitation may be issued only by a platform super admin.
- Sessions expire after 30 minutes without user activity; the plan may make this value configurable
  without changing the user-visible rule that inactive sessions expire.
- The single tenant Admin has all tenant connection, template, conversation, messaging, and audit
  permissions in this release but has no tenant user-management capability.
- The provider determines number eligibility, display-name review, quality rating, messaging tier,
  template policy, template review timing, coexistence eligibility, and availability of historical
  data; the console presents provider outcomes accurately.
- "Normal conditions" means the user's network and relevant provider services are available and
  are not actively throttling, degraded, or under scheduled maintenance.
- The monthly message target includes imported and live inbound and outbound messages; because
  active-tenant history is retained, capacity planning accounts for cumulative storage growth.
- A current service window begins only from an eligible live inbound customer message received
  after the business connection is active; imported history does not open the window.
- Received media is supported in v1, while outbound media outside approved template components is
  not part of this feature.
- Platform super admins may view tenant and connection operational metadata but not tenant contact
  or message content as part of routine oversight.
- Tenant closure is distinct from suspension and is initiated through an authorized platform
  offboarding process; a closure interface is not required in this phase-one console.
- English is the default console language; Arabic right-to-left behavior is required for template
  content preview, not for localization of the entire console in this release.

## Dependencies

- Access to a provider application configured for embedded signup, coexistence onboarding,
  template management, message delivery, event delivery, and App Review testing.
- Pilot test numbers and business accounts eligible for both coexistence and API-only onboarding.
- Reliable email delivery for invitations and a documented support path for failed delivery.
- Provider approval and behavior may affect when a template reaches an approved or rejected state;
  the console is responsible for reflecting reported state, not controlling review duration.

## Out of Scope

- Zoho CRM integration, Zoho widgets, or any embedded CRM experience.
- Automations, broadcasts, chatbots, artificial intelligence, analytics, and billing.
- Group chats, voice calls, and video calls.
- General outbound media messaging outside the template components explicitly described above.
- Full-console Arabic localization beyond right-to-left Arabic template content and preview.
- Additional tenant users, tenant-level invitations or user administration, and the Agent role.
- Zoho CRM cross-number history search remains a future feature; phase one preserves the tenant and
  linked-number associations required for it but does not expose the CRM widget or that search UI.
