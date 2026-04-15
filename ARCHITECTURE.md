# Architecture

System design of Agentheon. Focus: real architectural decisions and why they were made.

> Public overview. Source code is private.

---

## Design Principles

1. **Compounding learning over static prompts** — every campaign makes the next one better without per-customer fine-tuning.
2. **Async everything on hot paths** — API never blocks on LLM calls. Workers do the heavy lifting.
3. **State machine over ad-hoc logic** — campaign transitions are explicit and validated.
4. **Observability from day one** — every LLM call traced, every webhook event logged, every error captured.
5. **Modular monolith** — clear internal boundaries, single deploy unit. Microservices are a future option, not a starting point.

---

## Component Topology

```
┌──────────────────────────────────────────────────────────────────────┐
│                       Browser (Client)                                │
│           Next.js · TypeScript · React Query · Socket.IO             │
└────────────────────────────┬─────────────────────────────────────────┘
                             │ HTTPS (REST + WebSocket)
┌────────────────────────────▼─────────────────────────────────────────┐
│                    Next.js Server (Auth Proxy)                        │
│       NextAuth (Google OAuth, JWE)  →  HS256 JWT to backend          │
└────────────────────────────┬─────────────────────────────────────────┘
                             │
┌────────────────────────────▼─────────────────────────────────────────┐
│                     Backend API (FastAPI)                             │
│                                                                       │
│   Campaign CRUD · Approval · Webhooks · Socket.IO bridge             │
│   JWT validation · Ownership checks · State machine · Rate limits    │
└──────┬─────────────────────┬─────────────────────┬───────────────────┘
       │ enqueue             │ write               │ publish
       ▼                     ▼                     ▼
┌──────────────┐   ┌──────────────────┐   ┌────────────────────┐
│ Redis (ARQ)  │   │   PostgreSQL     │   │ Redis (Pub/Sub)    │
│              │   │   + pgvector     │   │                    │
│ Job queue    │   │                  │   │ Real-time events   │
└──────┬───────┘   │ • Campaigns      │   └──────────▲─────────┘
       │ dequeue   │ • Contacts       │              │ publish
       ▼           │ • Drafts         │              │
┌──────────────┐   │ • Email logs     │              │
│ ARQ Workers  │───┤ • Bandit stats   │──────────────┘
│              │   │ • Semantic mem   │
│ Import       │   │ • LG checkpoints │
│ Enrich       │   │ • Idempotency    │
│ Generate     │   └──────────────────┘
│ Send         │
│ Learn        │
└──────┬───────┘
       │ external APIs
       ▼
┌──────────────────────────────────────────────────┐
│ OpenAI · Tavily · Perplexity · Resend · Apollo   │
│ LangSmith · Sentry                                │
└──────────────────────────────────────────────────┘
```

---

## The Learning Loop

The architectural differentiator. Three tiers, all operational.

### Tier 1 — Prompt-Level Learning (Real-Time SQL)

Before generating each email, `EmailPerformanceAnalyzer` runs three queries scoped to the user's history:

1. Top-performing past emails for the target segment (industry + title)
2. Subject line pattern stats (open rate, reply rate per pattern)
3. Anti-patterns from bounced/complained emails

Results are formatted into a `PERFORMANCE INSIGHTS` section of the prompt. Cold start is handled — first-time users skip injection gracefully.

### Tier 2 — Multi-Armed Bandit for Subject Lines

`SubjectLineBandit` — Thompson Sampling over Beta(α, β) distributions, implemented with numpy.

- 3 default styles per campaign: question · specific-interest · common-ground
- State persisted per campaign in `subject_variant_stats`
- Each send: `np.random.beta(α, β)` selects the variant
- Open webhook event: atomic `UPDATE α = α + 1`
- Converges to the best variant within ~50 sends per campaign

### Tier 3 — Semantic Memory (pgvector)

After campaigns run, `EmailInsightStore.extract_campaign_insights` converts winning patterns into natural language, embeds with `text-embedding-3-small` (512d), and stores with HNSW index.

Future campaigns query by audience similarity; top-3 results are injected as a `SEMANTIC INSIGHTS` prompt section. Stored per-user (FK to `sdr_users`), so insights are private.

### Closing the Loop via LangSmith

Every email draft stores `generation_metadata` with `trace_id`, `input_tokens`, `output_tokens`, `model`, `prompt_version`, `variant_id`, `generation_time_ms`.

When Resend webhooks fire:

| Event | LangSmith score |
|-------|----------------|
| `replied` | 1.0 |
| `clicked` | 0.7 |
| `opened` | 0.3 |
| `bounced` / `complained` | -1.0 |

Scores flow through `langsmith.Client.create_feedback` asynchronously. Low-scoring traces surface bad prompts for review. High-scoring traces surface patterns worth extracting to Tier 3.

---

## Composable Prompts

The EmailAgent prompt is not a monolithic string. It's assembled from independent sections with defined cache semantics:

| Section | Volatility | Contents |
|---------|-----------|----------|
| base_instructions | cacheable | Role, format, structured output schema |
| campaign_type | cacheable | Job-search vs sales-outreach vs networking |
| tone_guidelines | cacheable | Voice, formality, CTA style |
| compliance_rules | cacheable | Anti-spam, length limits |
| performance_insights | volatile | Tier 1 SQL results |
| semantic_insights | volatile | Tier 3 pgvector results |
| few_shot_examples | volatile | Real past winners for audience |
| bandit_variant | volatile | Tier 2 selected style instruction |
| contact_context | volatile | Enrichment data for this prospect |

Cacheable sections are identical across a campaign's 50+ emails — ready for Anthropic prompt caching once the system migrates off GPT-5-mini. Volatile sections change per contact.

