# Recompra — Design Document

**Date:** 2026-02-17
**Status:** Approved
**Author:** Felipe Kafuri

## Summary

Recompra is a dunning management plugin for AbacatePay's plugin store. It monitors failed payments via webhooks, runs configurable recovery campaigns (automated retries + multi-channel notifications), and provides merchants with a dashboard showing recovered revenue.

**Goal:** Visibility in AbacatePay's ecosystem + recurring revenue from a freemium SaaS.

## Architecture

```
┌─────────────────────────────────────────────────┐
│              Recompra                     │
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐ │
│  │ Webhook  │  │  Retry   │  │  Notification │ │
│  │ Receiver │→ │  Engine  │→ │   Dispatcher  │ │
│  └──────────┘  └──────────┘  └───────────────┘ │
│       ↓             ↓              ↓            │
│  ┌──────────────────────────────────────────┐   │
│  │           SQLite (via Ent ORM)           │   │
│  └──────────────────────────────────────────┘   │
│       ↑                                         │
│  ┌──────────┐  ┌──────────┐                     │
│  │ Dashboard│  │ Recovery │                     │
│  │ (React)  │  │  Page    │                     │
│  └──────────┘  └──────────┘                     │
└─────────────────────────────────────────────────┘
         ↕                    ↕
   AbacatePay API       Email / WhatsApp
   (webhooks + billing)  (notification APIs)
```

### Components

1. **Webhook Receiver** — Listens for AbacatePay payment events (failed, expired, succeeded). Validates signatures, creates FailedPayment records.
2. **Retry Engine** — Background worker that schedules and creates new billing attempts via AbacatePay's `POST /billing/create` API.
3. **Notification Dispatcher** — Sends recovery emails/WhatsApp messages to customers at configurable intervals with unique recovery links.
4. **Recovery Page** — Public-facing page where customers can complete payment via a fresh PIX QR code or card link.
5. **Dashboard** — Merchant-facing UI showing recovery metrics, campaign management, and settings.

### Tech Stack

- **Backend:** Go + Echo + Ent ORM + SQLite
- **Frontend:** Inertia.js + React (same patterns as Bandeira/Pagode)
- **Deployment:** Docker (managed SaaS + self-hosted option)

## Data Model

### Merchant
| Field | Type | Description |
|-------|------|-------------|
| id | uuid | Primary key |
| abacatepay_account_id | string | AbacatePay account identifier |
| api_key | string | Encrypted AbacatePay API key |
| webhook_secret | string | For validating incoming webhooks |
| name | string | Merchant display name |
| email | string | Contact email |
| settings | json | Preferences: timezone, branding, email from-name |
| tier | enum | free, pro, business |
| created_at | time | |
| updated_at | time | |

### RecoveryCampaign
| Field | Type | Description |
|-------|------|-------------|
| id | uuid | Primary key |
| merchant_id | uuid | FK to Merchant |
| name | string | e.g., "Default PIX Recovery" |
| is_active | bool | Whether campaign is active |
| is_default | bool | Used when no specific match |
| min_amount | int | Optional: minimum payment amount (cents) |
| max_amount | int | Optional: maximum payment amount (cents) |
| max_retries | int | Total retry attempts allowed |
| created_at | time | |
| updated_at | time | |

### CampaignStep
| Field | Type | Description |
|-------|------|-------------|
| id | uuid | Primary key |
| campaign_id | uuid | FK to RecoveryCampaign |
| step_number | int | Order within campaign (1, 2, 3...) |
| delay_hours | int | Hours after previous step or initial failure |
| action | enum | retry_payment, send_notification, both |
| notification_channel | enum | email, whatsapp (nullable) |
| notification_template | enum | gentle_reminder, urgent_reminder, last_chance, custom |
| custom_message | text | Nullable, for custom templates |

Example default campaign:
```
Step 1: +1h   → retry_payment
Step 2: +24h  → both (retry + email gentle_reminder)
Step 3: +72h  → both (retry + email urgent_reminder)
Step 4: +7d   → send_notification (last_chance email)
Step 5: +14d  → abandon
```

