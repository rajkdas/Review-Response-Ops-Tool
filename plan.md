# Review Response Ops Tool — Technical Specification

**Version:** 1.0
**Audience:** Developer building this system
**Owner/operator:** Single operator managing multiple small-business clients

---

## 1. Purpose & Scope

This is an **internal operations tool**, not a customer-facing product. The end client (a small business owner) never logs into this system. The operator uses it to:

1. Monitor new Google reviews across multiple client locations.
2. Get an AI-drafted reply for each new review.
3. Review, edit, and approve the draft.
4. Post the approved reply (manually today, automatically once/if API access allows — see Section 6.3).
5. Send each client a weekly summary email.

**Adding a new client should require zero code changes** — only a new row of configuration (business name, Place ID, tone notes). This requirement drives the architecture in Section 3.

### 1.1 Explicit non-goals for v1

Do not build these now. The interfaces below should allow adding them later without restructuring, but none are in scope for the first build:

- No client-facing login, dashboard, or self-serve portal.
- No review platforms other than Google (no Yelp, TripAdvisor, Trustpilot) — the adapter pattern must allow adding one later as a new class, not a rewrite.
- No automatic review solicitation (QR codes, SMS/email review requests).
- No automatic posting via the official Google Business Profile API on day one — see Section 6.3 for why, and how the interface should be shaped so this slots in later without touching core logic.
- No multi-user/team accounts. Single operator login (or no login at all if self-hosted locally) is sufficient.

---

## 2. Architecture Overview

Modular, config-driven, single-tenant-operationally (one operator) but multi-client-in-data (many businesses/locations in the database).

```
┌─────────────────────────────────────────────┐
│              Admin Web UI (internal)         │
│   Review queue · Approve/edit · Client config │
└───────────────────┬───────────────────────────┘
                     │ REST/JSON
┌───────────────────▼───────────────────────────┐
│                Backend API                     │
│  ReviewService · ReplyService · SummaryService │
└───┬───────────────┬────────────────┬───────────┘
    │                │                │
┌───▼────┐     ┌─────▼──────┐   ┌────▼─────────┐
│ Review │     │ AI Reply   │   │ Notification │
│ Source │     │ Provider   │   │  Provider    │
│Adapter │     │ (any API)  │   │(email/Telegram)│
└────────┘     └────────────┘   └──────────────┘
    │                │                │
Google Places   OpenAI/Claude/    SMTP/Sendgrid/
API (v1)        self-hosted HF     Telegram Bot
                model endpoint
                (config-selected)
```

**Core principle:** the `ReviewService`, `ReplyService`, and `SummaryService` never call a specific vendor's SDK directly. They call an interface. Which vendor answers that interface is a config value on each `Location`, not a code branch. This is what lets you swap Claude for GPT-4 for a self-hosted Llama endpoint per client, or all at once, by editing a config row.

---

## 3. Domain Model

Relational database (PostgreSQL recommended; SQLite is fine at this scale if the developer prefers zero infra to start).

### `Business`
| Field | Type | Notes |
|---|---|---|
| id | UUID | |
| name | string | |
| primary_contact_name | string | |
| primary_contact_email | string | |
| created_at / updated_at | timestamp | |

### `Location`
One row per physical location (most clients will have exactly one).

| Field | Type | Notes |
|---|---|---|
| id | UUID | |
| business_id | FK → Business | |
| display_name | string | e.g. "Main Street Dental" |
| google_place_id | string | from Google Places, used for read access |
| time_zone | string | for scheduling weekly summaries |
| tone_notes | text | free-text guidance fed to the AI provider, e.g. "warm, uses first names, never defensive" |
| status | enum: `active`, `paused`, `offboarded` | |
| created_at / updated_at | timestamp | |

### `IntegrationConfig`
This is the config-driven swap point. One row per (location, integration type).

