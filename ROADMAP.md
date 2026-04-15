# Roadmap

Execution history and forward plan.

---

## Phase 0 — Foundation & Reliability ✓

Everything required for safe operation:

- Agent pipeline (Navigator → Planner → Coordinator) on LangGraph
- Campaign state machine with validated transitions
- Human-in-the-loop approval (inline edit, AI regenerate, reject-with-feedback)
- Real-time event pipeline (Redis Pub/Sub → Socket.IO)
- Resend webhook integration with signature verification + idempotency
- Retry + exponential backoff on every external API
- Campaign circuit breaker (auto-pause after 3 failures)
- Fernet encryption for stored credentials, PII-sanitized logs
- LangGraph persistent checkpointing (AsyncPostgresSaver)
- Observability: LangSmith traces, Sentry, structured logs

## Phase 1 — Feedback Learning Loop ✓

The system's moat. All three tiers operational:

- **Tier 1** — `EmailPerformanceAnalyzer` + `PromptComposer` · real-time SQL stats injected into prompts
- **Tier 2** — `SubjectLineBandit` · Thompson Sampling with persistent Beta state per campaign variant
- **Tier 3** — `EmailInsightStore` · pgvector (512d, HNSW) with post-campaign insight extraction
- **LangSmith feedback** · webhook events → `create_feedback` scores on generation traces

## Phase 1.5 — Production Readiness ✓

- Access control audit — ownership validated on every endpoint
- Draft → Campaign conversion wired end-to-end
- ARQ task chain auto-fires (import → enrich → generate)
- Dead code removed (SendEmailAgent eliminated from generation path)
- State machine bypass fixed (direct status updates forbidden)

## Phase 2 — Performance & Deployment ← CURRENT

**Performance** ✓
- Parallel LLM generation (`asyncio.Semaphore(5)`) — 5x faster campaigns
- DB connection pool tuning (size=20, overflow=10, recycle=3600)
- N+1 elimination (approve/reject, draft creation)
- Socket.IO Redis adapter for horizontal scaling
- Health check validates DB + Redis (ALB-compatible, returns 503 on failure)
- Frontend bundle reduction: consolidated icons (-40 KB), lodash deep imports (-72 KB), SWC minification re-enabled
- Polling reduced 3s → 30s (socket-driven updates)

**AWS migration** — fully designed in Terraform, ready to apply. See [INFRASTRUCTURE.md](./INFRASTRUCTURE.md).

---

## Phase 3 — Stack Modernization

- LangChain/LangGraph 0.3+ (prompt caching support, deprecation cleanup)
- Claude 4 Sonnet A/B evaluation for email generation
- Next.js 15 App Router migration (Server Components, streaming)
- Vercel AI SDK for AI response streaming
- MCP (Model Context Protocol) support in tool registry — pluggable integrations

## Phase 4 — Feature Expansion

- ResearchAgent wired into enrichment stage (currently scaffolded)
- Browser research via `browser-use` for LinkedIn/company pages
- Multi-step email sequences with engagement-based triggers
- Gmail/Outlook OAuth sending (better deliverability than shared domains)
- CRM sync (Salesforce, HubSpot via MCP)
- Analytics dashboard — bandit convergence, insights visualization, performance by segment

## Phase 5 — Cognitive Loop

- Predictive confidence scoring (auto-approve above threshold)
- Cross-campaign learning (semantic similarity transfer)
- Smart send-time optimization per contact
- Self-improving prompts via LangSmith evals + bandit testing
- Autonomous sequence optimization

---

## Explicitly Off-Roadmap

- **Per-customer fine-tuning** — prompt injection + feedback loop produces comparable results without retraining cost
- **Kubernetes** — ECS Fargate is sufficient for single-team scale; K8s is operational overhead without benefit here
- **No-code flow builder** — visual agent builders add UX complexity without differentiated value
- **Multi-channel (LinkedIn, SMS)** — focus on email until the learning loop is proven at scale

---

## Design Philosophy

Add complexity only when the pain is real. No preemptive microservices. No abstraction layers without a caller. The bet is on compounding learning, not architectural novelty.
