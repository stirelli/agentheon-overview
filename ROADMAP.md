# Roadmap

A high-level view of where Agentheon has been and where it's headed.

---

## Phase 1 — Core MVP ✓

The foundation is built and functional:

- **Agent pipeline** — Research, enrichment, and copywriting agents orchestrated via LangGraph
- **Campaign management** — Full lifecycle from lead import to email sending
- **Human-in-the-loop review** — Approve, edit, or reject generated emails before sending
- **Real-time event streaming** — WebSocket-based live updates across the UI
- **Webhook integration** — Delivery event tracking (opens, replies, bounces)
- **Observability** — LangSmith tracing, Sentry error tracking, structured logging
- **Auth and multi-tenancy** — Auth0 integration with isolated user environments
- **E2E test coverage** — Automated tests across the critical paths

---

## Phase 2 — Expansion (Current)

Transitioning from a working backend to a complete product:

- **Domain management UI** — Visibility and control over sending domains, health metrics, and capacity
- **CAO implementation** — Intelligent capacity allocation across domains with automated warm-up
- **Metrics dashboard** — Real-time campaign analytics with open/reply/bounce breakdowns
- **Improved campaign builder** — Multi-step creation flow with source selection and configuration
- **Billing integration** — Usage-based pricing tiers

---

## Phase 3 — Learning and Memory

Making the system smarter over time:

- **Long-term memory** — Persist context across campaigns so the system learns what works for each user's audience
- **Outcome-based feedback** — Track which email styles, structures, and angles drive replies, and feed that back into generation
- **Confidence scoring** — Agents assess their own output quality; low-confidence results are flagged for review
- **Dynamic plan refinement** — The pipeline adapts its approach based on accumulated data rather than running the same sequence every time

---

## Phase 4 — Integrations and Expansion

Broadening what the platform connects to and what it can do:

- **CRM integrations** — Sync campaign activity and prospect status with external CRMs
- **Email provider flexibility** — Support for multiple sending providers
- **Additional data sources** — Beyond Apollo, integrate other prospecting and enrichment sources
- **Use case expansion** — Apply the agent orchestration framework to adjacent workflows (recruiting outreach, customer re-engagement)

---

## What's Not on the Roadmap

Some things are intentionally deferred:

- **On-premise deployment** — Not a priority while the product is evolving rapidly
- **No-code flow builder** — The agent pipeline benefits from being code-defined; visual builders add complexity without clear value at this stage
- **Multi-channel outreach** (LinkedIn, SMS) — Focus remains on email until the core loop is fully optimized
