# Teammate

**An AI collaborator that represents you at work.** Address it in plain language,
starting with the word "Teammate", and it schedules meetings, joins and
contributes to them, takes notes and minutes, presents decks, sends and broadcasts
messages, reads and answers from your files, recalls context from everything you
have connected, and drafts what you should say, across Microsoft Teams, Google
Meet, Slack, Telegram, and Outlook.

The goal: a private second brain plus an autonomous proxy, so you never miss
anything, never forget anything, and always walk in with full context.

---

## What it does

- **Meetings, on your behalf.** Set exactly what Teammate may do (attend, take
  notes, answer when asked, speak, interject, present, record). It follows the
  conversation, contributes within your rules, and writes the minutes. Live
  presence (join, record, speak, transcript) runs through a meeting-bot provider;
  an assisted mode works with no media provider.
- **A personal knowledge base.** Everything you connect or do (documents, cloud
  files, meeting notes, chats, your own activity) is indexed privately. Recall
  answers across all of it with citations, blended with general knowledge.
- **A unified file manager.** Browse every connected cloud (Dropbox, Drive,
  OneDrive) like one file system: folders, upload, download, delete, in one place.
- **Messaging and scheduling.** Single or bulk messages and reminders, real meeting
  creation with join links, deck talk-tracks, and role-aware next steps, on
  whichever platform you use.
- **Your data stays yours.** Your own cloud takes precedence; Teammate storage is a
  fallback with clear per-tier limits and retention.

## The agents

Central Intelligence routes each request to a specialist: **Scheduler,
Note-Taker, Presenter, Communicator, Broadcast, Document-Reader, Finder, Recall,
Contribution, Meeting, Suggestor, Role Analyzer, Fallback.**

## How it is built

- **Everything is admin-controlled at runtime.** Every third-party service (LLM,
  speech, meeting bot, chat platforms, cloud, payments, currency, email, SMS,
  storage, sign-in) is a swappable provider you configure and switch from the admin
  dashboard, with keys encrypted at rest. No code changes, no redeploys.
- **Any AI model.** All model calls go through a universal gateway, plus per-user
  bring-your-own-LLM.
- **Consumption billing.** Metered token usage per command, flexible tier-gated
  top-ups, and live currency conversion for local pricing (Stripe and Paystack).
- **Secure by design.** OAuth2 sign-in (Google, GitHub), JWT sessions with instant
  revocation, RBAC, Fernet-encrypted secrets, signed webhooks, rate limiting, and
  emergency kill-switches.

## Architecture at a glance

```
React dashboard + admin  ->  FastAPI (gate, auth, envelope)  ->  Central Intelligence
   ->  sub-agents  ->  services  ->  one active provider per category
   ->  PostgreSQL + Redis + object storage
```

Stack: Python 3.12 / FastAPI / SQLAlchemy / PostgreSQL / Redis / Celery / LiteLLM
on the backend; React 18 / Vite / Tailwind on the frontend; Docker and a Helm
chart for deployment.

---

## Repositories

| Repo | What it is |
|---|---|
| **teammate-backend** | The FastAPI multi-agent engine and API. |
| **teammate-frontend** | The React dashboard and admin console. |
| **teammate-infrastructure** | Docker, Helm, CI, and the full documentation set. |

## Documentation

The complete, A-to-Z documentation lives in **teammate-infrastructure/docs**:

- **SYSTEM.md**, what Teammate is, architecture, every feature and agent.
- **PROVIDERS.md**, every service, how to get its keys (with links), what to enter.
- **BILLING.md**, subscriptions, tokens, and the exact per-tier math.
- **ALGORITHMS.md**, every algorithm and technique in the code, with references.
- **SECURITY_AND_COMPLIANCE.md**, the safeguards and the laws it respects.
- **OPERATIONS.md**, first-run setup and admin configuration.
