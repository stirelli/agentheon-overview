# Infrastructure

Deployment architecture — current production on Railway, AWS migration fully designed in Terraform.

---

## Current Production — Railway

Agentheon runs on Railway today. Deployment is Git-based: `push to staging` → auto-build → rolling deploy.

| Component | Service |
|-----------|---------|
| Backend API + Worker | Railway container (single Dockerfile, `start-with-worker.sh` runs both) |
| Database | Railway PostgreSQL (with pgvector extension) |
| Redis | Railway Redis (ARQ queue + Pub/Sub) |
| Frontend | Vercel (Next.js, edge-cached) |
| Secrets | Railway environment variables |

**Why Railway initially**: zero config for a single-container app, managed Postgres and Redis out of the box, free SSL, fast iteration. Trade-off: limited scaling beyond a single region, no infra-as-code primitive, pricing less predictable at volume.

---

## Target — AWS (Terraform, ready to apply)

The AWS migration is fully designed and coded in Terraform as a modular project. The architecture is hybrid — ECS Fargate for long-running workloads, Lambda for event-driven bursts.

```
                      Route 53 (DNS) + ACM (TLS)
                              │
                       CloudFront (optional, future static frontend)
                              │
                              ▼
                    ┌─────────────────────┐
                    │   ALB (HTTPS + WS)  │
                    │   Path-based rules: │
                    │   /api/* → backend  │
                    │   /ws/*  → backend  │
                    │   /*     → frontend │
                    └─────────┬───────────┘
                              │
                 ┌────────────┼────────────┐
                 ▼            ▼            ▼
          ┌────────────┐ ┌────────┐ ┌──────────────┐
          │ ECS Fargate │ │ ECS    │ │ ECS Fargate  │
          │ Backend API │ │ Worker │ │ Frontend     │
          │ (WebSocket) │ │ (ARQ)  │ │ (Next.js)    │
          └──────┬──────┘ └───┬────┘ └──────────────┘
                 │             │
         ┌───────┴─────────────┴───────┐
         ▼                             ▼
   ┌─────────────┐              ┌──────────────┐
   │ RDS         │              │ ElastiCache  │
   │ PostgreSQL  │              │ Redis        │
   │ + pgvector  │              │              │
   └─────────────┘              └──────────────┘

   ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

   Webhooks (isolated):
   Resend POST → API Gateway HTTP API → Lambda (webhook handler)
                                          │
                                          ├─ writes to RDS
                                          └─ publishes to ElastiCache Pub/Sub → Socket.IO
```

### AWS Services Used

Every service specified in Terraform, with its role:

| Service | Role |
|---------|------|
| **VPC + Subnets** | 2 public + 2 private subnets across 2 AZs |
| **NAT Gateway** | Private subnets' egress to internet (OpenAI, Resend APIs) |
| **Security Groups** | ALB (public 80/443) · ECS (only from ALB) · RDS (only from ECS/Lambda) · Redis (same) · Lambda (egress only) |
| **ALB (Application Load Balancer)** | HTTPS termination · path-based routing · sticky sessions (cookie `io` for Socket.IO) · health check on `/api/v1/health` |
| **ECS Fargate** | Cluster with 3 services: `backend-api` (1 task, 512 CPU / 1 GB) · `worker` (1 task, 1024 CPU / 2 GB) · `frontend` (1 task, 256 CPU / 512 MB). Circuit breaker + rollback on failed deploys |
| **ECR** | 2 repositories (backend, frontend) with lifecycle policies (keep last 10 images) |
| **RDS PostgreSQL 15** | `db.t3.micro` (free tier) · encrypted at rest · daily backups · pgvector extension |
| **ElastiCache Redis 7** | `cache.t3.micro` (free tier) · ARQ queue + Pub/Sub |
| **Lambda + API Gateway v2** | Webhook handler isolated from main API · reserved concurrency = 50 · VPC-attached for RDS/Redis access |
| **Secrets Manager** | All API keys (OpenAI, Resend, Auth0, etc.) pulled into ECS tasks as container secrets — never in env vars |
| **Parameter Store** | Non-sensitive config (AUTH0_DOMAIN, SENTRY_DSN, log levels) |
| **CloudWatch Logs** | 14-day retention per service log group |
| **CloudWatch Alarms** | API CPU > 80%, 5xx rate > 1%, DB connections > 80% of pool |
| **Route 53** | Hosted zone for agentheon.ai, alias to ALB |
| **ACM** | Wildcard TLS certificate, DNS-validated |
| **IAM** | Separate execution role (pull images, read secrets) and task role (AWS API calls). Least-privilege policies |
| **Terraform** | 8 modules: networking, database, redis, ecr, secrets, ecs, lambda, frontend. Remote state in S3 (optional) |

### Why hybrid (Fargate + Lambda)

| Workload | Service | Rationale |
|----------|---------|-----------|
| Backend API | ECS Fargate | Socket.IO needs persistent connections — Lambda can't do that |
| Worker | ECS Fargate | Generation jobs can run 30 min — Lambda caps at 15 |
| Frontend | ECS Fargate (initially) | Auth proxy needs server execution; static migration is a post-AuthApp-Router step |
| Webhooks | Lambda | Bursty, stateless, 5-second processing window. Isolates from user traffic so a 1000-webhook burst doesn't affect dashboard latency |
| Nightly aggregation (future) | Lambda + EventBridge | Scheduled, short-lived, zero idle cost |