| Field | Type | Notes |
|---|---|---|
| id | UUID | |
| location_id | FK → Location | |
| type | enum: `review_source`, `ai_provider`, `notification` | |
| provider_key | string | e.g. `google_places`, `openai_gpt4`, `claude_sonnet`, `self_hosted_hf`, `smtp`, `telegram` |
| config | JSON | provider-specific: API keys, model name, endpoint URL, chat_id, etc. |
| is_active | boolean | allows disabling without deleting |
| created_at / updated_at | timestamp | |

Why this table exists: switching a client's AI provider, or adding a second review source later, is an `UPDATE`/`INSERT` on this table — never a deploy.

### `Review`
| Field | Type | Notes |
|---|---|---|
| id | UUID | |
| location_id | FK → Location | |
| source | enum: `google_places` (extendable) | |
| source_review_id | string | platform's own ID, used for dedup |
| author_name | string | |
| rating | int 1–5 | |
| text | text | |
| language | string | |
| created_at_source | timestamp | when the review was actually posted |
| fetched_at | timestamp | when this system saw it |
| status | enum: `new`, `reply_drafted`, `reply_approved`, `reply_posted`, `flagged` | |
| flagged_reason | string, nullable | e.g. "possible legal/safety claim — do not auto-reply" |

### `Reply`
| Field | Type | Notes |
|---|---|---|
| id | UUID | |
| review_id | FK → Review | |
| reply_text | text | |
| generated_by | enum: `ai`, `human` | |
| ai_provider_key | string, nullable | which `IntegrationConfig.provider_key` generated it |
| status | enum: `draft`, `edited`, `approved`, `posted` | |
| posted_at | timestamp, nullable | |
| posted_by | enum: `manual`, `api`, nullable | how it actually went live |

### `WeeklySummary`
| Field | Type | Notes |
|---|---|---|
| id | UUID | |
| location_id | FK → Location | |
| period_start / period_end | date | |
| num_reviews | int | |
| avg_rating | decimal | |
| num_replies_posted | int | |
| num_flagged | int | |
| sent_at | timestamp, nullable | |

---

## 4. Core Interfaces

These are the contracts every provider must satisfy. The developer can implement them in whatever language is chosen — shown here in TypeScript-flavored pseudocode for clarity, not as a mandate.

### 4.1 `ReviewSourceAdapter`

```ts
interface ReviewSourceAdapter {
  fetchRecentReviews(config: IntegrationConfig, sinceDate?: Date): Promise<ReviewDTO[]>;
  postReply(config: IntegrationConfig, review: Review, reply: Reply): Promise<PostReplyResult>;
}
```

- **v1 implementation: `GooglePlacesAdapter`.** Uses the Google Places API (Place Details endpoint) to read a location's public reviews by Place ID. This requires **no permission from the client and no approval process** — it's a standard pay-as-you-go Google Maps Platform API key. Note its real limitation: it typically surfaces only a handful of the most recent reviews per location, which is sufficient for "is there a new one since I last checked" but not for bulk historical import.
- `postReply()` in `GooglePlacesAdapter` should be implemented as a **no-op that raises `ManualActionRequired`** — the operator posts manually via the Google Business Profile dashboard (see Section 6). This keeps the interface honest: the method exists, it's just not automated yet.
- **Future implementation: `GoogleBusinessProfileAdapter`.** Same interface, but backed by the official GBP API with OAuth, enabling real `postReply()`. Do not build this until API access is actually granted (see Section 6.3) — but because it satisfies the same interface, adding it later is a new class and a config change, not a rewrite of `ReplyService`.
- **Future implementation: `YelpAdapter`, `TrustpilotAdapter`, etc.** Same pattern, added only if a client needs it.

### 4.2 `AIReplyProvider`

```ts
interface AIReplyProvider {
  generateReply(input: {
    reviewText: string;
    rating: number;
    businessContext: { name: string; toneNotes: string };
    language?: string;
  }): Promise<string>;
}
```

