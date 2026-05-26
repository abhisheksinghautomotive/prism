# Milestones and Issues

Four milestones, eight weeks. Each issue lists **Required** (the deliverable and why) and **How** (the concrete steps). Work in order — later issues depend on earlier ones.

---

## Milestone Overview

| # | Milestone | Due | Issues |
|---|---|---|---|
| 1 | Foundation Infrastructure | Week 2 | 1, 1a, 2, 3, 4, 5, 6, 7, 7a |
| 2 | Gateway Core | Week 4 | 8, 9, 10, 11, 12, 13, 14, 15, 16 |
| 3 | LLMOps Features | Week 6 | 17, 18, 19, 20, 21, 22, 23, 24, 25 |
| 4 | Observability and Eval Gate | Week 8 | 26, 27, 28, 29, 30, 31, 32 |

**Total:** 34 issues across 8 weeks.

---

## Milestone 1 — Foundation Infrastructure

**Due:** Week 2
**Theme:** Stand up a safe, cheap AWS playground with cost guardrails, a reusable VPC module, and Bedrock model access enabled. By the end, `make up` and `make down` cycle cleanly with zero leakage.

---

### Issue 1: Create S3 bucket and DynamoDB table for Terraform remote state

**Required:** A versioned S3 bucket for tfstate and a DynamoDB table for state locking. Prevents state corruption and gives audit history.

**How:**
- Create `terraform/bootstrap/` with local state.
- S3 bucket: versioning on, SSE-S3, public access blocked.
- DynamoDB table `prism-tflock`, hash key `LockID` (string), `PAY_PER_REQUEST`.
- Configure `terraform/environments/dev/backend.tf` to use them.
- `terraform init -migrate-state` to push local state to S3.

---

### Issue 1a: Create AWS Budget with $10 / $25 / $50 / $75 SNS alerts

**Required:** Budget alarms before any chargeable resource is provisioned. Must catch Bedrock token overspend in addition to infra cost.

**How:**
- SNS topic `prism-budget-alerts`, subscribe your email.
- `aws_budgets_budget` of type `COST`, monthly limit `$100`.
- Four `notification` blocks at 10%, 25%, 50%, 75% of actual spend.
- Confirm SNS email subscription.
- Add a second budget filtered by service `Amazon Bedrock` to specifically track token spend.

---

### Issue 2: Provision VPC with configurable AZs and subnet topology

**Required:** Reusable VPC module supporting 1-3 AZs and optional private subnets. Dev uses 1 AZ public; prod uses 2 AZ with private subnets.

**How:**
- `terraform/modules/vpc/` with `main.tf`, `variables.tf`, `outputs.tf`.
- Inputs: `cidr_block` (default `10.0.0.0/16`), `availability_zones`, `enable_private_subnets`, `enable_nat`.
- Use `for_each` on subnet resources.
- Outputs: `vpc_id`, `public_subnet_ids`, `private_subnet_ids`.

---

### Issue 3: Configure Internet Gateway and conditional NAT Gateway

**Required:** IGW always on (free). NAT Gateway gated by `enable_nat`. Dev keeps NAT off.

**How:**
- Always create `aws_internet_gateway`.
- `aws_eip` + `aws_nat_gateway` with `count = var.enable_nat ? 1 : 0`.
- When enabled, place NAT in the first public subnet.

---

### Issue 4: Create route tables for public and private subnets

**Required:** Public RT routing `0.0.0.0/0` to IGW. Private RT routing to NAT when enabled; no default route otherwise.

**How:**
- `aws_route_table` for public; `aws_route` to IGW.
- `aws_route_table_association` looped over all public subnet IDs.
- Private RT created only when private subnets exist.
- NAT default route conditional via `count`.

---

### Issue 5: Define security groups for EKS nodes and ALB

**Required:** Node SG for inter-node and pod traffic. ALB SG for inbound 443. ALB SG built in dev but unattached.

**How:**
- `terraform/modules/vpc/security_groups.tf`.
- Node SG: ingress self all ports, ingress from ALB SG on 1025-65535, egress all.
- ALB SG: ingress 443 and 80 from `0.0.0.0/0`, egress to node SG only.
- Output both SG IDs.

