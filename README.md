# Alex Financial Advisor

Alex, short for Agentic Learning Equities eXplainer, is a production-style AI financial planning application. It combines a Next.js frontend, a FastAPI backend, AWS serverless infrastructure, and a multi-agent analysis system that can inspect a user's portfolio, generate written analysis, produce chart-ready data, and run retirement projections.

This repository contains the project deliverables only. Course walkthrough material and deployment guides have been intentionally excluded.

![Alex application image](assets/alex.png)

## What It Does

- Manages user investment accounts, positions, cash balances, and portfolio preferences.
- Authenticates users with Clerk and keeps portfolio data separated by user.
- Runs asynchronous portfolio analysis through an SQS-triggered planner workflow.
- Coordinates specialist AI agents for instrument tagging, portfolio reporting, chart generation, and retirement planning.
- Stores structured user, account, position, instrument, and job data in Aurora PostgreSQL through the Data API.
- Supports semantic market research retrieval through SageMaker embeddings and S3 Vectors.
- Deploys as a serverless AWS application with Terraform-managed infrastructure.

## Architecture

```text
User Browser
  -> CloudFront + S3 static Next.js frontend
  -> API Gateway
  -> FastAPI Lambda backend
  -> Aurora Serverless v2 PostgreSQL
  -> SQS analysis queue
  -> Planner Lambda
       -> Tagger Lambda
       -> Reporter Lambda
       -> Charter Lambda
       -> Retirement Lambda
  -> Results persisted back to Aurora
```

The research pipeline uses a separate ingestion path:

```text
Researcher service
  -> Bedrock model through LiteLLM
  -> Ingest API Lambda
  -> SageMaker embedding endpoint
  -> S3 Vectors index
```

## Tech Stack

- **Frontend:** Next.js, React, TypeScript, Tailwind CSS, Clerk
- **Backend API:** FastAPI, AWS Lambda, Pydantic
- **AI agents:** OpenAI Agents SDK, LiteLLM, AWS Bedrock
- **Data:** Aurora Serverless v2 PostgreSQL, RDS Data API, S3 Vectors
- **ML infrastructure:** SageMaker serverless endpoint for embeddings
- **Cloud:** API Gateway, Lambda, SQS, S3, CloudFront, ECR, CloudWatch
- **Infrastructure:** Terraform
- **Python tooling:** uv

## Repository Structure

```text
assets/       Shared images and project assets
backend/      Agent code, API Lambda, database library, ingestion, packaging, and tests
frontend/     Next.js application
scripts/      Local development, deployment, and cleanup helpers
terraform/    Independent Terraform stacks for each infrastructure layer
```

Each Python service directory is a uv project. The Terraform directories are intentionally independent so individual layers can be deployed and destroyed separately.

## Local Development

Prerequisites:

- Python with `uv`
- Node.js 20+
- AWS CLI configured
- Terraform
- Docker Desktop for Lambda packaging
- Clerk application credentials

Create local environment files from examples:

```bash
cp .env.example .env
```

Create `frontend/.env.local` with your Clerk keys and local API URL for frontend development.

Install frontend dependencies:

```bash
cd frontend
npm install
```

Run the local frontend and API together:

```bash
cd scripts
uv run run_local.py
```

The local frontend runs on `http://localhost:3000` and the local API runs on `http://localhost:8000`.

## Testing

Run local agent tests with mocked Lambda calls:

```bash
cd backend
uv run test_simple.py
```

After AWS deployment, run the full integration test:

```bash
cd backend
uv run test_full.py
```

Individual agent directories also include their own `test_simple.py` and `test_full.py` scripts.

## Deployment Overview

Infrastructure is split into independent Terraform stacks:

- `terraform/2_sagemaker` deploys the embedding endpoint.
- `terraform/3_ingestion` deploys ingestion infrastructure for S3 Vectors.
- `terraform/4_researcher` deploys the research service infrastructure.
- `terraform/5_database` deploys Aurora PostgreSQL and database credentials.
- `terraform/6_agents` deploys the multi-agent Lambda system and SQS queue.
- `terraform/7_frontend` deploys the frontend hosting and API Gateway backend.
- `terraform/8_enterprise` adds monitoring and enterprise operational resources.

Before applying any stack, copy that stack's `terraform.tfvars.example` to `terraform.tfvars` and fill in account-specific values. `terraform.tfvars`, `.env`, Terraform state, virtual environments, build output, and secrets are ignored by Git.

## Security Notes

- Clerk handles authentication; API requests validate Clerk JWTs.
- AWS Secrets Manager stores database credentials.
- IAM roles are scoped by service responsibilities.
- API Gateway, CloudFront, CORS, and JWT validation protect the public API surface.
- Local environment files and Terraform state are intentionally excluded from version control.

## Cost Notes

This project deploys real AWS infrastructure. Aurora Serverless v2 is usually the largest standing cost while active. Destroy unused Terraform stacks when the application is not being developed or demonstrated.

## Attribution

This project was built as part of the AI in Production course capstone and adapted here as a deliverables-only repository.