### CI/CD — GitHub Actions

Path-based change detection means only what changed gets deployed:

```
push to main
    │
    ├─ detect-changes (paths-filter)
    │   ├─ backend: rebuild + push to ECR + update ECS services (api, worker)
    │   ├─ frontend: rebuild + push to ECR + update ECS service (frontend)
    │   └─ lambdas: zip + aws lambda update-function-code
```

A change to the webhook Lambda deploys in ~5 seconds. A change to the backend rolls through ECS in ~2 minutes with health check validation. A change to an unrelated file deploys nothing.

---

## Cost Optimization

This is where the AWS design pays off. Cost-conscious architecture choices at every layer.

### MVP scale (learning / single-user, <100 contacts/mo)

Target: **~$60–80/month**

| Service | Strategy | Cost |
|---------|----------|------|
| ECS Fargate (3 tasks, minimal sizing) | Always-on baseline (no free tier) | ~$50 |
| RDS PostgreSQL | `db.t3.micro` — 12-month free tier | $0 |
| ElastiCache Redis | `cache.t3.micro` — 12-month free tier | $0 |
| ALB | 750 free hours/month | ~$5 after free allowance |
| Lambda | 1M free requests/month — nowhere near it | $0 |
| API Gateway HTTP API | 1M free requests/month | $0 |
| NAT Gateway | Single AZ (not HA) saves 50% | ~$32 |
| ECR + CloudWatch + Route 53 + misc | | ~$5 |

Optimizations specific to MVP:

- **Single-AZ NAT Gateway** instead of one per AZ → -$32/mo (acceptable for non-critical learning environment)
- **Free-tier eligible DB and Redis** for 12 months
- **No CloudFront** initially (ALB serves frontend directly; add CDN when frontend traffic justifies)
- **Fargate SPOT for workers** — 70% discount. Workers tolerate interruption (ARQ re-enqueues on failure).
- **ECR lifecycle policies** — auto-delete images beyond the last 10
- **CloudWatch log retention** = 14 days (not default indefinite)

Optional: eliminate NAT entirely by running ECS tasks in public subnets with `assign_public_ip=true`. Saves $32/mo, costs some security isolation. Viable for learning environments.

**Running cost with every free tier exhausted**: ~$75/mo.

### Growth scale (~500 users)

Target: **~$300–500/month**

- ECS Fargate scales to 2–4 API tasks + 3–5 workers (auto-scaling on CPU + SQS queue depth analog)
- RDS bumped to `db.t3.medium` with a read replica for `EmailPerformanceAnalyzer` heavy queries
- ElastiCache bumped to `cache.t3.small`
- Fargate SPOT continues for workers

### Production scale (1K–5K concurrent users)

Target: **~$1,200–2,000/month**

- Aurora PostgreSQL for faster failover and read replica autoscaling
- ElastiCache cluster mode for Pub/Sub throughput
- ECS auto-scaling targets: API (2–8 tasks, CPU-triggered), workers-per-type (5–15 tasks, queue-depth-triggered)
- CloudFront in front of the ALB for static asset caching
- Multi-AZ NAT Gateways for HA
- Potential: move workers to EC2 Spot fleet for further savings at volume

### The non-infrastructure reality

At the growth/production scale, **infrastructure is 15-20% of total cost**. The other 80-85% is:

- OpenAI API (~$1,100/mo at 200K emails/mo with gpt-5-mini)
- Tavily enrichment (~$2,000/mo at same volume)
- Resend email sending (~$200/mo after free tier)

This drives AI-cost-specific optimization work: prompt caching (Anthropic), model routing (cheap models for simple tasks, expensive for critical ones), result caching (avoid regenerating similar emails), and aggressive observability on tokens-per-task. Cost tracking per-email via `generation_metadata` enables this analysis.

### Lifecycle: teardown and recreate

Because the entire AWS stack is in Terraform, the full environment can be torn down with `terraform destroy` and recreated in ~10 minutes. For non-24/7 learning environments this is meaningful — running the stack only during active development cuts cost 70%+.

---

## Migration Plan (Railway → AWS)

Actual execution order, from the Terraform plan:

1. `terraform apply -target=module.networking` — VPC, subnets, SGs (3 min)
2. `terraform apply -target=module.database -target=module.redis -target=module.ecr -target=module.secrets` — data layer (8 min)
3. Build and push Docker images to ECR
4. Update `terraform.tfvars` with image URIs
5. `terraform apply` — ECS services, Lambda, ALB (5 min)
6. Run migrations via `aws ecs run-task` with `alembic upgrade head`
7. `psql → CREATE EXTENSION vector;` on RDS
8. Update Resend webhook URL to API Gateway endpoint
9. Update NextAuth/Auth0 callback URLs to ALB DNS
10. Smoke test: `curl http://<alb-dns>/api/v1/health` → 200 with `{database: connected, redis: connected}`
11. DNS cutover via Route 53

Rollback plan: Railway remains running during cutover; DNS flip is reversible.
