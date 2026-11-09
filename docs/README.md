# Alex project guide

This is the entry point to the project documentation. It is written for someone who needs to understand the system deeply enough to explain, operate, and discuss it in an interview.

## Read in this order

1. [System architecture](architecture.md) — component maps and request/data flows.
2. [Components and data](components-and-data.md) — responsibilities, endpoints, database model, and repository map.
3. [Security, guardrails, and operations](security-operations.md) — authentication, IAM, resilience, observability, deployment, and known gaps.
4. [Interview guide](interview-guide.md) — a concise narrative, likely questions, and facts worth remembering.

## Project in one paragraph

Alex is a portfolio-management and research demo. A Next.js static frontend uses Clerk for sign-in and calls a FastAPI API running in Lambda. The API verifies Clerk JWTs, scopes user data by the verified Clerk subject, and stores data in Aurora PostgreSQL through the RDS Data API. Portfolio analysis is queued on SQS and handled by a planner Lambda that invokes specialist Lambdas backed by Amazon Bedrock. A separate research service browses public web pages, generates a short research note, and ingests it into an S3 Vectors knowledge base using a SageMaker embedding endpoint.

## Deployed environment snapshot

The AWS deployment was verified on 2026-11-08 in account `635863384208`, region `eu-west-2`.

- App: <https://dnq9g3q5oxi1c.cloudfront.net>
- API Gateway: <https://gerftjz2zl.execute-api.eu-west-2.amazonaws.com>
- Research service health: <https://s4klgb3unadaqfy4bobah2adi40uzppi.lambda-url.eu-west-2.on.aws/health>
- CloudWatch dashboards: `alex-ai-model-usage`, `alex-agent-performance`
- Aurora cluster: `alex-aurora-cluster` (Aurora PostgreSQL, Data API enabled)
- SageMaker endpoint: `alex-embedding-endpoint` (`InService` at the last verification)
- SQS queue: `alex-analysis-jobs`; dead-letter queue: `alex-analysis-jobs-dlq`

This is a dated snapshot, not a live status page. Check AWS before relying on the deployment details; individual resources may have changed since it was verified.

## Documentation scope and source of truth

The docs describe the checked-in application and Terraform, and distinguish those from the verified deployment choices. Some older README text is stale: `terraform/README.md` still refers to an App Runner researcher and an `8_observability` stack, while the current Terraform deploys the researcher as a Lambda container and calls stack 8 `8_enterprise`. The current stack files and deployed state are the source of truth.

Sensitive values, Terraform state, and local environment files are intentionally not copied into these docs.

## Coverage review

The documentation was checked against the requested interview-prep scope: whole-system architecture, component-to-component connections and protocols, Clerk and ingestion authentication, data model and API, AWS service responsibilities, security controls and known gaps, observability, deployment operations, costs, and likely interview questions. All three Mermaid diagrams were rendered and visually inspected. They use top-to-bottom layouts and separate the authenticated API, async agent workflow, and research/retrieval path to keep labels legible and lines manageable.