This must be a **generic HTTP-based interface with no vendor assumptions baked into the core system.** Concretely:

- Implement one thin adapter class per provider family: `OpenAIProvider`, `AnthropicProvider`, `SelfHostedHFProvider` (calls any OpenAI-compatible or custom HTTP endpoint — covers self-hosted Llama/Mistral/etc. via something like text-generation-inference or Ollama's API).
- All three take the same input shape and return plain text. `ReplyService` calls whichever one is configured on `IntegrationConfig` for that location — it does not know or care which model answered.
- Model name, API key, and endpoint URL all live in `IntegrationConfig.config` (JSON), never hardcoded.
- Practical recommendation: build the `OpenAIProvider` and `AnthropicProvider` adapters first since they're both a single HTTP call away and cover the vast majority of quality/cost tradeoffs; add `SelfHostedHFProvider` only if/when a specific reason to self-host arises (data sensitivity, cost at scale). The interface supports it from day one either way.

### 4.3 `NotificationProvider`

```ts
interface NotificationProvider {
  sendWeeklySummary(summary: WeeklySummary, recipientEmail: string): Promise<void>;
  sendOperatorAlert(message: string): Promise<void>;
}
```

- `sendWeeklySummary` → goes to the client (SMTP/Sendgrid).
- `sendOperatorAlert` → goes to you, the operator (Telegram bot or email), for things like "new review drafted, ready for your approval" or "review flagged for manual attention." This is what replaces a customer-facing dashboard notification — you get pinged directly instead of needing to check a UI.

---

## 5. Backend Services

### 5.1 `ReviewService`
- Runs on a schedule (cron/job queue, every 30–60 minutes is plenty for this use case — reviews aren't that time-sensitive).
- For each active `Location`, loads its `review_source` `IntegrationConfig`, calls `fetchRecentReviews()`.
- Dedupes against `source_review_id`; inserts new `Review` rows with `status = 'new'`.

### 5.2 `ReplyService`
- For each `Review` with `status = 'new'`, loads the location's `ai_provider` config, calls `generateReply()`.
- Saves a `Reply` (`status = 'draft'`), sets `Review.status = 'reply_drafted'`.
- Fires `sendOperatorAlert()` so the operator knows a draft is ready.
- Exposes endpoints for the admin UI: edit reply text, approve, mark posted.
- On "approve," if `postReply()` on the active adapter throws `ManualActionRequired`, the UI should surface a clear instruction: "Post this manually in Google Business Profile," with the reply text ready to copy.

### 5.3 `SummaryService`
- Weekly, per location: aggregates the period's reviews/replies, computes metrics, creates a `WeeklySummary`, calls `sendWeeklySummary()`.

### 5.4 Simple review-flagging rule (v1, not AI-based)
Before generating a reply, run a lightweight check: if the review mentions terms suggesting legal/safety risk (e.g. "lawsuit," "injury," "food poisoning," "discrimination" — a short, editable keyword list is enough for v1), set `status = 'flagged'` instead of drafting a reply automatically, and alert the operator to handle it personally. Don't over-engineer this into an ML classifier for v1 — a keyword list the operator can edit is sufficient and transparent.

---

## 6. Access & Posting Model

### 6.1 Reading reviews
Google Places API, keyed by Place ID, per location. No client permission needed — this is public data.

### 6.2 Posting replies (v1: manual)
The operator is added as **Manager** on each client's Google Business Profile (owner sends the invite by email — no password sharing, standard Google feature). The operator posts the approved reply directly in Google's own interface. The tool's job is to make this fast (draft ready, one-click "copy text"), not to eliminate the click entirely.

### 6.3 Posting replies (v2: automatic, conditional)
The official Google Business Profile API supports posting replies programmatically, but requires an application and Google's approval — and as of now, Google has publicly acknowledged unusually high backlogs with no committed processing time. **Do not schedule any work around getting this access by a specific date.** Build `GoogleBusinessProfileAdapter` only once approval actually comes through; the interface in Section 4.1 is designed so this is additive, not a rewrite.

### 6.4 Secrets
All API keys (Google Places, AI providers, SMTP/Telegram) in environment variables or a secrets manager — never committed to source control.

---

## 7. Admin UI (internal — operator only)

Keep this minimal. It's a tool you use daily, not a product anyone else evaluates.

- **Review queue**: list of reviews across all clients, filterable by status and by location. This is the main working screen.
- **Review detail**: full text, rating, AI draft, an editable text box, "Approve" button, "Flag" button.
- **Client list/config**: add a `Business` + `Location`, set the Place ID, set tone notes, set which AI provider to use for that location. This is the whole "onboarding a client" flow — a form, not a deploy.
- **Weekly summaries log**: what was sent to each client and when, for your own record.

A single-user login (or none, if this only ever runs on your own machine/private server) is sufficient. No client accounts, no roles, no OAuth complexity — that's the biggest scope cut versus the previous version of this doc.

---

## 8. Client Onboarding Flow (why this satisfies your "don't rebuild" requirement)

Adding client #6 should look like:

1. Find their Google Place ID (a public lookup, takes a minute).
2. Add a `Business` + `Location` row via the admin UI form.
3. Add an `IntegrationConfig` row picking their AI provider (defaults to whatever you use for everyone, override only if a client needs something specific — e.g. a different tone model, or a self-hosted endpoint for a client sensitive about data).
4. Get added as Manager on their Google Business Profile.
5. Done — the scheduler picks up the new location on its next run automatically.

No code touched, no deploy needed, for a standard client. Code changes are only needed when adding a genuinely new *type* of thing — a new review platform, a new AI vendor family — which is exactly the boundary the adapter pattern is meant to protect.

---

## 9. Suggested Tech Stack (recommendations, not requirements — swap freely)

- **Backend:** Python (FastAPI) or Node.js (Express/Fastify) — either is fine; pick whichever the developer is faster in.
- **Database:** PostgreSQL, or SQLite if you want zero infrastructure to start (genuinely fine at 1–20 clients).
- **Scheduler:** a simple cron job or lightweight task queue (e.g. `node-cron`, Python `APScheduler`) — no need for Celery/Redis-scale infrastructure at this size.
- **Admin UI:** a minimal React or server-rendered (e.g. FastAPI + Jinja, or Next.js) app — this does not need to be a polished product; function over form.
- **Hosting:** a single small VM or a platform like Railway/Render is more than sufficient at this scale; no need for container orchestration.

---

## 10. Phased Build Plan

**Phase 0 (skip if possible):** You manually check GBP dashboards and draft replies with any AI chat tool for your first 1–3 clients, before any of this is built. Validates the offer before spending dev budget. (Not part of this spec — mentioned for sequencing.)

**Phase 1 — MVP (this is what to actually scope with the developer first):**
- Domain model + database (Section 3)
- `GooglePlacesAdapter` (read-only)
- One `AIReplyProvider` implementation (pick one to start; the interface makes adding the second one trivial later)
- `ReviewService` + `ReplyService`, manual posting flow
- Minimal admin UI: review queue, review detail/approve, client config form
- Operator alerts via Telegram or email
- Rough estimate for a competent freelance developer: **1–2 weeks**, not 4–6 — this is the scope check to hold them to.

**Phase 2 — once Phase 1 is proven with real clients:**
- `WeeklySummary` + client-facing email
- Second `AIReplyProvider` if you want to A/B tone/quality/cost across providers
- Keyword-based flagging rule (Section 5.4)

**Phase 3 — only if/when justified:**
- `GoogleBusinessProfileAdapter` for auto-posting (contingent on API approval — see 6.3)
- Additional `ReviewSourceAdapter`s (Yelp, Trustpilot) if a client specifically needs them