---

### Issue 6: Package VPC as a clean Terraform module

**Required:** VPC code from issues 2-5 as a self-contained, consumable module.

**How:**
- All VPC resources under `terraform/modules/vpc/`.
- Inputs in `variables.tf` with description, type, defaults, validation.
- Outputs: `vpc_id`, `public_subnet_ids`, `private_subnet_ids`, `node_security_group_id`, `alb_security_group_id`.
- Consume from `terraform/environments/dev/main.tf` via `source = "../../modules/vpc"`.

---

### Issue 7: Validate full destroy and re-apply cycle works cleanly

**Required:** Proof `terraform destroy` followed by `terraform apply` produces identical infrastructure with zero orphans. Foundation of cost discipline.

**How:**
- `terraform apply` from clean state.
- Note resource IDs in Console.
- `terraform destroy`; verify no orphan ENIs, EIPs, subnets.
- `terraform apply` again; confirm `terraform plan` shows "No changes."
- Document manual cleanup steps in `docs/runbooks/teardown-checklist.md`.

---

### Issue 7a: Write Makefile with up / down / nuke / cost targets

**Required:** A `Makefile` at repo root abstracting the cost lifecycle. `make down` becomes session-end reflex.

**How:**
- Targets:
  - `bootstrap`: `terraform init && apply` in `terraform/bootstrap/`.
  - `up`: `terraform apply` in dev env then `helm install`.
  - `down`: `helm uninstall` then `terraform destroy`.
  - `nuke`: destroys bootstrap with confirmation prompt.
  - `cost`: `aws ce get-cost-and-usage`, filtered separately for Bedrock vs infra.
- `.PHONY` declarations, `help` target.

---

## Milestone 2 — Gateway Core

**Due:** Week 4
**Theme:** Stand up EKS + ECR + DynamoDB + S3 audit bucket. Build the FastAPI gateway with Bedrock integration, API key auth, request logging, and streaming. End state: send a curl request with an API key, get a streamed Claude response, see the row in DynamoDB and the object in S3.

---

### Issue 8: Write Terraform module for EKS cluster

**Required:** Reusable EKS module supporting public or private node placement via variable. Dev uses public.

**How:**
- `terraform/modules/eks/`.
- Inputs: `cluster_name`, `cluster_version` (default `1.30`), `subnet_ids`, `node_subnet_placement`, `node_security_group_id`.
- `aws_eks_cluster` + service role.
- `aws_iam_openid_connect_provider` from cluster OIDC issuer URL.
- Outputs: `cluster_endpoint`, `cluster_name`, `oidc_provider_arn`, `oidc_provider_url`.

---

### Issue 9: Configure managed node group with t3.small SPOT

**Required:** Managed node group, dev runs 1x t3.small SPOT. Prod variable: 2x t3.medium on-demand.

**How:**
- `aws_eks_node_group` inside the EKS module.
- Inputs: `instance_types = ["t3.small"]`, `capacity_type = "SPOT"`, `desired_size = 1`, `min_size = 1`, `max_size = 2`.
- IAM policies: `AmazonEKSWorkerNodePolicy`, `AmazonEKS_CNI_Policy`, `AmazonEC2ContainerRegistryReadOnly`.
- Verify: `aws eks update-kubeconfig --region ap-south-1 --name prism-dev` then `kubectl get nodes`.

---

### Issue 10: Create ECR repository with scan-on-push and image lifecycle

**Required:** Private ECR with scan-on-push, immutable tags, keep-last-5 lifecycle.

**How:**
- `terraform/modules/ecr/`.
- `aws_ecr_repository` with `scan_on_push = true`, `image_tag_mutability = "IMMUTABLE"`.
- `aws_ecr_lifecycle_policy` keep last 5 images.
- Output `repository_url`.

---

### Issue 11: Enable Bedrock model access for Claude 3 Haiku and Titan Embed

**Required:** Bedrock model access granted in the AWS account for the two models the gateway will use. Without this, API calls fail with `AccessDeniedException`.

