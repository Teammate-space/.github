# Teammate, full system documentation (A to Z)

The complete picture of what Teammate is, how it is built, and everything it does.
This is the master reference. For focused topics see:

- [Providers and API keys](https://github.com/Teammate-space/teammate-infrastructure/blob/main/docs/PROVIDERS.md), how to get every key.
- [Billing and subscriptions](https://github.com/Teammate-space/teammate-infrastructure/blob/main/docs/BILLING.md), the token math per tier.
- [Algorithms and techniques](https://github.com/Teammate-space/teammate-infrastructure/blob/main/docs/ALGORITHMS.md), with references.
- [Security and compliance](https://github.com/Teammate-space/teammate-infrastructure/blob/main/docs/SECURITY_AND_COMPLIANCE.md), measures and laws.
- [Operations and configuration](https://github.com/Teammate-space/teammate-infrastructure/blob/main/docs/OPERATIONS.md), first run and admin.

---

## 1. What Teammate is

Teammate is a multi-agent AI collaborator that represents a person at work. You
address it in plain language starting with the word "Teammate" and it schedules
meetings, joins and contributes to them, takes notes and minutes, presents decks,
sends and broadcasts messages, reads and answers from your files, recalls context
from everything you have connected, and drafts what you should say, across
Microsoft Teams, Google Meet, Slack, Telegram, and Outlook.

The design goal: a private second brain plus an autonomous proxy, so the user
never misses anything, never forgets anything, and can contribute better than
everyone in the room by always having full context.

Three principles run through the whole system:

1. **Everything is admin-controllable at runtime.** Every third-party service is a
   swappable provider configured from the admin dashboard, with keys encrypted at
   rest. No code changes or redeploys to switch an LLM, a payment processor, or a
   meeting bot.
2. **Every platform is equal.** Teams, Meet, Slack, Telegram, and Outlook all
   expose the same capability surface (messaging, meetings, files, notes,
   presenting). The only difference is which one the user connects.
3. **The user owns their data.** Their own connected cloud takes precedence for
   their files; Teammate storage is a fallback with clear, per-tier limits and
   retention.

---

## 2. Technology stack

**Backend** (`teammate-backend`): Python 3.12, FastAPI, SQLAlchemy 2 (async) with
Alembic migrations, PostgreSQL (asyncpg), Redis, Celery + beat, LiteLLM (universal
LLM gateway), Authlib/PyJWT, structlog, httpx, boto3, cryptography (Fernet).

**Frontend** (`teammate-frontend`): React 18, Vite 6, Tailwind CSS, React Router 6,
TanStack Query 5, Zustand, axios, framer-motion, lucide-react, react-hot-toast.

**Infrastructure** (`teammate-infrastructure`): Docker Compose for local/all-in-one,
a Helm chart for Kubernetes and on-prem, GitHub Actions CI in every repo, and the
cross-cutting docs and roadmap.

The design docs originally described a different stack (MongoDB/TimescaleDB,
LangGraph, Hugging Face, Auth0, Stripe only). The shipped system deliberately
diverges: PostgreSQL as the single database, a plain async dict router instead of
LangGraph, LiteLLM instead of Hugging Face, OAuth2 (Google and GitHub) instead of
Auth0, and Stripe plus Paystack for payments. The code is the source of truth.

---

## 3. Architecture layers

```
 Presentation   React dashboard + admin console  ·  platform webhooks
      |
 API / Gateway  FastAPI, gate middleware (rate limit, maintenance, lockdown,
      |         security headers), auth (OAuth2 JWT or X-API-Key), response envelope
      |
 Orchestration  Central Intelligence: listener -> intent -> router -> sub-agents,
      |         metering, context
      |
 Services        LLM, embeddings, knowledge, billing, FX, discounts, meetings,
      |          storage, cloud, integrations, notifications, insights, roles, audit
      |
 Providers       one active adapter per category, hot-swappable, encrypted creds
      |
 Data            PostgreSQL (durable) + Redis (context/cache/broker) + object store
```

### 3.1 The command pipeline (core flow)

A user utterance flows through `src/agents/`:

1. **listener.py** gates on the literal word "Teammate" and strips it. A bare
   dismissal ("Teammate, ignore that") is acknowledged and does nothing.
2. **central_intelligence.py** is the orchestrator: loads Redis context, resolves
   the user's BYO-LLM config, classifies intent, routes, meters tokens for every
   LLM call, persists context and history, and learns from the exchange.
3. **intent_detector.py** classifies the utterance into an intent with a confidence
   score via an LLM call; low confidence or unknown routes to the fallback.
4. **task_router.py** maps the intent to a sub-agent handler via a plain dict.
5. **Sub-agents** each produce a response and can deliver it to the user's platform.

Token metadata (`cost_tokens`, `model_used`, `is_byollm`, `fell_back`) is threaded
through every stage so the orchestrator can bill correctly. BYO-LLM usage is billed
only a platform fee, never the LLM markup.

### 3.2 The provider registry

`src/providers/registry.py` holds a catalog of adapters that self-register with
`@register_adapter`. Categories: **llm, embeddings (via setting), email, sms,
speech, storage, cloud, payments, fx, oauth, integration, meeting_bot**. Each
category resolves its one active `ProviderConfig` from the database, decrypts
credentials, and instantiates the adapter. To add a provider you implement the
base class and register it once; to switch you configure, Test, and Set active in
the admin dashboard. See [PROVIDERS.md](https://github.com/Teammate-space/teammate-infrastructure/blob/main/docs/PROVIDERS.md).

### 3.3 Data layer

PostgreSQL via SQLAlchemy async with Alembic migrations. Key tables: `users`,
`oauth_accounts`, `organizations`, `provider_configs`, `runtime_settings`,
`audit_logs`, `subscription_plans`, `token_packs`, `token_transactions`,
`subscriptions`, `payments`, `discounts`, `discount_redemptions`,
`integration_connections`, `documents`, `interactions`, `notifications`,
`api_keys`, `workflows`, `knowledge_chunks`, `meeting_sessions`. Redis backs
conversational context (1 hour TTL), caching, rate limits, and the Celery broker.

---

## 4. The agents

| Agent | Intent | What it does |
|---|---|---|
| Central Intelligence | (orchestrator) | Runs the whole pipeline, meters, remembers. |
| Listener | (gate) | Trigger word detection and dismissal handling. |
| Intent Detector | (classify) | LLM classification with confidence + fallback. |
| Scheduler | schedule_meeting, cancel_meeting, get_meeting_info | Creates real meetings on connected platforms, sources a join link, sends reminders. |
| Note-Taker | take_notes | Structured notes and summaries, saved to the user's chosen storage and indexed. |
| Presenter | present_slides | Reads a deck (PPTX/PDF) and builds a slide-by-slide talk track. |
| Communicator | send_message | Drafts and sends a single message on the chosen platform. |
| Broadcast | broadcast_message | Bulk messages and reminders to many recipients at once. |
| Document-Reader | query_document | Answers from a specific uploaded document's real text. |
| Finder | find_files | One search across every connected cloud and imported documents. |
| Recall | ask_knowledge | Answers across the whole personal knowledge base with citations, blended with general knowledge. |
| Contribution | draft_contribution | Drafts what the user could say, grounded in their knowledge. |
| Meeting | (meeting sessions) | Follows a live transcript and contributes within the user's autonomy policy; writes minutes. |
| Suggestor | get_suggestions | Turns discussion into next steps, role-aware. |
| Role Analyzer | (opt-in) | Infers the user's role to personalize suggestions. |
| Fallback | (unknown) | Clarifies or handles anything unrecognized gracefully. |

Recall, Finder, Contribution, Broadcast, and the Meeting autonomy agent are
capabilities invented during the build beyond the original vision.

---

## 5. Features, end to end

### 5.1 Command and conversation
Natural-language commands from the dashboard or any connected platform webhook. The
word "Teammate" triggers; a dismissal stands down.

### 5.2 Meetings and autonomy
A meeting session carries an **autonomy policy** (attend, take notes, answer when
asked, speak, interject, present, record, plus a persona and standing
instructions). The meeting agent reads the transcript and contributes only within
those permissions, grounded in the user's knowledge. At the end it writes minutes,
delivers them, saves them to the user's chosen storage, and files them into the
knowledge base. Live audio/video presence (join, record, speak, transcript) is
performed by a configured meeting-bot provider (Recall.ai); a built-in assisted
mode works with no media provider by taking transcript turns via the API.

### 5.3 Personal knowledge base (second brain)
Everything the user connects or does is indexed into a private, per-user knowledge
base: uploaded documents, cloud files, meeting notes, chats, and their own
interactions. Recall answers across all of it with citations, blended with general
model knowledge. Embeddings go through the LLM gateway when an embedding model is
set, otherwise a built-in lexical fallback so recall works with zero setup. A
background job keeps indexing new cloud files. See [ALGORITHMS.md](https://github.com/Teammate-space/teammate-infrastructure/blob/main/docs/ALGORITHMS.md).

### 5.4 Files and documents
A unified **file manager** browses every connected cloud (Dropbox, Drive, OneDrive)
like a native file system: folders, upload, download, new folder, delete, all in
one place. Documents can also be uploaded directly. Parsers extract text from PDF,
DOCX, PPTX, XLSX, CSV, and TXT so agents can read and present from real content.

### 5.5 Cloud storage precedence and retention
Per-user OAuth means each user connects their own cloud; Teammate reads and writes
only within their account. Files Teammate produces follow the user's storage
preference (their cloud, Teammate storage, or automatic). Teammate storage is
capped and retained per tier; files past retention are auto-deleted unless the user
downloads them or moves them to their own cloud. See [BILLING.md](https://github.com/Teammate-space/teammate-infrastructure/blob/main/docs/BILLING.md).

### 5.6 Messaging, scheduling, presenting, suggestions
Single and bulk messaging, meeting creation with reminders, deck talk-tracks, and
role-aware next steps, all platform-agnostic.

### 5.7 History, notifications, insights, workflows
Durable interaction history (searchable, filterable). In-app and email
notifications. AI insights over activity. User-defined workflow automations
(command/message/summary actions, manual or scheduled via Celery beat).

### 5.8 Billing, tokens, and BYO-LLM
Consumption-based token metering, per-tier plans, flexible top-ups, promo codes,
and bring-your-own-LLM (billed at the platform-fee rate, with automatic fallback to
the platform model if the user's key fails). See [BILLING.md](https://github.com/Teammate-space/teammate-infrastructure/blob/main/docs/BILLING.md).

### 5.9 Enterprise API
Enterprise-tier users get SHA-256-hashed API keys (`X-API-Key`) for programmatic
access to the whole surface.

### 5.10 Admin control plane
Providers (configure/test/switch), runtime settings, users (roles, tiers,
ban/unban, force-logout), discounts, and emergency controls.

### 5.11 UX details
Teal/coral/lemon design system, Inter font, light and dark, and an animated
"working" indicator so the screen is never blank during a long action.

---

## 6. Request lifecycle and cross-cutting concerns

- **Auth**: OAuth2 sign-in (Google or GitHub) issues JWT access/refresh tokens;
  Enterprise API uses `X-API-Key`. `get_current_db_user` is the standard route
  dependency.
- **Gate middleware**: per-identity rate limiting, maintenance and lockdown modes,
  IP blocklist, and security headers on every request.
- **Response envelope**: success `{success, message, data}`; error
  `{success, error:{code, message}}`.
- **Errors**: typed errors and `safe_execute`/`retry_async` helpers rather than
  ad-hoc try/except in agents.
- **Audit**: sensitive actions (billing, admin, emergency) are recorded in
  `audit_logs`.

---

## 7. Background jobs (Celery beat)

| Job | Cadence | Purpose |
|---|---|---|
| run_due_workflows | every minute | Scheduled workflows and reminders. |
| sync_knowledge | every 30 min | Index new cloud files into the knowledge base. |
| purge_expired_storage | daily | Delete Teammate files past their tier retention. |
| reset_monthly_usage | monthly | Reset monthly command counters. |

---

## 8. Repositories

- **[teammate-backend](https://github.com/Teammate-space/teammate-backend)**, the FastAPI multi-agent engine and API.
- **[teammate-frontend](https://github.com/Teammate-space/teammate-frontend)**, the React dashboard and admin console.
- **[teammate-infrastructure](https://github.com/Teammate-space/teammate-infrastructure)**, Docker, Helm, CI, and the full documentation set.

Each repo has its own README scoped to its concerns; this document ties them
together.

---

## 9. Where to go next

- Set it up: [OPERATIONS.md](https://github.com/Teammate-space/teammate-infrastructure/blob/main/docs/OPERATIONS.md).
- Get every key: [PROVIDERS.md](https://github.com/Teammate-space/teammate-infrastructure/blob/main/docs/PROVIDERS.md).
- Understand the money: [BILLING.md](https://github.com/Teammate-space/teammate-infrastructure/blob/main/docs/BILLING.md).
- Understand the internals: [ALGORITHMS.md](https://github.com/Teammate-space/teammate-infrastructure/blob/main/docs/ALGORITHMS.md).
- Understand the safeguards and laws: [SECURITY_AND_COMPLIANCE.md](https://github.com/Teammate-space/teammate-infrastructure/blob/main/docs/SECURITY_AND_COMPLIANCE.md).
