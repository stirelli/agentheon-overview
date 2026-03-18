# Agentheon — AI Outbound System

Agentheon is an end-to-end outbound system built with LLM-based agents, designed to run real campaigns with personalization, deliverability control, and human oversight.

---

## The Problem

Most outbound tools treat email as a single-step problem: fill a template, send. The result is generic outreach that damages domain reputation and gets ignored.

The real challenge is orchestrating the full pipeline — research, generation, domain health, human oversight, and delivery — as a single system. Agentheon treats outbound as an infrastructure problem, not a copywriting problem.

---

## How It Works

```
Apollo → Lead Ingestion → Research → Email Generation → Human Review → Send → Analytics
                                          ↑                                      |
                                          └──────────── Feedback Loop ───────────┘
```

1. **Import** — Prospects are ingested from Apollo via the BYOA (Bring Your Own Apollo) model
2. **Research** — Specialized agents gather context on each prospect: company news, funding signals, relevant initiatives
3. **Generate** — A copywriting agent produces personalized emails grounded in the research output
4. **Review** — Users approve, edit, or reject every email before it's sent
5. **Send** — The CAO engine manages sending capacity, domain warm-up, and delivery timing
6. **Track** — Webhook-driven analytics surface opens, replies, and bounces in real time

---

## Key System Components

### Agent Orchestration
The workflow is decomposed into specialized agents — research, enrichment, copywriting — coordinated through a LangGraph-based pipeline. Each agent is modular, independently testable, and traced via LangSmith.

### Capacity Allocation Optimizer (CAO)
Deliverability is a first-class concern. The CAO engine manages domain warm-up, enforces sending limits per account, and dynamically adjusts volume to protect domain reputation. This allows campaigns to start sending real emails early without the typical 30-day warm-up penalty.

### Human-in-the-Loop Approval
Nothing gets sent without explicit user approval. The review interface lets users inspect generated emails, edit content, and approve or reject on a per-message basis. This keeps automation accountable.

### Real-Time Analytics
Campaign metrics (opens, clicks, replies, bounces) are captured via webhooks and streamed to the dashboard through WebSockets. No polling, no delayed reports.

### Observability
- **LangSmith** for agent execution tracing — every step in the pipeline is logged and inspectable
- **Sentry** for error tracking with pre-configured alert rules
- **Vercel Analytics** for frontend behavior tracking

---

## Tech Stack

| Layer | Technology |
|-------|------------|
| Backend | Python · FastAPI |
| Frontend | Next.js · TypeScript · React Query · Tailwind · shadcn/ui |
| Database | PostgreSQL |
| Queue / Async | Redis · ARQ |
| Real-time | WebSockets · Socket.IO |
| AI | OpenAI · LangChain · LangGraph |
| Auth | Auth0 |
| Infra | Railway (backend) · Vercel (frontend) |

---

## Screenshots

### Dashboard
![Dashboard](./assets/dashboard.png)

### Campaign Creation
![Campaign](./assets/campaign.png)

### Email Review (Human-in-the-Loop)
![Review](./assets/review.png)

---

## About This Repo

This repository is a product and architecture overview. The full implementation — backend, frontend, agent pipeline, CAO engine, and infrastructure — lives in a private repository.

For a deeper look at the system design, see [ARCHITECTURE.md](./ARCHITECTURE.md).
For the project roadmap, see [ROADMAP.md](./ROADMAP.md).

---

## Author

Built end-to-end by a single engineer, covering product design, backend architecture, agent workflows, and infrastructure.