**How:**
- In AWS Console, Bedrock, Model Access, request access to:
  - Anthropic Claude 3 Haiku
  - Amazon Titan Text Embeddings V2
- Document the chosen region in `docs/runbooks/bedrock-region.md` (likely `us-east-1` for cross-region; gateway in `ap-south-1`).
- Add the region to `terraform/environments/dev/terraform.tfvars` as `bedrock_region`.

---

### Issue 12: Create DynamoDB tables for tenants, cache, and request log

**Required:** Three DynamoDB tables. `tenants` (API key hash → tenant + budget + rate limit). `cache` (prompt embedding hash → completion). `requests` (per-call audit row, short TTL since full audit lives in S3).

**How:**
- `terraform/modules/dynamodb/`.
- `tenants`: hash key `api_key_hash` (string), `PAY_PER_REQUEST`.
- `cache`: hash key `embedding_hash` (string), TTL attribute `expires_at` (24h).
- `requests`: hash key `request_id` (string), sort key `tenant_id`, TTL 7 days.
- All tables encrypted with customer-managed KMS key.

---

### Issue 13: Create S3 audit bucket for request/response logs

**Required:** Versioned, KMS-encrypted S3 bucket. One object per request, partitioned by `year=/month=/day=/hour=/` for Athena querying.

**How:**
- `terraform/modules/s3/audit.tf`.
- Bucket with versioning, SSE-KMS, block public access.
- Lifecycle: transition to Glacier Instant Retrieval after 30 days, delete after 365.
- Output bucket name.

---

### Issue 14: Configure IRSA for the gateway pod

**Required:** Kubernetes ServiceAccount annotated with an IAM role granting Bedrock invoke, DynamoDB read/write, S3 put, Secrets Manager get.

**How:**
- `terraform/modules/iam/irsa.tf`.
- IAM role `prism-gateway-irsa`, trust policy on OIDC provider, `sub = system:serviceaccount:prism:prism-gateway`.
- Policy: `bedrock:InvokeModel` and `bedrock:InvokeModelWithResponseStream` on specific model ARNs; `dynamodb:GetItem/PutItem/UpdateItem` on the three table ARNs; `s3:PutObject` on the audit bucket; `secretsmanager:GetSecretValue` on signing key secret.
- Annotate Helm `serviceaccount.yaml` with role ARN.

---

### Issue 15: Build FastAPI gateway with /v1/chat, /v1/embed, /health endpoints

**Required:** Minimal FastAPI service.
- `POST /v1/chat`: accepts `{prompt, model, stream}`, returns Bedrock completion.
- `POST /v1/embed`: returns Titan embedding vector.
- `GET /health`: 200 OK.
- Auth: `X-API-Key` header validated against DynamoDB.

**How:**
- `app/main.py` registers routers from `app/routers/`.
- `app/core/bedrock.py` wraps `boto3.client("bedrock-runtime")` with `bedrock_region` from env.
- `app/core/auth.py` hashes incoming key with argon2, looks up `tenants` table.
- `app/core/audit.py` puts request/response JSON to S3 with partition prefix.
- `app/requirements.txt`: `fastapi`, `uvicorn`, `boto3`, `argon2-cffi`, `python-json-logger`.

---

### Issue 16: Implement streaming responses via Server-Sent Events

**Required:** `/v1/chat` with `stream=true` returns SSE chunks as Bedrock streams tokens. Cancellable mid-response.

**How:**
- Use `bedrock-runtime.invoke_model_with_response_stream`.
- FastAPI `StreamingResponse` with `media_type="text/event-stream"`.
- Each Bedrock chunk emitted as `data: {json}\n\n`.
- On client disconnect, cancel the Bedrock iterator.
- Test with `curl -N` and `httpx` async client.

---

### Issue 17: Write multi-stage Dockerfile, Helm chart, deploy to EKS

**Required:** Containerize the gateway, deploy via Helm, verify via `kubectl port-forward`.

