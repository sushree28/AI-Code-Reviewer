# AI Code Reviewer

An AI-powered GitHub App that automatically reviews Pull Requests the moment they're opened. Four specialized agents — static analysis, security, style, and architecture — analyze the diff in parallel using LangGraph and post inline comments directly on the PR.

## Features

- **Automatic PR review** — triggers on every `pull_request` webhook event, no manual invocation needed
- **Four parallel review agents** — static analysis (complexity, unused variables, naming), security (OWASP Top 10, hardcoded secrets, injection), style (formatting, readability), and architecture (separation of concerns, error handling)
- **Inline GitHub comments** — findings are posted directly on the relevant lines of the diff
- **Pattern learning** — on merge, accepted findings are stored and used to improve future reviews
- **Full observability** — LangFuse traces every LLM call (prompt, response, tokens, cost); Prometheus + Grafana track request rate, latency, and error rate per service
- **Weekly quality evaluation** — a scheduled job scores recent review comments with RAGAS (faithfulness, answer relevancy) and fails if quality drops

## Architecture

```
GitHub ──webhook──▶ gateway ──▶ webhook ──▶ Celery/Redis ──▶ orchestrator ──▶ GitHub API (fetch diff)
                   (verify HMAC) (persist to     (queue)      │
                                  PostgreSQL)                  ├─ static analysis agent ─┐
                                                                ├─ security agent          ├─ LangGraph, run in parallel
                                                                ├─ style agent             │
                                                                └─ architecture agent     ─┘
                                                                     │
                                                                     ▼
                                                                 reviewer ──▶ GitHub API (post inline comments)
                                                                     │
                                                                [on merge]
                                                                     ▼
                                                                 learner ──▶ PostgreSQL (store patterns)
```

| Service | Responsibility |
|---|---|
| `gateway` | Verifies GitHub webhook signatures, forwards valid events |
| `webhook` | Parses events, persists PRs to PostgreSQL, queues analysis jobs |
| `orchestrator` | Fetches the PR diff and runs the four review agents via LangGraph |
| `reviewer` | Posts agent findings as inline comments on the GitHub PR |
| `learner` | Stores accepted findings as patterns after a PR merges |

**Infrastructure:** AWS EKS (Kubernetes) · RDS PostgreSQL · ElastiCache Redis · ECR · S3 · LangFuse (tracing) · Prometheus + Grafana (metrics)

## Tech Stack

- **AI/Orchestration:** LangGraph, LangChain, OpenAI (GPT-4o-mini), LangFuse
- **Services:** FastAPI, Celery, SQLAlchemy (async) + asyncpg, Alembic
- **Infrastructure:** Terraform, AWS EKS/RDS/ElastiCache/ECR/S3, Kubernetes, Helm
- **Observability:** Prometheus, Grafana, LangFuse, RAGAS

## Getting Started

This system runs as five containerized services on Kubernetes and requires AWS infrastructure (EKS, RDS, ElastiCache), a GitHub App, an OpenAI API key, and a LangFuse account.

For the complete step-by-step setup — accounts, infrastructure provisioning, GitHub App configuration, Kubernetes deployment, monitoring, and teardown — see **[docs/DEPLOYMENT.md](docs/DEPLOYMENT.md)**.

### Repository Layout

```
services/
  gateway/        webhook signature verification
  webhook/        event ingestion + job queueing
  orchestrator/   LangGraph review agents
  reviewer/       posts comments to GitHub
  learner/        pattern storage on merge
infra/
  terraform/      AWS infrastructure (EKS, RDS, Redis, ECR, S3)
  k8s/             Kubernetes manifests
monitoring/        Prometheus scrape config + Grafana dashboard
scripts/           weekly RAGAS evaluation job
```

### Local Development

Each service is a standalone FastAPI app:

```bash
cd services/<service-name>
pip install -r requirements.txt
uvicorn main:app --reload
```

## Observability

| Tool | Purpose |
|---|---|
| **Grafana** | Request rate, p99 latency, and error rate per service |
| **LangFuse** | Full trace of every PR review — prompts, responses, tokens, cost per agent |
| **GitHub Actions** | Per-service CI/CD (test → build → deploy), one pipeline per service |

See [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md#accessing-dashboards) for how to reach each dashboard.

## Cost

Running 24/7, the AWS infrastructure costs roughly **$180/month** (EKS control plane, worker nodes, RDS, Redis, load balancer, ECR/S3 storage), plus usage-based OpenAI costs (~$0.15 per 100 PRs reviewed with GPT-4o-mini). See [docs/DEPLOYMENT.md](docs/DEPLOYMENT.md#cost-estimate) for the full breakdown and teardown instructions.
