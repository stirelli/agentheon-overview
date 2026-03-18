# Architecture

This document describes the high-level architecture of Agentheon, the design decisions behind it, and the trade-offs involved.

---

## Design Goals

1. **Modularity** — Each part of the pipeline (research, generation, sending, tracking) is an independent component that can be developed, tested, and replaced in isolation.
2. **Observability** — Every agent execution, email event, and system error is traceable. If something goes wrong, you can see exactly where and why.
3. **Human control** — Automation handles the heavy lifting, but humans make the final call on what gets sent.
4. **Deliverability as infrastructure** — Domain health and sending capacity are managed at the system level, not left to the user to figure out.

---

## System Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        Frontend                             │
│                  Next.js · TypeScript                        │
│         React Query · Socket.IO · shadcn/ui                 │
│                                                             │
│  ┌───────────┐  ┌────────────┐  ┌────────────┐             │
│  │ Dashboard  │  │ Campaign   │  │ Review     │             │
│  │ & Metrics  │  │ Builder    │  │ Interface  │             │
│  └───────────┘  └────────────┘  └────────────┘             │
└────────────────────────┬────────────────────────────────────┘
                         │ REST + WebSocket
┌────────────────────────▼────────────────────────────────────┐
│                     Backend API                             │
│                  Python · FastAPI                            │
│                                                             │
│  ┌───────────┐  ┌────────────┐  ┌────────────┐             │
│  │ Campaign   │  │ Auth       │  │ Webhook    │             │
│  │ Management │  │ (Auth0)    │  │ Handlers   │             │
│  └─────┬─────┘  └────────────┘  └─────┬──────┘             │
│        │                               │                    │
│  ┌─────▼──────────────────────────────▼──────┐             │
│  │          Agent Workflow Layer              │             │
│  │       LangChain · LangGraph               │             │
│  │                                           │             │
│  │  Research → Enrichment → Copywriting      │             │
│  └─────────────────┬─────────────────────────┘             │
│                    │                                        │
│  ┌─────────────────▼─────────────────────────┐             │
│  │        Deliverability Layer                │             │
│  │    CAO · Domain Warmup · Send Limits      │             │
│  └───────────────────────────────────────────┘             │
└────────────────────────┬────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
   ┌──────────┐   ┌──────────┐   ┌──────────────┐
   │PostgreSQL│   │  Redis   │   │Email Provider│
   │          │   │  (ARQ)   │   │              │
   └──────────┘   └──────────┘   └──────────────┘
