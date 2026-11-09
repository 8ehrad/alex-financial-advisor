# Interview guide

## A 60-second explanation

“Alex is a serverless portfolio-analysis demo built around a Next.js frontend, Clerk authentication, FastAPI on Lambda, Aurora PostgreSQL, and an asynchronous agent workflow. The browser sends a Clerk JWT to the API through CloudFront and API Gateway. FastAPI verifies it against Clerk JWKS, derives the user's subject, checks ownership, and persists work through the Aurora Data API. Analysis requests are stored as jobs and queued in SQS. A planner Lambda loads portfolio context and coordinates tagging, report-writing, charting, and retirement Lambdas, which write separate result payloads back to the job row. A separate researcher Lambda browses public finance pages and stores summaries through an API-key-protected ingestion Lambda, SageMaker embeddings, and S3 Vectors. CloudWatch dashboards cover model usage and agent performance. The important caveats are that research URL auth is public, price lookup falls back to random numbers without a Polygon key, and the project is a demo rather than financial advice.”

## Architecture points to emphasize

- **Async boundary:** the API does not wait for a complete analysis. It creates a durable job row, enqueues SQS work, returns a job ID, and the UI polls.
- **Orchestration:** Planner is a Lambda-based orchestrator; specialists are separate deployable Lambdas. The LLM can call specialist tools; the code does not promise parallel execution.
- **Identity and ownership:** Clerk supplies identity, but the backend validates tokens and enforces resource ownership. API Gateway is only routing/throttling, not the JWT authorizer.
- **Persistence:** relational user/portfolio/job state is in Aurora; agent outputs are JSONB payload fields on a job. Searchable research notes are in S3 Vectors, not the relational store.
- **Managed integration:** Lambda accesses Aurora through Data API and IAM/Secrets Manager, avoiding connection pooling and direct DB sockets in app code.
- **Independent infrastructure stacks:** Terraform is segmented by layer, which helps progressive deployment but local state and manual sequencing reduce portability.

## Questions you may get

### Why SQS and Lambda for analysis?

Agent analysis is slower and more failure-prone than a normal HTTP request. SQS gives the API a quick acknowledgement path, durable buffering, retry behavior, and a DLQ. Lambda event-source mapping runs Planner, while the job record provides status and results for polling.

### How do you prevent one user from seeing another user's portfolio?

FastAPI validates the Clerk session JWT against the Clerk JWKS endpoint and reads `sub` from the decoded/validated token. List queries are scoped to that subject. For a detail/update/delete, handlers load the owning account or job and compare its stored Clerk subject before returning or mutating it. UI-only signed-in checks are not the security control.

### Why use both Aurora and S3 Vectors?

Aurora is suited to structured, relational account, position, instrument, and job state with owner constraints. S3 Vectors holds embedded research text for semantic retrieval. SageMaker converts query/document text into vectors; Reporter uses vector search to add relevant background to its prompt.

### What happens if an agent fails?

The planner records failed job state and error text. Model throttling retries are implemented in several agent functions. The queue is configured for three receives before a DLQ move, but the Planner handler catches exceptions and returns a normal Lambda response; for SQS that can count as a successful invocation, so the message may not be retried. I would change the handler to re-raise failures or return explicit partial-batch failures, then validate retry/DLQ behavior. Also, Planner currently marks a job complete when the model run returns without verifying that every specialist output was saved.

### What is the role of Bedrock and LiteLLM?

Bedrock hosts the language model; LiteLLM gives agent code a provider adapter. The deployed portfolio agents used `amazon.nova-pro-v1:0` in `eu-west-2` at the last verified deployment. The researcher is independently configured and can use a different Bedrock model/region.

### How is retrieval implemented?

The researcher agent's ingestion tool calls API Gateway with an API key. The ingestion Lambda embeds the research text with SageMaker, then stores the vector and metadata in the `financial-research` index. Reporter embeds a query and requests up to three nearest vectors. The vector index was reused from existing account resources rather than created by the current Terraform stack.

### What are the key reliability/cost trade-offs?

The serverless stack scales operationally and keeps agent work separate from web requests, but Bedrock and SageMaker usage is metered, Lambda has max-duration limits, and the user-visible job flow depends on polling. Aurora has a nontrivial standing cost. The current stack is demo-oriented: local Terraform state, one-day backup, no final snapshot, and no paging alarms.

### What would you improve before production?

Prioritize: (1) add auth, quotas, and abuse protection to the public researcher URL; (2) remove random-price fallback and require timestamped, licensed market data; (3) make SQS failure signaling explicit and validate all specialist outputs before marking a job complete; (4) improve IAM least privilege and secret rotation; (5) use remote Terraform state and CI/CD; (6) add alarms, tracing/correlation IDs, idempotency and a documented DLQ replay process; (7) set durable backup/final-snapshot policies; (8) validate/model-gate financial outputs and clearly define the product's advice/compliance scope.

## Facts worth remembering

- Frontend: Next.js static export on S3 + CloudFront.
- API: FastAPI + Mangum on Lambda; API Gateway HTTP API routes `/api/*`.
- Auth: Clerk JWT validated in FastAPI using JWKS; user key is JWT `sub`.
- Relational data: Aurora PostgreSQL via RDS Data API.
- Async queue: `alex-analysis-jobs`; DLQ max receive count 3; one-day retention.
- Agents: Planner, Tagger, Reporter, Charter, Retirement.
- LLMs: Amazon Bedrock; OpenAI Agents SDK/LiteLLM orchestration integrations.
- Retrieval: SageMaker embeddings + S3 Vectors index `financial-research`.
- Researcher: separate Playwright/MCP Lambda Function URL; public authorization mode in current Terraform.
- Observability: CloudWatch logs/metrics and two dashboards; Langfuse is optional.
- Known demo caveat: when Polygon key is absent or the call fails, price helper returns random values from 1–100.
- Retirement assumptions: 500 Monte Carlo runs with fixed asset-class returns/volatilities, hard-coded contributions, and withdrawal assumptions; this is educational output, not calibrated financial planning.

## Questions to ask the interviewer

- What reliability target and job completion latency would you set for long-running analysis?
- Would you use Step Functions instead of an LLM-controlled Lambda orchestrator for more explicit retries and workflow state?
- How should market data provenance, timestamping, licensing, and stale-price behavior be handled?
- What authorization, audit trail, and human-review requirements apply to financial outputs?
- Would you keep vectors and relational data in separate stores, and how would you measure retrieval quality?