**How:**
- Multi-stage Dockerfile, `python:3.12-slim`, non-root UID 10001, target < 200 MB.
- `helm/prism-gateway/` with `Chart.yaml`, `values-dev.yaml`, `values-prod.yaml`, `templates/`.
- Templates: `deployment.yaml`, `service.yaml`, `serviceaccount.yaml` (annotated with IRSA ARN), `ingress.yaml` (conditional), `hpa.yaml` (conditional).
- `helm install prism ./helm/prism-gateway -n prism --create-namespace -f values-dev.yaml`.
- `kubectl port-forward -n prism svc/prism-gateway 8080:80`.
- `curl -H "X-API-Key: ..." -d '{"prompt":"hello","model":"haiku"}' localhost:8080/v1/chat` returns Claude response.

---

## Milestone 3 — LLMOps Features

**Due:** Week 6
**Theme:** Add the operator layer that turns a "FastAPI in front of Bedrock" tutorial into an LLMOps platform. Per-tenant budgets, semantic cache, A/B routing, eval harness.

---

### Issue 18: Implement per-tenant token budgets in DynamoDB

**Required:** Each tenant has `monthly_token_limit` and `tokens_used_current_month`. Requests over the limit return 429.

**How:**
- On every `/v1/chat`, after Bedrock returns usage:
  - `dynamodb:UpdateItem` with `ADD tokens_used_current_month :n` where `n = input_tokens + output_tokens`.
  - Read updated counter; if > limit, raise 429 on next request.
- Add a scheduled Lambda (or EKS CronJob) that resets counters at month boundary.
- Admin endpoint `POST /v1/tenants/{id}/reset-budget` for testing.

---

### Issue 19: Implement rate limiting per tenant

**Required:** Burst + sustained limits. Burst: 10 req/sec. Sustained: 1k req/min. Both per-tenant, in DynamoDB.

**How:**
- Token-bucket algorithm in `app/core/ratelimit.py`.
- DynamoDB atomic counter with TTL on the bucket row.
- Return 429 with `Retry-After` header when exceeded.
- Configurable per tenant in the `tenants` table.

---

### Issue 20: Implement semantic cache via Titan Embed + DynamoDB

**Required:** Cache key = hash of normalized prompt embedding. On lookup, compute embedding, query DynamoDB for nearest match within cosine similarity threshold 0.95. Cache hits skip Bedrock entirely.

**How:**
- On request: embed prompt with Titan; hash embedding to fixed buckets (LSH or simple binning).
- Query `cache` table for bucket; iterate small candidate set; compute exact cosine.
- If hit (similarity >= 0.95): return cached completion, log cache hit, charge 0 Bedrock tokens.
- On miss: invoke Bedrock; store `{embedding, completion, model, ttl}` in cache.
- Add response header `X-Prism-Cache-Hit: true|false`.
- Note: for portfolio scale this DDB-based cache is fine. ADR-005 documents that pgvector/OpenSearch would be the prod path.

---

### Issue 21: Add cost metadata to responses

**Required:** Every `/v1/chat` response includes headers: `X-Prism-Tokens-In`, `X-Prism-Tokens-Out`, `X-Prism-Cost-Cents`, `X-Prism-Cache-Hit`, `X-Prism-Model`, `X-Prism-Latency-Ms`.

**How:**
- Maintain a model pricing table in `app/core/pricing.py` (Claude Haiku, Titan Embed rates).
- After Bedrock call, compute `cost_cents = (tokens_in * rate_in + tokens_out * rate_out) * 100`.
- Set headers before returning response.
- For streaming, send final cost as a trailing SSE event `event: cost\ndata: {...}\n\n`.

---

### Issue 22: Implement model A/B routing

**Required:** Tenants can opt into A/B mode. Request goes to two models in parallel; both responses returned; logged with `experiment_id`. Used to compare quality offline.

**How:**
- Tenant attribute `ab_routing_enabled` and `ab_models: ["claude-haiku", "llama-3-8b"]`.
- When enabled, `/v1/chat` returns `{responses: [{model, completion, cost}, ...], experiment_id}`.
- Both calls audited to S3 with same `experiment_id` for offline grouping.
- Stream mode disabled when A/B mode is on (simpler).

---

### Issue 23: Build the eval harness

