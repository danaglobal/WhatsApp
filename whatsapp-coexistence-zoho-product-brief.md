# WhatsApp Coexistence Inbox for Zoho CRM — Product Brief & Feature Catalog (v2)

> **Purpose:** single source of input for Spec Kit (`/speckit.specify`, `/speckit.plan`).
> Fill the **Keep?** column (✔ / ✘ / ?) then feed groups to `/speckit.specify` in the order of §10.
> Research date: October 2026. v2 adds: Meta onboarding path (§5), full administration catalog (groups J–R), competitor coexistence check.

---

## 1. Product Vision

A respond.io / Wati–class conversation platform that lives **inside Zoho CRM** (Web Tab + record-level widget) and shows, for every Lead/Contact phone number, **one unified WhatsApp conversation** merging:

1. Messages sent/received via the **WhatsApp Cloud API** (CRM inbox, automations, broadcasts), and
2. Messages the agent sends from the **WhatsApp Business App on their own phone** (Meta's official *Coexistence*).

Backed by a **multi-tenant SaaS backend**, a **Tenant Admin console** (customer company) and a **Platform Super-Admin console** (us).

### Positioning
- Official API only (no WhatsApp Web / QR-session tools) → no ban risk.
- Zoho-native: agents never leave the CRM.
- Coexistence-first: agents keep chatting from their phone; nothing is lost from the CRM.
- Arabic/RTL + MENA first.

---

## 2. Personas

| Persona | Description | Main surface |
|---|---|---|
| Agent | Zoho CRM user chatting with leads, partly from their phone | Web Tab inbox, record widget, WhatsApp Business App |
| Team Lead / Manager | Supervises, reassigns, reviews performance | Inbox (team view), analytics |
| Tenant Admin | Customer's admin: links numbers, maps users, builds templates & automations | Tenant Admin console |
| Platform Super Admin | Us: tenants, billing, health, support, Meta app config | Platform Admin console |

---

## 3. Glossary

| Term | Meaning |
|---|---|
| Coexistence | Same number active on WhatsApp Business App **and** Cloud API |
| Echo | `smb_message_echoes` webhook: message the business sent from the phone app |
| History sync | `history` webhook: up to 180 days of past 1:1 chats |
| CSW | 24h Customer Service Window; free-form API messages allowed inside it |
| Template | Meta-approved message, required outside the CSW |
| WABA | WhatsApp Business Account |
| Tech Provider | Meta program letting us onboard and message on behalf of client businesses |
| Solution Partner | Meta BSP (e.g. 360dialog, Twilio, Infobip) that can provide credit line/billing |
| Tenant | One customer company (one Zoho CRM org) |

---

## 4. Hard Constraints (verified)

### 4.1 Meta / Coexistence
- We must be a **Tech Provider** (or Solution Partner) and use **Embedded Signup** with session logging.
- **Embedded Signup v2 is deprecated Oct 15, 2026 → build on v4.**
- Business App **2.24.17+**; number must be on the **Business App** (not personal WhatsApp) — onboarding is one-way App → API.
- Coexistence numbers: **fixed 20 msg/s**.
- History: **180 days** text, **14 days** media, phases 0/1/2, chunked and possibly out of order.
- Contacts + history sync must start **within 24h** of onboarding and can be requested **once**.
- Business may decline history sharing (error `2593109`).
- Webhooks needed: `messages`, `history`, `smb_app_state_sync`, `smb_message_echoes`, `account_update`, template status updates.
- **Not supported:** groups, broadcast lists (read-only), app calls, catalog/status, labels, quick replies, greeting/away messages, profile edits via API, Marketing Messages Lite API.
- Phone-app messages free and outside the CSW; API messages billed and inside CSW rules.
- Disconnection only from the phone app → `account_update` / `PARTNER_REMOVED` (`PRIMARY_INACTIVITY` ≈ 14 days without opening the app, `CHANGE_NUMBER`, `USER_RE_REGISTERED` …).
- Error `131060` for first-time contacts / unsupported companions; WhatsApp for Windows & WearOS companions don't mirror.
- Country exclusions (per Zoho docs): Nigeria, South Africa. **Egypt/MENA supported.**

### 4.2 Zoho CRM
- Widgets in iframe, Embedded App JS SDK (`init`, `PageLoad`, `ZOHO.CRM.API.*`), React allowed, max 250 files, externally hosted.
- Web Tab has no record context → also need record-level widget.
- Multi data center; API credit limits per org.

### 4.3 Design implication
Each agent phone = one coexistence-onboarded number. Data model: **N numbers per tenant**, number ↔ agent/team mapping.

---

## 5. Meta Onboarding Path (how we become allowed to do this)

### 5.1 Options

| Option | What it means | Pros | Cons |
|---|---|---|---|
| **A. Independent Tech Provider** | Our own Meta app; clients add their own credit card to their WABA | Full control, no middleman fees | We handle all support/billing questions with Meta |
| **B. Tech Provider + Solution Partner** (recommended to start) | Same app, plus a "partner solution" with a BSP (360dialog, Twilio, Infobip…) | BSP handles credit line & billing, gives support, has guides for coexistence | BSP fees; dependency |
| C. Resell a BSP's platform | Use another platform's API (e.g. 360dialog Partner API) without our own app | Fastest | Least control; not really our product |

### 5.2 Step-by-step (Option A or B)

| # | Step | Owner | Typical duration |
|---|---|---|---|
| 1 | Create **Meta Business Portfolio**, enable **2FA** for admins | Us | 1 day |
| 2 | **Business Verification** (legal name, address, phone, website, documents) — start day 1 | Us | ~2–5 business days |
| 3 | Create **Meta App** (Business type) with **WhatsApp use case**; add icon, privacy policy URL, terms URL, category | Dev | 1 day |
| 4 | Build minimal demo: send message from our app + create template from our app (or use API Setup cURL & WhatsApp Manager) | Dev | 2–5 days |
| 5 | Record **2 App Review videos** (message sent & received; template created) | Dev | 1 day |
| 6 | Submit **App Review** for **Advanced access** to `whatsapp_business_messaging` and `whatsapp_business_management` | Us | days–weeks |
| 7 | (Option B) **Create partner solution** with the BSP's app ID | Us + BSP | days |
| 8 | **Access Verification** | Us | ~5 business days |
| 9 | Configure **Embedded Signup v4** with "Connect existing WhatsApp Business App" + session logging | Dev | 2–4 days |
| 10 | Subscribe webhooks (§4.1) and test coexistence with **our own** Business App number | Dev | 2–3 days |
| 11 | Pilot with 1–3 friendly clients | Us | 2–4 weeks |

**Parallel work:** steps 1–8 are paperwork; engineering (backend, Zoho widget, admin) proceeds in parallel using the Cloud API **test number** from the App Dashboard. Coexistence onboarding itself is tested once Embedded Signup is configured.

### 5.3 Risks & mitigations
- App Review rejection → follow videos precisely; reuse BSP guides.
- Business Verification fails → make website, legal name and documents match exactly.
- Embedded Signup version changes → abstract ES config behind our own onboarding service (feature I-series).

---

## 6. Competitive Landscape & Coexistence Support

| Solution | Zoho-native? | Official coexistence? | Notes |
|---|---|---|---|
| respond.io | No (HubSpot/Salesforce sync) | ✅ | Benchmark for inbox, lifecycle, workflows, AI |
| Wati | No (integrations) | ✅ | Benchmark for templates, broadcasts, chatbot builder, admin |
| WOZTELL | Yes (Zoho CRM extension) | ✅ | **Closest competitor** |
| TimelinesAI | Yes (extension) | ✅ + unofficial linked-device mode | Default Zoho mode is linked-device |
| Zoho Campaigns | Zoho | ✅ | Campaigns only, not an inbox |
| Zoho CRM native WhatsApp | Yes | ❌ not documented | Templates + workflow notifications |
| Ulgebra WABA | Yes | ❌ (asks for number without active WhatsApp) | Inbox, assign, labels |
| Spellty, Eazybe | Yes | ❌ (WhatsApp Web / Chrome ext) | Unofficial |
| Picky Assist | Yes | ⚠️ unclear | |
| Wazzup, Mottasl, 360 SMS | Yes | ❓ not verified | |
| 360dialog, YCloud, Jelou, Zavu | No | ✅ | Infra / BSPs |

---

## 7. Feature Catalog — End-User Side

Legend — **Feasibility:** ✅ supported · ⚠️ possible with limits · ❌ not possible. **Phase:** MVP · V2 · V3.

### A. Coexistence Sync Engine
| ID | Feature | Feas. | Phase | Keep? |
|---|---|---|---|---|
| A1 | Contacts sync (`smb_app_state_sync`) + live updates | ✅ | MVP | |
| A2 | History sync (180d text / 14d media) with phase & progress tracking | ✅ | MVP | |
| A3 | Graceful "history declined" state | ✅ | MVP | |
| A4 | Real-time echo mirroring of phone-app messages | ✅ | MVP | |
| A5 | Auto-start sync within 24h + alert if missed | ✅ | MVP | |
| A6 | Media download & storage (Meta URLs expire) | ✅ | MVP | |
| A7 | Group chats / app calls / app labels sync | ❌ | — | |

### B. Unified Inbox (Web Tab)
| ID | Feature | Feas. | Phase | Keep? |
|---|---|---|---|---|
| B1 | Conversation list + filters (mine, unassigned, team, number, unread, status, tag) | ✅ | MVP | |
| B2 | Source badge per message (CRM / Phone app / Automation / Broadcast) | ✅ | MVP | |
| B3 | Send text, emoji, image, document, audio, video, location | ✅ | MVP | |
| B4 | Receive all types incl. reactions, quoted replies, stickers, contacts | ✅ | MVP | |
| B5 | Edited / deleted message display | ✅ | V2 | |
| B6 | Delivery ticks + failure reasons | ✅ | MVP | |
| B7 | 24h window indicator; composer switches to template picker when closed | ✅ | MVP | |
| B8 | Internal comments & @mentions | ✅ | V2 | |
| B9 | Snippets (canned replies) with variables | ✅ | V2 | |
| B10 | Assign / reassign / close (with closing note) / snooze / reopen | ✅ | MVP | |
| B11 | Full-text search | ✅ | V2 | |
| B12 | Real-time updates + browser notifications | ✅ | MVP | |
| B13 | Send read receipts | ✅ | V2 | |
| B14 | Contact side panel with Zoho fields, owner, stage, deals | ✅ | MVP | |
| B15 | Arabic/RTL + English UI | ✅ | MVP | |
| B16 | Collision detection | ✅ | V3 | |
| B17 | Send interactive messages (buttons, lists) inside CSW | ✅ | V2 | |
| B18 | AI reply suggestions, summarize, translate | ✅ | V3 | |

### C. Zoho Record Integration
| ID | Feature | Feas. | Phase | Keep? |
|---|---|---|---|---|
| C1 | E.164 normalization & matching to Leads/Contacts (multiple phone fields) | ✅ | MVP | |
| C2 | Record-level widget with full thread | ✅ | MVP | |
| C3 | "Send WhatsApp" button on records & list views | ✅ | MVP | |
| C4 | Auto-create Lead for unknown numbers (configurable) | ✅ | MVP | |
| C5 | Multiple-match resolution | ✅ | MVP | |
| C6 | Transcript/summary to Notes or custom module | ✅ | V2 | |
| C7 | Assignee follows record owner (optional) | ✅ | V2 | |
| C8 | Events to Zoho workflows (message received, closed) | ✅ | V2 | |
| C9 | Lifecycle stages ↔ Lead Status / Deal Stage | ✅ | V2 | |
| C10 | Zoho CRM mobile app widget | ⚠️ | V3 | |

---

## 8. Feature Catalog — Administration (Tenant Admin Console)

### J. Number / Channel Administration
| ID | Feature | Feas. | Phase | Keep? |
|---|---|---|---|---|
| J1 | **Link number** via Embedded Signup v4 — coexistence ("existing Business App") flow | ✅ | MVP | |
| J2 | Link API-only (new) number | ✅ | V2 | |
| J3 | Onboarding wizard with prerequisites checklist (app version, Business App, verified portfolio) | ✅ | MVP | |
| J4 | Numbers list: display name, display-name approval status, quality rating, messaging-limit tier, throughput, coexistence flag (`is_on_biz_app`) | ✅ | MVP | |
| J5 | Number nickname, color, default team, default assignee | ✅ | MVP | |
| J6 | Number ↔ agents/teams access mapping | ✅ | MVP | |
| J7 | Per-number business hours & routing policy | ✅ | V2 | |
| J8 | Sync status panel per number (contacts, history phases, last echo received) | ✅ | MVP | |
| J9 | Disconnect detection with reason + guided re-onboarding | ✅ | MVP | |
| J10 | "Phone inactive" warning before ~14-day auto-disconnect (inferred) | ⚠️ | V2 | |
| J11 | Archive number (keep history, stop sending) | ✅ | V2 | |
| J12 | Business profile view (edits only in phone app for coexistence) | ⚠️ | V2 | |
| J13 | Billing payment-method status of the WABA | ✅ | V2 | |

### K. Template Administration
| ID | Feature | Feas. | Phase | Keep? |
|---|---|---|---|---|
| K1 | Sync existing templates from WABA | ✅ | MVP | |
| K2 | Template builder: category (Marketing / Utility / Authentication), language(s) incl. Arabic, header (text/image/video/document), body with variables, footer, buttons (quick reply, URL, call, copy code) | ✅ | MVP | |
| K3 | Live preview (LTR/RTL) and sample values | ✅ | MVP | |
| K4 | Submit to Meta + status tracking via webhook (pending/approved/rejected/paused/disabled) with rejection reason | ✅ | MVP | |
| K5 | Carousel templates | ✅ | V2 | |
| K6 | Clone / new language version / delete | ✅ | V2 | |
| K7 | **Variable mapping profiles**: template variable → Zoho field (Lead/Contact/Deal/Owner) with fallbacks | ✅ | MVP | |
| K8 | Media library for template headers | ✅ | V2 | |
| K9 | Template permissions (which teams/roles can use which template) | ✅ | V2 | |
| K10 | Template quality & performance (sent/delivered/read/clicked) | ✅ | V2 | |

### L. Automation Configuration
| ID | Feature | Feas. | Phase | Keep? |
|---|---|---|---|---|
| L1 | **CRM-triggered template automations**: when Zoho record created/updated/field changes/stage changes → send template X with mapping profile Y from number Z | ✅ | MVP | |
| L2 | Expose "Send WhatsApp template" as Zoho workflow action (custom function / our API) | ✅ | MVP | |
| L3 | Scheduled / delayed sends (e.g. reminder 1 day before appointment date field) | ✅ | V2 | |
| L4 | Keyword rules → auto reply / assign / tag / trigger flow | ✅ | V2 | |
| L5 | Default (fallback) reply | ✅ | V2 | |
| L6 | Welcome & away messages (our own; app ones don't apply to API) | ✅ | V2 | |
| L7 | Auto-assignment rules: round-robin, load-balanced, by number, by CRM owner, by team, by language | ✅ | V2 | |
| L8 | SLA & escalation rules (no reply in X min → notify/reassign) | ✅ | V2 | |
| L9 | Auto-close inactive conversations | ✅ | V2 | |
| L10 | Opt-out keywords (STOP) → update consent in Zoho + block future marketing | ✅ | MVP | |
| L11 | Visual workflow / chatbot flow builder | ✅ | V3 | |
| L12 | AI agent with knowledge sources and human hand-off | ✅ | V3 | |
| L13 | Automation test mode / dry-run + execution logs | ✅ | V2 | |
| L14 | Guard: automations must not reply when an agent is chatting from the phone (detect recent echo) | ✅ | MVP | |

### M. Broadcasts / Campaigns
| ID | Feature | Feas. | Phase | Keep? |
|---|---|---|---|---|
| M1 | Audience from Zoho custom view / module filter / CSV | ✅ | V2 | |
| M2 | Schedule, throttle (20 msg/s cap for coexistence numbers) | ✅ | V2 | |
| M3 | Exclude opted-out / invalid numbers | ✅ | V2 | |
| M4 | Broadcast analytics + retarget (non-readers, repliers) | ✅ | V2 | |
| M5 | Cost estimate before sending | ✅ | V2 | |

### N. Contacts & Data
| ID | Feature | Feas. | Phase | Keep? |
|---|---|---|---|---|
| N1 | Field mapping: which Zoho phone fields to match, which fields show in side panel | ✅ | MVP | |
| N2 | Tags (local) with optional sync to a Zoho multi-select field | ✅ | V2 | |
| N3 | Consent / opt-in field mapping in Zoho | ✅ | MVP | |
| N4 | Block / spam contacts | ✅ | V2 | |
| N5 | Phone number masking per role | ✅ | V2 | |
| N6 | Duplicate contact detection & merge rules | ✅ | V2 | |
| N7 | Import / export contacts and conversations | ✅ | V2 | |

### O. Inbox Configuration
| ID | Feature | Feas. | Phase | Keep? |
|---|---|---|---|---|
| O1 | Snippets library (personal + shared) | ✅ | V2 | |
| O2 | Closing notes: required on/off, conversation categories | ✅ | V2 | |
| O3 | Conversation statuses & tags setup | ✅ | V2 | |
| O4 | Notification preferences | ✅ | MVP | |
| O5 | SLA targets for reports | ✅ | V2 | |

### P. Users, Roles & Teams
| ID | Feature | Feas. | Phase | Keep? |
|---|---|---|---|---|
| P1 | Import Zoho users as agents; seat assignment | ✅ | MVP | |
| P2 | Roles: Owner, Admin, Manager, Agent, Read-only | ✅ | MVP | |
| P3 | Granular permission matrix (numbers, templates, broadcasts, automations, exports, analytics) | ✅ | V2 | |
| P4 | Teams with members & leads | ✅ | MVP | |
| P5 | Visibility rules (own chats / team / all) | ✅ | MVP | |
| P6 | Agent availability (online/away) & capacity | ✅ | V2 | |

### Q. Integrations & Developer
| ID | Feature | Feas. | Phase | Keep? |
|---|---|---|---|---|
| Q1 | Public REST API with tenant API keys | ✅ | V2 | |
| Q2 | Outgoing webhooks to customer systems | ✅ | V2 | |
| Q3 | Zoho Flow / Zapier / Make / n8n connectors | ✅ | V3 | |
| Q4 | Click-to-WhatsApp ads attribution + Conversions API events | ✅ | V3 | |
| Q5 | Zoho Analytics export | ✅ | V3 | |

### R. Tenant Analytics, Compliance & Billing
| ID | Feature | Feas. | Phase | Keep? |
|---|---|---|---|---|
| R1 | Response/resolution times per agent/team/number | ✅ | V2 | |
| R2 | Volume by source (API vs phone app) | ✅ | V2 | |
| R3 | Meta messaging cost view | ✅ | V2 | |
| R4 | Scheduled email reports | ✅ | V3 | |
| R5 | Audit log viewer | ✅ | V2 | |
| R6 | Retention & deletion settings, data export | ✅ | V2 | |
| R7 | Plan, seats, usage, invoices | ✅ | V2 | |

---

## 9. Feature Catalog — Platform Super-Admin Console

| ID | Feature | Feas. | Phase | Keep? |
|---|---|---|---|---|
| H1 | Tenant list, create, suspend, delete, plan change | ✅ | MVP | |
| H2 | Tenant onboarding tracker (Zoho installed → number linked → sync done → first message) | ✅ | MVP | |
| H3 | Meta app configuration: app ID, Embedded Signup config ID & version, webhook verify token rotation | ✅ | MVP | |
| H4 | Solution Partner / credit line status (if Option B) | ✅ | V2 | |
| H5 | Global number health across tenants (quality, disconnections, inactive phones) | ✅ | MVP | |
| H6 | Webhook monitor: inbound log, failures, dead-letter queue, **replay** | ✅ | MVP | |
| H7 | Immutable audit log of all admin actions | ✅ | MVP | |
| H8 | Support access to tenant only with consent + reason, time-boxed, logged | ✅ | V2 | |
| H9 | Feature flags per tenant/plan | ✅ | V2 | |
| H10 | Plans, pricing, metering, billing provider integration | ✅ | V2 | |
| H11 | System health: queues, latency SLOs, error rates, Meta/Zoho API status | ✅ | MVP | |
| H12 | Zoho extension version & data-center configuration | ✅ | MVP | |
| H13 | Data deletion / export request queue | ✅ | V2 | |
| H14 | Announcements to tenants | ✅ | V3 | |
| H15 | Cost dashboard (infra + Meta + BSP) per tenant | ✅ | V3 | |

---

## 10. Spec Kit Roadmap

| # | Spec | Includes |
|---|---|---|
| 001 | Tenant install, identity & roles | P1, P2, P4, P5, H1, H12 |
| 002 | Number linking (coexistence onboarding) | J1, J3, J5, J6, A1, A2, A3, A5, H3 |
| 003 | Webhook ingestion & message store | A4, A6, B6, H6, H7 |
| 004 | Contact matching with Zoho | C1, C4, C5, N1, N3 |
| 005 | Web Tab inbox | B1–B4, B7, B10, B12, B14, B15, O4 |
| 006 | Template administration | K1–K4, K7 |
| 007 | CRM-triggered template automations | L1, L2, L10, L14 |
| 008 | Record widget & send button | C2, C3 |
| 009 | Number health & platform monitoring | J4, J8, J9, H2, H5, H11 |
| 010 | Collaboration & inbox config | B5, B8, B9, B11, B13, O1–O3, P6 |
| 011 | Routing & rules | L4–L9, L13, J7 |
| 012 | Broadcasts | M1–M5, K5, K10 |
| 013 | Contacts & data admin | N2, N4–N7 |
| 014 | Analytics & compliance | R1–R6 |
| 015 | Billing & platform ops | R7, H4, H8–H10, H13 |
| 016 | Integrations | Q1–Q5 |
| 017 | AI & flow builder | B18, L11, L12 |

### Example: spec 002
```
/speckit.specify A tenant admin links a WhatsApp number to the platform from the
admin console. The wizard checks prerequisites, then launches Meta Embedded Signup v4
in coexistence mode ("connect existing WhatsApp Business App"). After success the
system automatically requests contacts sync and message history sync within the
24-hour window, shows sync progress per phase, handles declined history, and lets
the admin set a nickname, default team and which agents can use the number.
Out of scope: groups, calls, broadcast lists.
```

### Example: spec 007
```
/speckit.specify A tenant admin creates automation rules that send an approved
WhatsApp template when a Zoho CRM event occurs (record created, field changed,
stage changed, date-based reminder). Each rule selects the sending number, the
template, and a variable mapping profile from Zoho fields with fallbacks. Contacts
who opted out are skipped. Every execution is logged with its result. Rules never
fire a reply into a conversation where the agent sent a message from the phone app
in the last N minutes.
```

---

## 11. Open Decisions
1. Option A (independent Tech Provider) or B (with a Solution Partner)? Which BSP?
2. One number per agent, shared number(s), or both?
3. Tech stack & hosting region (data residency).
4. Transcripts stored in Zoho custom module or only in our DB?
5. Pricing model (seat / number / usage / hybrid).
6. Private extension vs public Zoho Marketplace listing.

---

## 12. Sources
- Meta — Become a Tech Provider: https://developers.facebook.com/documentation/business-messaging/whatsapp/solution-providers/get-started-for-tech-providers
- Meta — Onboard WhatsApp Business app users (Coexistence): https://developers.facebook.com/documentation/business-messaging/whatsapp/embedded-signup/onboarding-business-app-users/
- 360dialog — Become a Meta Tech Provider: https://docs.360dialog.com/partner/get-started/tech-provider-program/become-a-meta-tech-provider
- Twilio — Tech Provider integration guide: https://www.twilio.com/docs/whatsapp/isv/tech-provider-program/integration-guide
- Infobip — Tech Provider setup: https://www.infobip.com/docs/whatsapp/tech-provider-program/setup-and-integration
- respond.io — Coexistence: https://respond.io/help/whatsapp/whatsapp-coexistence
- respond.io — Glossary / settings: https://respond.io/help/quick-start/glossary-of-terms
- Wati — Features overview: https://support.wati.io/en/articles/11375155-wati-overview-features-channels-and-integrations
- WOZTELL — Coexistence FAQ: https://support.woztell.com/portal/en/kb/articles/co-existance-faq
- Zoho Campaigns — Coexistence: https://help.zoho.com/portal/en/kb/campaigns-3-0/user-guide/whatsapp-campaigns/articles/v3-whatsapp-coexistence-in-zoho-campaigns-3-0
- Zoho CRM — WhatsApp integration: https://help.zoho.com/portal/en/kb/crm/connections/articles/whatsapp-integration
- Zoho CRM — Widgets: https://www.zoho.com/crm/developer/docs/widgets/create-widget.html
- TimelinesAI — Coexistence guide: https://timelines.ai/whatsapp-coexistence-account-setup-guide
