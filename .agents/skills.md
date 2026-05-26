# Required Antigravity Skills for Prism

This document lists the required Antigravity skills mapped to the architectural design, milestones, and security/cost guardrails of the `prism` LLM gateway.

---

## 1. Core Infrastructure & IaC

### [terraform-skill](file:///.gemini/config/skills/terraform-skill/SKILL.md) & [terraform-module-library](file:///.gemini/config/skills/terraform-module-library/SKILL.md)
* **Application**: Used to build and provision modular environments under `terraform/modules/` (VPC, EKS, DynamoDB, S3, IAM, Secrets Manager, CloudWatch, Budgets).
* **Alignment**: Ensures compliance with variable typing, validation blocks, outputs, remote backend configurations, KMS encryption, and DRY infrastructure code.
* **Milestones**: Milestone 1 (Issues 1-7) & Milestone 2 (Issues 8-10, 12-14, 24).

### [aws-skills](file:///.gemini/config/skills/aws-skills/SKILL.md)
* **Application**: Governs AWS cloud architecture patterns, cross-region service calls (gateway in `ap-south-1` to Bedrock in `us-east-1`), OIDC provider setups, and IRSA trust policy configurations.
* **Alignment**: Crucial for coordinating cluster networking, node security group rules, and private EKS topology.
* **Milestones**: Milestone 1, 2, & 4.

---

## 2. FinOps, Budgeting & Cost Discipline

### [cost-optimization](file:///.gemini/config/skills/cost-optimization/SKILL.md)
* **Application**: Enforces the $100 lifetime budget constraints defined in `agent.md`. Directs architectural decisions to keep NAT Gateways, ALBs, and VPC interface endpoints disabled in development.
* **Alignment**: Guides the deployment of spot instances (t3.small) and aggressive caching policies.
* **Milestones**: Cross-cutting (Milestone 1, 2, & 3).

### [aws-cost-optimizer](file:///.gemini/config/skills/aws-cost-optimizer/SKILL.md) & [aws-cost-cleanup](file:///.gemini/config/skills/aws-cost-cleanup/SKILL.md)
* **Application**: Powers the `make cost` target using the AWS CLI / Cost Explorer to analyze active infra spend vs. Bedrock token utilization. Drives `make down` and `make nuke` lifecycle patterns to prevent resource leaks.
* **Alignment**: Essential for cleaning up untagged or orphaned elastic IPs, NATs, and EBS volumes in dev.
* **Milestones**: Issues 1a, 7, 7a, & 29.

---

## 3. Application & API Development

### [pydantic-models-py](file:///.gemini/config/skills/pydantic-models-py/SKILL.md)
* **Application**: Governs FastAPI request/response contracts for key endpoints like `/v1/chat` and `/v1/embed`.
* **Alignment**: Enforces strict typing, field validations (e.g. valid models, temperature limits), and metadata payloads.
* **Milestones**: Milestone 2 (Issue 15, 16) & Milestone 3 (Issues 18, 20-22, 24).

### [jq](file:///.gemini/config/skills/jq/SKILL.md)
* **Application**: Parses structured JSON logs produced by python-json-logger and queries CloudWatch Logs Insights via the AWS CLI.
* **Alignment**: Leveraged in verification workflows and CI/CD pipelines to validate JSON metrics and API payloads.
* **Milestones**: Milestone 4 (Issue 28, 30).

---

## 4. LLMOps & Semantic Caching

### [embedding-strategies](file:///.gemini/config/skills/embedding-strategies/SKILL.md)
* **Application**: Informs the design of the semantic cache using Amazon Titan Text Embeddings V2 and DynamoDB.
* **Alignment**: Guides tokenization, vector similarity mathematics (cosine similarity threshold $\ge 0.95$), candidate pruning, and fixed-bin indexing design in DynamoDB.
* **Milestones**: Milestone 3 (Issue 20).

---

## 5. DevSecOps & Security Scaffolding

