# Components and data model

## Frontend

The app is Next.js 15 with the Pages Router, React, TypeScript, and Tailwind. Production is a static export hosted from an S3 website origin behind CloudFront. The same CloudFront hostname serves `/api/*` through API Gateway, so the deployed app can use relative API paths. Local development uses `http://localhost:8000` for the API and `http://localhost:3000` for the site.

Main pages:

| Route | Purpose |
|---|---|
| `/` | Public landing page and Clerk sign-in/sign-up controls |
| `/dashboard` | User profile, accounts, positions, and portfolio overview |
| `/accounts` | Account listing and management |
| `/accounts/:id` | Account details, positions, editing |
| `/analysis` | Analysis reports and visualizations |
| `/advisor-team` | Agent workflow and analysis job progress |

`frontend/lib/api.ts` builds authenticated requests, handles expired sessions and 429 responses, and groups API operations into a client. `frontend/lib/config.ts` selects local API origin in development and same-origin routing in production. UI `SignedIn` / `SignedOut` components provide presentation and navigation behavior; the backend remains the security boundary.

## API surface

API Gateway forwards `ANY /api/{proxy+}` to the FastAPI Lambda. The application routes use the following operations (all require a valid Clerk bearer token):

| Route | Use |
|---|---|
| `GET`, `PUT /api/user` | Load/create profile and update preferences |
| `GET`, `POST /api/accounts` | List and create accounts |
| `PUT`, `DELETE /api/accounts/{account_id}` | Update or delete an owned account |
| `GET /api/accounts/{account_id}/positions` | List an owned account's positions |
| `POST /api/positions` | Add a position to an owned account |
| `PUT`, `DELETE /api/positions/{position_id}` | Update or delete a position after checking its account owner |
| `GET /api/instruments` | Autocomplete/reference instrument list |
| `POST /api/analyze` | Create analysis job and enqueue it |
| `GET /api/jobs`, `GET /api/jobs/{job_id}` | List or poll the current user's jobs/results |
| `DELETE /api/reset-accounts` | Delete current user's portfolio data (destructive) |
| `POST /api/populate-test-data` | Add demo data for the current user |

FastAPI also defines `GET /health`, but the deployed API Gateway route is only `/api/{proxy+}`. The health path was verified through direct Lambda invocation during deployment; it is not exposed by the API Gateway route configuration.

## Analysis agents

### Planner — `backend/planner`

SQS-triggered orchestrator. It updates job status, identifies instruments with missing classifications, attempts price updates, builds portfolio context, and runs an LLM planner with tools to invoke specialist Lambdas. Lambda names default to `alex-tagger`, `alex-reporter`, `alex-charter`, and `alex-retirement`. The planner persists final state and marks the job complete or failed.

### Tagger — `backend/tagger`

Classifies an instrument's name/symbol and fills asset-class, sector, and geographic allocation fields. It writes those attributes to `instruments`. Input and output structures use Pydantic schemas and model prompts.

### Reporter — `backend/reporter`

Computes portfolio metrics, optionally retrieves up to three semantically relevant market notes from S3 Vectors, then generates a markdown narrative. It stores content and generated-at metadata in `jobs.report_payload`. Its optional Langfuse path can run a judge and replace a low-scoring report with a fallback message.

### Charter — `backend/charter`

Creates JSON chart specifications from portfolio data and saves them in `jobs.charts_payload`. The frontend renders those specifications with Recharts.

### Retirement — `backend/retirement`

Uses portfolio value, user retirement preferences, and Monte Carlo-style scenarios to generate a projection narrative and summary values. The current code runs 500 scenarios using fixed annual return/volatility assumptions (equity 7%/18%, bonds 4%/5%, real estate 6%/12%, cash 2%), adds a hard-coded $10,000 annual contribution, models a 30-year retirement, and increases withdrawals by 3% per year in the Monte Carlo phase. It stores the output in `jobs.retirement_payload`. Some profile fields currently use defaults (for example, current age is hard-coded in the handler); the simulation does not model fees, taxes, or a user's real contribution schedule. Treat the output as an illustrative demo, not a validated financial plan.

## Database

Aurora PostgreSQL uses the RDS Data API. `backend/database` provides a boto3 client, typed schemas, and table-oriented model helpers.

| Table | Key contents | Relationships / purpose |
|---|---|---|
| `users` | Clerk subject, display name, retirement targets and allocation preferences | User profile; `clerk_user_id` is the primary key |
| `accounts` | UUID, owner subject, account name/purpose, cash balance and interest | Belongs to a user; cascading delete from user |
| `instruments` | Ticker, name, price, type, region/sector/asset-class JSONB | Shared reference and allocation metadata |
| `positions` | UUID, account UUID, symbol, quantity and as-of date | Unique per account/symbol; references account and instrument |
| `jobs` | UUID, owner, type/status, request and result JSONB payloads, errors/timestamps | Async analysis lifecycle and result storage |

Indexes support account-by-user, positions-by-account/symbol, jobs-by-user and jobs-by-status access. Timestamp triggers update `updated_at`. `backend/database/run_migrations.py` is the deployment migration entrypoint; `seed_data.py` loads reference instruments. Terraform enables one-day cluster backups and does not create a final snapshot on destroy.

## Research subsystem

- `backend/researcher/server.py` — FastAPI application and agent run.
- `backend/researcher/mcp_servers.py` — Playwright MCP process configuration.
- `backend/researcher/context.py` — research prompt, including brief browsing and output constraints.
- `backend/researcher/tools.py` — ingestion tool and retry logic.
- `backend/ingest/ingest_s3vectors.py` — create embeddings and upsert vectors.
- `backend/reporter/agent.py` — query embeddings and fetch related research context.

The current vector index is named `financial-research`. Stored metadata includes the text, topic, and timestamp. The researcher ingestion API key and database password are deployment secrets; never place their values in documentation or logs.

## Key code map

```text
frontend/                 Pages, Clerk integration, components, API client
backend/api/              FastAPI routes and Mangum Lambda adapter
backend/database/         RDS Data API wrapper, schemas, migrations, seed data
backend/planner/          SQS orchestrator, market-data step and specialist tools
backend/{tagger,reporter,
  charter,retirement}/     Specialist agents and Lambda entrypoints
backend/researcher/       Web research agent and public HTTP service
backend/ingest/           Embedding ingestion/search scripts and Lambda package
backend/scheduler/        Optional researcher scheduler invocation
terraform/2_sagemaker/    Embedding endpoint
terraform/3_ingestion/    Ingestion API, Lambda, storage and IAM
terraform/4_researcher/   Research container Lambda, URL and optional schedule
terraform/5_database/     Aurora, secret and Data API IAM
terraform/6_agents/       SQS, agent Lambdas, roles and log groups
terraform/7_frontend/     Static hosting, CloudFront, API Gateway and API Lambda
terraform/8_enterprise/   CloudWatch dashboards
scripts/                  Local runner and deployment/cleanup helpers
```