**Required:** A `evals/runner.py` script that takes a prompt set + expected behaviors and runs them against the live gateway (or a target URL). Outputs a pass/fail report.

**How:**
- `evals/datasets/basic.jsonl` — list of `{prompt, must_contain: [...], must_not_contain: [...], max_tokens: N, max_latency_ms: N}`.
- `evals/runner.py`:
  - Loads dataset.
  - For each prompt, calls `POST /v1/chat`.
  - Asserts on completion content, token count, latency.
  - Outputs JSON summary: `{total, passed, failed, regressions: [...]}`.
- Exit code 0 on all pass, 1 on any fail.

---

### Issue 24: Admin endpoints for tenant management

**Required:** Bootstrap path to create tenants without a UI. Admin-key-protected endpoints.

**How:**
- `POST /v1/admin/tenants` → creates tenant, returns plaintext API key (only time it's visible).
- `GET /v1/admin/tenants/{id}` → returns tenant info (no key).
- `PATCH /v1/admin/tenants/{id}` → update budget, rate limits.
- Admin key separate from tenant API keys; stored in Secrets Manager; required via `X-Admin-Key` header.

---

### Issue 25: Example client scripts in examples/

**Required:** Two example clients in `examples/` so a recruiter can curl your gateway in 30 seconds.

**How:**
- `examples/curl/chat.sh`, `examples/curl/stream.sh`, `examples/curl/embed.sh`.
- `examples/python-client/client.py` — minimal class with `chat()`, `chat_stream()`, `embed()` methods.
- README in `examples/` showing how to set `PRISM_API_KEY` and `PRISM_URL` env vars.

---

## Milestone 4 — Observability and Eval Gate

**Due:** Week 8
**Theme:** Wire observability and the prompt eval PR gate. End state: dashboard shows tokens/sec, $/req, cache hit rate. PR that changes a prompt or eval set triggers the gate; regression blocks merge.

---

### Issue 26: Configure GitHub OIDC provider and IAM role for Actions

**Required:** OIDC provider for `token.actions.githubusercontent.com`, IAM role `gha-prism-deploy` with trust scoped to the repo. Lives in `terraform/bootstrap/` so it survives `make down`.

**How:**
- `aws_iam_openid_connect_provider` in bootstrap.
- `aws_iam_role` with trust policy: `aud = sts.amazonaws.com`, `sub LIKE repo:abhishek-singh/prism:*`.
- Permissions: ECR push, EKS describe, Helm via `aws-auth` ConfigMap, S3 read on audit bucket, Bedrock invoke (for eval gate).
- Output role ARN as repo variable `AWS_ROLE_ARN`.

---

### Issue 27: Enable CloudWatch Container Insights with 1-day retention

**Required:** Container Insights on EKS for pod metrics. Log retention 1 day in dev, 7 days in prod.

**How:**
- Install `amazon-cloudwatch-observability` EKS Add-On via `aws_eks_addon`.
- `aws_cloudwatch_log_group` for `/aws/containerinsights/prism-dev/*`, `retention_in_days = 1`.
- Verify pod CPU/memory visible in Container Insights console.

---

### Issue 28: Implement structured JSON logging in FastAPI

**Required:** All logs single-line JSON. Includes `request_id`, `tenant_id`, `model`, `tokens_in`, `tokens_out`, `cost_cents`, `cache_hit`, `latency_ms`.

**How:**
- Configure Python `logging` with `python-json-logger`.
- FastAPI middleware injects `request_id` and `tenant_id` via `contextvars`.
- `logger.info("request_completed", extra={...})` at end of every handler.
- Verify in CloudWatch Logs Insights: `fields @timestamp, model, tokens_out, cost_cents, cache_hit | sort @timestamp desc | limit 50`.

---

### Issue 29: Create CloudWatch dashboard with LLMOps signals

**Required:** Terraform-managed dashboard with 8 widgets covering LLMOps-specific signals, not just generic golden signals.

**How:**
- `terraform/modules/observability/dashboard.tf`.
- `aws_cloudwatch_dashboard` with widgets:
  1. Requests/min (total)
  2. p50/p95/p99 latency
  3. Tokens/sec (input + output)
  4. Bedrock cost per hour (computed from logs via metric filter)
  5. Cache hit rate (%)
  6. Per-tenant request volume (top 5)
  7. 4xx + 5xx error rate
  8. Pod CPU + memory
- Verify dashboard renders.

---

### Issue 30: Write prompt-eval-gate.yml workflow

**Required:** A GitHub Actions workflow triggered on PRs touching `prompts/`, `evals/`, or `app/core/bedrock.py`. Spins up the gateway in dev, runs the eval harness, comments results on the PR. Regression (any eval fail) blocks merge.

**How:**
- `.github/workflows/prompt-eval-gate.yml`.
- Trigger: `on: pull_request: paths: ["prompts/**", "evals/**", "app/core/bedrock.py"]`.
- `permissions: id-token: write, contents: read, pull-requests: write`.
- Steps:
  - Assume OIDC role.
  - Bring up gateway: `make up`.
  - Wait for `/health` 200.
  - Run `python evals/runner.py --target http://localhost:8080 --dataset evals/datasets/basic.jsonl --output eval-results.json`.
  - Post results to PR as sticky comment.
  - `make down` in `if: always()` step.
  - Set status check based on exit code.
- Branch protection on `main` requires this check.

---

### Issue 31: Add Athena setup for querying S3 audit logs

**Required:** Athena workgroup + table over the S3 audit bucket so you can query token spend per tenant per day via SQL.

**How:**
- `terraform/modules/observability/athena.tf`.
- `aws_athena_workgroup` with query results bucket.
- `aws_glue_catalog_table` with partitions on year/month/day/hour.
- Example query in `docs/runbooks/cost-analysis.md`:
  ```sql
  SELECT tenant_id, SUM(cost_cents)/100.0 AS cost_dollars
  FROM prism_audit WHERE year='2026' AND month='05'
  GROUP BY tenant_id ORDER BY cost_dollars DESC;
  ```
- Verify query runs and returns data.

---

### Issue 32: README with architecture diagram, examples, final cost report

**Required:** Repo README that lets a recruiter understand prism in < 2 min, run an example in < 15 min, see the cost story with screenshots.

**How:**
- Elevator pitch (one paragraph).
- Architecture diagram (Mermaid from architecture.md).
- Tech stack table.
- Quick start: `make bootstrap`, `make up`, `examples/curl/chat.sh`, `make down`.
- LLMOps feature checklist (tenant budgets, semantic cache, A/B, eval gate, cost headers).
- Cost story: Cost Explorer screenshot + Athena query showing token spend.
- Security highlights.
- Link to `architecture.md` and `milestones.md`.
- Pin repo on GitHub profile.
- Add to profile README: "Currently building [prism](https://github.com/abhishek-singh/prism) — LLM gateway on AWS Bedrock with per-tenant budgets, semantic cache, and prompt eval PR gate."

---

## What This Project Proves on a Resume

| Skill | Where |
|---|---|
| AWS (EKS, ECR, Bedrock, DynamoDB, S3, KMS, IAM, CloudWatch, Athena, Budgets) | Every module |
| LLMOps patterns (tenant budgets, semantic cache, A/B, cost tracking, eval gates) | M3 + M4 |
| Kubernetes (EKS, IRSA, Helm, HPA, securityContext) | M2 + M4 |
| Terraform (modules, remote state, multi-env) | All milestones |
| CI/CD (GitHub Actions, OIDC, reusable workflows, eval gate) | M4 |
| Docker (multi-stage, non-root, slim) | M2 |
| API design (streaming, rate limiting, structured errors) | M2 + M3 |
| Security (OIDC, IRSA, KMS, hashed API keys, audit log) | Cross-cutting |
| Observability (JSON logs, dashboards, Athena queries) | M4 |
| Cost engineering (token-level tracking, semantic caching, ephemeral lifecycle) | M3 + this doc |
| FinOps mindset (per-tenant budgets, real $ in response headers) | M3 |

Pair with Verdict, you have a portfolio that covers DevOps + LLMOps without overlap.