```

---

## Frontend

**Next.js (Pages Router) · TypeScript · React Query v5 · Tailwind · shadcn/ui**

The frontend is a single-page application responsible for:

- **Campaign management** — creating campaigns, selecting prospect sources, configuring sending parameters
- **Email review** — displaying generated emails for human approval, with edit and reject actions
- **Real-time dashboard** — live metrics for opens, replies, bounces, streamed via Socket.IO
- **Domain management** — visibility into domain health and sending capacity

State management uses React Query with optimistic updates. Real-time events arrive through a persistent WebSocket connection and trigger cache invalidations so the UI stays current without polling.

---

## Backend API

**Python · FastAPI**

The API layer handles:

- **Campaign lifecycle** — state machine managing campaign creation, activation, pausing, and completion
- **Authentication** — Auth0 integration with JWT validation
- **Webhook ingestion** — receiving delivery events (opens, clicks, bounces, replies) from the email provider and writing them to the database
- **Agent job dispatch** — queuing research and generation tasks to async workers via Redis/ARQ
- **WebSocket broadcasting** — pushing real-time events to connected frontend clients

The API does not run agent workflows synchronously. All AI-related work is dispatched to background workers to keep API response times predictable.

---

## Database — PostgreSQL

PostgreSQL is the system of record for:

- Campaigns, prospects, and generated emails
- Email event history (opens, replies, bounces)
- Domain configuration and sending capacity state
- User accounts and settings

The schema is designed around the campaign lifecycle — prospects move through defined states (imported → researched → generated → reviewed → sent → tracked) and each transition is recorded.

---

## Async Workers — Redis / ARQ

Long-running tasks are processed by ARQ workers backed by Redis:

- **Research jobs** — agent-based prospect research and enrichment
- **Generation jobs** — email copywriting using LLM pipelines
- **Sending jobs** — email dispatch respecting CAO capacity limits
- **Analytics processing** — aggregating webhook events into campaign-level metrics

This keeps the API responsive and allows the system to process large prospect lists without blocking.

---

## Agent Workflow Layer

**LangChain · LangGraph**

The core AI pipeline is built as a directed graph of specialized agents:

1. **Research Agent** — gathers prospect and company context from external sources
2. **Enrichment Agent** — structures and scores the research output
3. **Copywriting Agent** — generates personalized emails grounded in the enriched data

Each agent is:
- **Independently testable** — can be run in isolation with mocked inputs
- **Traced** — every execution is logged to LangSmith with full input/output visibility
- **Configurable** — model selection, prompts, and parameters are adjustable per agent

The graph topology is managed by LangGraph, which handles state transitions, conditional branching, and error recovery within the pipeline.

---

## Human-in-the-Loop

The review system sits between generation and sending:

```
Agent generates email → Email enters review queue → User approves/edits/rejects → Approved emails enter send queue
```

This is not optional — every email passes through review before sending. The review interface shows the generated content alongside the research context that informed it, so users can judge whether the personalization is accurate.

Rejected emails can be regenerated with adjusted parameters.

---

## Deliverability Layer — CAO

The Capacity Allocation Optimizer (CAO) manages sending at the domain level:

- **Domain warm-up** — new domains start with low daily sending limits that increase gradually as reputation builds
- **Capacity allocation** — available daily capacity is split between campaign emails and warm-up traffic
- **Health monitoring** — bounce rates and complaint signals feed back into capacity adjustments
- **Multi-domain distribution** — campaigns can spread volume across multiple sending domains to reduce per-domain risk

The CAO runs as part of the sending pipeline and enforces limits before any email reaches the provider.

---

## Analytics + Webhooks

Email events flow through webhooks from the email provider:

```
Email Provider → Webhook endpoint → Event processing → Database → WebSocket → Dashboard
```

Events are processed asynchronously and written to PostgreSQL. Aggregated metrics (open rate, reply rate, bounce rate) are computed and pushed to the frontend via WebSocket so the dashboard updates in real time.

---

## Observability

| Tool | Purpose |
|------|---------|
| LangSmith | Agent execution tracing — full visibility into every step of the AI pipeline |
| Sentry | Error tracking with alert rules for critical failures |
| Vercel Analytics | Frontend performance and user behavior |
| Application logs | Structured logging across all backend services |

The goal is that when something fails — an agent produces bad output, an email bounces unexpectedly, a webhook is malformed — you can trace the full chain from user action to system response.

---

## Deployment

| Component | Platform | Rationale |
|-----------|----------|-----------|
| Backend API + Workers | Railway | Simple container deployment, built-in PostgreSQL and Redis, easy scaling |
| Frontend | Vercel | Native Next.js support, edge caching, zero-config deploys |
| Auth | Auth0 | Managed authentication, JWT-based, avoids building auth from scratch |

---

## Key Trade-offs

### Flexibility vs. Control
The agent pipeline uses LangGraph for flexible orchestration, but the overall campaign flow follows a strict state machine. Agents can adapt within their step; the system enforces the sequence.

### Automation vs. Trust
The system generates emails automatically but requires human approval before sending. This adds friction to the workflow but prevents bad emails from reaching prospects — a trade-off that's worth it when domain reputation is at stake.

### Speed vs. Deliverability
The CAO could send more emails faster, but throttles volume to protect domain health. Short-term throughput is sacrificed for long-term sending capacity.

### Monolith vs. Microservices
The backend is a modular monolith — logically separated components (API, workers, agents, CAO) in a single deployable unit. This allows faster iteration and lower operational overhead than a distributed setup, while maintaining clear internal boundaries that make future extraction straightforward if needed.