### FailedPayment
| Field | Type | Description |
|-------|------|-------------|
| id | uuid | Primary key |
| merchant_id | uuid | FK to Merchant |
| campaign_id | uuid | FK to RecoveryCampaign |
| abacatepay_billing_id | string | Original billing ID from AbacatePay |
| customer_email | string | |
| customer_name | string | |
| customer_phone | string | Optional, for WhatsApp |
| amount | int | Amount in cents |
| currency | string | BRL |
| failure_reason | string | insufficient_funds, expired, declined, etc. |
| status | enum | pending, retrying, recovered, abandoned |
| original_failed_at | time | When the payment first failed |
| recovered_at | time | Nullable, when payment was recovered |
| retry_count | int | Current retry count |
| created_at | time | |
| updated_at | time | |

### RetryAttempt
| Field | Type | Description |
|-------|------|-------------|
| id | uuid | Primary key |
| failed_payment_id | uuid | FK to FailedPayment |
| abacatepay_billing_id | string | New billing created for this retry |
| attempt_number | int | |
| scheduled_at | time | When this retry was scheduled |
| executed_at | time | When it actually ran |
| result | enum | success, failed, pending, skipped |
| created_at | time | |

### Notification
| Field | Type | Description |
|-------|------|-------------|
| id | uuid | Primary key |
| failed_payment_id | uuid | FK to FailedPayment |
| channel | enum | email, whatsapp |
| template_key | string | e.g., gentle_reminder |
| sent_at | time | |
| opened_at | time | Nullable, email tracking |
| clicked_at | time | Nullable, link click tracking |
| recovery_link | string | Unique URL to recovery page |
| status | enum | queued, sent, delivered, failed |

### RecoveryEvent
| Field | Type | Description |
|-------|------|-------------|
| id | uuid | Primary key |
| failed_payment_id | uuid | FK to FailedPayment |
| event_type | string | webhook_received, retry_scheduled, notification_sent, payment_recovered, abandoned |
| metadata | json | Additional context |
| created_at | time | |

## User Flows

### 1. Merchant Onboarding
1. Merchant installs plugin from AbacatePay store
2. Redirected to Recompra setup page
3. Enters AbacatePay API key
4. System validates key (fetches merchant info from AbacatePay API)
5. System registers webhook URL on their account
6. Default recovery campaign auto-created
7. Dashboard loads — merchant is live

### 2. Failed Payment Recovery
1. AbacatePay fires webhook: `billing.failed` or `billing.expired`
2. Webhook receiver validates signature, creates FailedPayment record
3. System matches payment to a campaign (by amount range, or default)
4. Retry engine schedules Step 1 based on campaign delay
5. At each step:
   - `retry_payment`: Create new billing via AbacatePay `POST /billing/create`
   - `send_notification`: Dispatch email/WhatsApp with unique recovery link
   - `both`: Do both
6. On `billing.paid` webhook for recovery billing → mark as `recovered`
7. All steps exhausted → mark as `abandoned`

### 3. Customer Recovery Page
1. Customer clicks recovery link from email/WhatsApp
2. Branded page shows: amount owed, merchant name, payment options
3. Fresh PIX QR code generated via AbacatePay API
4. Customer pays → webhook confirms → marked recovered
5. Success confirmation displayed

### 4. Dashboard
- **Overview:** Total failed payments, recovered amount (R$), recovery rate %, MRR saved
- **Active campaigns:** In-progress recovery attempts with status
- **Campaign editor:** Create/edit campaigns with step builder UI
- **History:** All recovered and abandoned payments with timeline
- **Settings:** API key, notification preferences, branding

## Monetization

| Tier | Price | Limits |
|------|-------|--------|
| Free | R$0 | 50 recovery attempts/month, email only, 1 campaign |
| Pro | R$49/month | Unlimited recoveries, email + WhatsApp, unlimited campaigns, custom branding |
| Business | R$149/month | Pro + priority support, API access, white-label recovery pages |

## Deployment

- Docker image + docker-compose.yml
- SQLite (single-file DB, no external dependencies)
- Single Go binary with embedded React assets
- Health check endpoint
- Managed SaaS (primary) + self-hosted option

## External Dependencies

- **AbacatePay API** — Billing creation, webhook registration, payment status
- **Email provider** — Resend, SendGrid, or Amazon SES
- **WhatsApp API** — Twilio or Meta Business API (Pro tier)