### [security-auditor](file:///.gemini/config/skills/security-auditor/SKILL.md)
* **Application**: Validates pod-level security configurations (`securityContext: runAsNonRoot, readOnlyRootFilesystem, drop capabilities`) and encryption requirements (KMS CMK for DynamoDB and S3).
* **Alignment**: Audit API key storage patterns, ensuring argon2 hashing takes place before keys touch DynamoDB.
* **Milestones**: Milestone 2 (Issues 12-14) & Milestone 3 (Issue 24).

### [secrets-management](file:///.gemini/config/skills/secrets-management/SKILL.md) & [varlock](file:///.gemini/config/skills/varlock/SKILL.md)
* **Application**: Directs retrieval of credentials from Secrets Manager and local environment setups.
* **Alignment**: Strict compliance with rule 8: "No secrets in code, env, ConfigMaps, or tfstate. Secrets Manager + KMS only."
* **Milestones**: Milestone 1 (Issue 1) & Milestone 2 (Issue 14).

---

## 6. Testing, Quality & Robustness

### [test-driven-development](file:///.gemini/config/skills/test-driven-development/SKILL.md) & [tdd-workflow](file:///.gemini/config/skills/tdd-workflow/SKILL.md) (incl. Red/Green/Refactor)
* **Application**: Enforces strict `pytest` implementation to verify auth, rate-limiting, budget enforcement, and streaming.
* **Alignment**: Ensures coverage metrics exceed the 70% threshold while using mocks and stubs to prevent live Bedrock token leaks.
* **Milestones**: Milestone 3 & 4.

### [systematic-debugging](file:///.gemini/config/skills/systematic-debugging/SKILL.md), [debugger](file:///.gemini/config/skills/debugger/SKILL.md) & [vibe-code-auditor](file:///.gemini/config/skills/vibe-code-auditor/SKILL.md)
* **Application**: Root-cause analysis tools for EKS scheduling failures, cross-region Bedrock routing latency, and failing prompt evals.
* **Alignment**: Eliminates fragile or "vibe-coded" blocks, auditing Python and Terraform scripts against static checkers (`ruff`, `mypy`, `tflint`, `tfsec`, `checkov`).
* **Milestones**: Continuous.

---

## 7. CI/CD & GitHub Actions Automation

### [github](file:///.gemini/config/skills/github/SKILL.md)
* **Application**: Integrates the `gh` CLI in development and CI/CD runs to debug GitHub Actions triggers, view job logs, and update PR annotations.
* **Milestones**: Milestone 4 (Issue 26, 30).

### [git-pr-review](file:///.gemini/config/skills/git-pr-review/SKILL.md) & [pr-writer](file:///.gemini/config/skills/pr-writer/SKILL.md)
* **Application**: Automated commit summarization and PR verification checks to adhere strictly to Trunk-Based squash merges and Conventional Commits.
* **Milestones**: Continuous.

### [incident-response-incident-response](file:///.gemini/config/skills/incident-response-incident-response/SKILL.md)
* **Application**: Authoring operational runbooks, disaster recovery strategies, and nightly teardown protocols.
* **Milestones**: Milestone 4.

---

## 8. Architecture, Documentation & Standards

### [architect-review](file:///.gemini/config/skills/architect-review/SKILL.md) & [c4-container](file:///.gemini/config/skills/c4-container/SKILL.md)
* **Application**: Maintains the systemic architectural diagrams (Mermaid flows), request flows, and deployment topology documentation.
* **Alignment**: Keeps the ten architectural decisions (ADRs) and `architecture.md` accurate.
* **Milestones**: Milestone 4 (Issue 32) & Continuous.

### [wiki-architect](file:///.gemini/config/skills/wiki-architect/SKILL.md)
* **Application**: Structures standard onboarding guides, project milestones, and developer runbooks.
* **Milestones**: Continuous.

### [avoid-ai-writing](file:///.gemini/config/skills/avoid-ai-writing/SKILL.md) & [unslop](file:///.gemini/config/skills/unslop/SKILL.md)
* **Application**: Ensures all project descriptions, runbooks, logs, and commit logs adhere to the blunt, terse, technical style requested by `agent.md`.
* **Milestones**: Continuous.