Output is structured JSON via `bind_tools` (`EmailOutput` Pydantic model with `subject`, `body_html`, `body_text`, `personalization_score`, `reasoning`).

---

## Agent System

LangGraph-based orchestration:

```
Orchestrator → Navigator → Planner → Coordinator → [Specialized Agents]
```

- **Orchestrator** — top-level workflow entry, owns the LangSmith trace context
- **Navigator** — LLM-based intent interpretation, handles clarification requests
- **DynamicSalesPlanner** — generates execution plans with tool/agent validation
- **Coordinator** — executes plans via LangGraph with dynamic graph building
- **EmailAgent** — the only agent in the generation hot path. Composable prompts, structured output
- **ResearchAgent** — Tavily + Perplexity (scaffolded; full wiring in progress)
- **Tool Registry** — Pydantic-validated, dynamic discovery

LangGraph workflow state persists via `AsyncPostgresSaver` — generation paused for human review can resume hours or days later across worker restarts.

---

## Campaign State Machine

```
DRAFT → IMPORTING → ENRICHING → GENERATING → REVIEWING → SCHEDULED → ACTIVE → COMPLETED
                                     ↓                      ↓
                                  PAUSED ← ─ ─ ─ ─ ─ ─ ─ PAUSED
```

Contact-level state machine runs in parallel: `IMPORTED → ENRICHED → GENERATED → APPROVED → QUEUED → SENT → DELIVERED/OPENED/CLICKED/REPLIED/BOUNCED`.

**Rule**: direct status updates are forbidden. All transitions go through `CampaignStateMachine.transition()` which validates source→target legality and runs post-transition hooks.

---

## Real-Time Architecture

```
Worker fires event
   → Redis PUBLISH "orchestration:updates:<campaign_id>"
      → API Pub/Sub listener (long-lived asyncio task)
         → Socket.IO (Redis adapter for multi-instance)
            → Browser room: campaign_<id> or sdr_global
```

Everything that matters in the UI comes through this pipeline: generation progress, webhook events, approval requests, campaign status. Polling is a 30s fallback only.

The Socket.IO Redis adapter is essential — without it, horizontally scaling the API breaks real-time delivery. The ALB uses sticky sessions (cookie `io`) because Socket.IO's HTTP polling→WebSocket upgrade requires consistent backend affinity.

---

## Observability Stack

| Tool | Coverage |
|------|---------|
| LangSmith | Every LLM call. Full I/O. Feedback scores from webhooks. Per-trace cost tracking |
| Sentry | Backend + frontend error tracking. 10 pre-configured alert rules |
| Loguru | Structured logs with a PII-sanitization filter on every line |
| Vercel Analytics | Frontend user behavior |
| CloudWatch (AWS target) | Infra-level logs, metrics, alarms |

Per-email `generation_metadata` enables analysis queries like "what's my cost-per-reply for gpt-5-mini vs gpt-4.1?" or "which prompt_version has the best performance_score?"

---

## Resilience Patterns

- **Retry with exponential backoff** — every external API call (OpenAI 429, Resend 5xx, Tavily timeouts, Apollo rate limits)
- **Webhook idempotency** — Resend sends duplicates; SHA256 hash stored in `processed_webhook_events` prevents double-processing
- **Circuit breaker** — campaign auto-pauses after 3 consecutive send failures
- **Connection pooling** — explicitly tuned (size=20, overflow=10, recycle=3600) to prevent exhaustion under load
- **Controlled parallelism** — email generation runs 5 concurrent LLM calls via `asyncio.Semaphore`, balancing throughput against rate limits
- **Graceful fallback** — missing API keys degrade to mock mode in development, not crash

---

## Data Model Highlights

27 Alembic migrations, all auto-generated, with explicit constraint naming conventions for migration consistency.

Notable tables:

| Table | Purpose |
|-------|---------|
| `sdr_campaigns` | Campaign state, settings (JSONB), counters |
| `sdr_campaign_contacts` | Per-contact state, enrichment data (JSONB), generation metadata (JSONB) |
| `email_drafts` | Pre-send drafts with approval state, revised subject/body |
| `email_logs` | Delivery tracking (resend_email_id, status, timestamps, open/click counts) |
| `subject_variant_stats` | Bandit state (α, β) per campaign variant |
| `email_insights` | pgvector store (512d embeddings, HNSW index, FK to user) |
| `processed_webhook_events` | Idempotency hashes |
| `rate_limit_usage` | Daily/hourly send tracking per user |
| LangGraph checkpoint tables | `checkpoint_blobs`, `checkpoint_migrations`, `checkpoint_writes`, `checkpoints` |

Sensitive fields (API keys, OAuth tokens) are Fernet-encrypted at rest.

---

## Key Trade-offs

### Prompt injection over fine-tuning

Competitors fine-tune per customer. Agentheon injects historical data into prompts. Bet: prompt injection is faster, cheaper, transferable across models, and — with multi-tier feedback — produces comparable results without per-customer retraining cost.

### Human-in-the-loop, always

Every email requires explicit approval. Adds friction. But: sending reputation takes months to build and seconds to destroy, and an LLM hallucinating a compliment about the wrong company gets the domain mass-reported. Review is the feature.

### Modular monolith over microservices

One codebase, one deploy unit, clear boundaries between API / agents / workers / feedback. Extraction is possible later; premature decomposition is a cost that pays nothing at current scale.

### Fargate over Kubernetes

For the AWS migration: ECS Fargate, not EKS. With three deployable services and one engineer, Kubernetes is overhead without benefit. The conceptual model (task definitions, services, target groups) maps cleanly to Kubernetes if scale later demands it.
