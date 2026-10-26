# Alex Financial Advisor

**An agentic financial-planning application built with Next.js, FastAPI, AWS Lambda, SQS, Aurora PostgreSQL, and Terraform.**

Alex (Agentic Learning Equities eXplainer) is an end-to-end AI engineering project exploring how specialised agents can turn structured portfolio data into reports, charts, and retirement-planning analysis. The emphasis is on the **system around the models**: authenticated data access, orchestration, asynchronous jobs, persistence, and cloud deployment.

![Alex application screenshot](assets/alex.png)

> **Project status:** Engineering portfolio demonstration, not a regulated financial advice service. The repository contains source code and infrastructure definitions but does not promise a currently running public demo. Retirement projections use illustrative assumptions, not validated individual forecasts.

## At a glance

| Dimension | Implementation |
| --- | --- |
| User experience | Next.js and React frontend with Clerk authentication |
| API | FastAPI on AWS Lambda behind API Gateway |
| Agent orchestration | SQS-triggered planner and specialist Lambda agents |
| Agent capabilities | Instrument classification, narrative reporting, chart generation, retirement analysis |
| Model integration | OpenAI Agents SDK, LiteLLM, Amazon Bedrock |
| Data persistence | Aurora Serverless v2 PostgreSQL through the RDS Data API |
| Research retrieval | SageMaker embeddings and S3 Vectors |
| Infrastructure | Terraform stacks, CloudFront/S3, ECR and CloudWatch |

## What the application does

- Stores user-scoped investment accounts, cash balances, positions and preferences.
- Initiates asynchronous portfolio-analysis jobs, rather than keeping API requests open during model execution.
- Uses a planner to coordinate specialist functions for instrument tagging, reporting, visualisation and retirement modelling.
- Persists analysis artifacts for retrieval by the frontend.
- Includes a separate research-ingestion and vector-retrieval path.
- Computes retirement estimates using a 500-scenario Monte Carlo simulation and deterministic milestone projections.

## Architecture

~~~mermaid
flowchart TD
    Browser[Next.js frontend] -->|Authenticated requests| API[API Gateway / FastAPI Lambda]
    API --> DB[(Aurora PostgreSQL / RDS Data API)]
    API --> Queue[SQS analysis queue]
    Queue --> Planner[Planner Lambda]
    Planner --> Tagger[Instrument tagger]
    Planner --> Reporter[Report writer]
    Planner --> Charter[Chart generator]
    Planner --> Retirement[Retirement analysis]
    Tagger --> DB
    Reporter --> DB
    Charter --> DB
    Retirement --> DB
    Researcher[Research ingestion service] --> Embed[SageMaker embeddings]
    Embed --> Vectors[(S3 Vectors)]
    CDN[CloudFront / S3 hosting] --> Browser
~~~

*Simplified logical diagram; consult the Lambda handlers and Terraform definitions for deployment details.*

### Engineering decisions and trade-offs

**Keep expensive analysis outside the request-response cycle.** The API queues analysis work in SQS; a planner Lambda coordinates specialist functions. This separates interactive latency from model execution, while introducing the need to track job states, handle failures and inspect retries. See [planner handler](backend/planner/lambda_handler.py).

**Combine structured computation with generative analysis.** Aurora holds accounts, positions, instruments and job outputs. Calculation and chart-generation paths complement the LLM-generated narratives, rather than relying on a model to invent financial figures.

**Use independent infrastructure layers.** Terraform is organised by embeddings, ingestion, research, database, agents, frontend and monitoring. Individual stacks are easier to deploy or destroy during development, at the cost of more cross-stack configuration and lifecycle management.

**Treat generated recommendations as unverified output.** The system is a demonstration of applied AI engineering, not a validated advisory product.

## Retirement modelling: assumptions and limits

The [retirement module](backend/retirement/agent.py) runs **500 simulations** over a **30-year retirement period**. Current assumptions include:

- Illustrative annual Gaussian returns for equities (**7% mean, 18% standard deviation**), bonds (**4%, 5%**) and real estate (**6%, 12%**). Cash has a fixed **2%** annual return in the simulation.
- A hard-coded **$10,000 annual contribution** during accumulation.
- Retirement withdrawals increased by **3% annually** as a simplified inflation adjustment.
- Independently sampled asset-class returns, without explicit correlations or changes in market regimes.
- No claim that scenario success rates represent empirically calibrated real-world probabilities.

These assumptions are suited to an engineering demonstration, not individual investment decisions. Improvements would include user-configurable contributions, controlled random seeds, correlated-return models, taxes/fees and sensitivity analysis.

## Repository guide

| Path | Purpose |
| --- | --- |
| [assets/](assets/) | Project image |
| [backend/](backend/) | FastAPI, agents, database access, packaging and test scripts |
| [frontend/](frontend/) | Next.js application |
| [scripts/](scripts/) | Local development and deployment helpers |
| [terraform/](terraform/) | Separately managed infrastructure stacks |

## Running locally

**Prerequisites:** Node.js 20+ and npm, Python with [uv](https://docs.astral.sh/uv/), a configured Clerk application, and access to the external AWS/database services needed by the backend. AWS CLI, Terraform and Docker are needed for relevant deployment or packaging tasks.

The application is **not a self-contained offline demo**. Local development still depends on cloud services and configuration. Copy the root environment example, then create a separate frontend environment file containing your own Clerk keys and local API URL:

~~~bash
cp .env.example .env
# Create frontend/.env.local with your own local configuration
cd frontend && npm install && cd ..
cd scripts && uv run run_local.py
~~~

The development launcher expects a frontend on http://localhost:3000 and an API on http://localhost:8000. Consult [scripts/run_local.py](scripts/run_local.py) and individual service configuration before running the commands. Never commit local credentials.

## Testing and validation

Agent directories contain local/mocked scripts and cloud-dependent integration scripts. For example:

~~~bash
cd backend/planner
uv run test_simple.py
~~~

Some scripts still connect to an external database even with mocked Lambda invocations. End-to-end tests require deployed infrastructure, AWS permissions and may incur charges. **The presence of these scripts is not a claim that a CI test suite is passing.** No test executions are claimed in this documentation update.

Areas for further automated testing include isolated portfolio calculations, user-scoped authorisation, reproducible retirement simulations, Lambda contracts and deployment smoke tests.

## Infrastructure and costs

The infrastructure is divided into Terraform stacks at [terraform/2_sagemaker](terraform/2_sagemaker) through [terraform/8_enterprise](terraform/8_enterprise), covering embeddings, vector ingestion, the researcher, Aurora, agents, frontend/API and operational resources.

Review each stack's variables and available terraform.tfvars.example before applying. These provision **real, potentially billable AWS resources**. Inspect terraform plan, apply only with appropriate permissions, and destroy unused environments. The stacks maintain independent state; their README and definitions are the source of truth for deployment.

## Security and operational considerations

- Clerk authentication and server-side identity checks are implemented; individual endpoint authorisation still deserves targeted testing.
- AWS Secrets Manager is used for database credentials. Do not commit .env files, Terraform state or production secrets.
- Service-specific IAM, CORS and infrastructure boundaries are part of the design, **not a substitute for a security review**.
- Verify queue failure semantics, agent timeouts, retry behaviour and observability before treating a demonstration as production-ready.
- Do not upload real personal financial data to an unreviewed demo deployment.

## Potential next steps

1. Add deterministic retirement-model tests and configurable financial assumptions.
2. Turn the existing scripts into reproducible assertion-based tests and a CI workflow.
3. Document bounded demo costs, a deployment checklist and teardown procedure.
4. Add representative sanitised screenshots showing portfolio analysis and visualisations.
5. Strengthen end-to-end error handling, permissions and asynchronous job recovery.

## Disclaimer

Alex is an AI engineering portfolio project. Its numbers and generated commentary are illustrative, may be inaccurate and should not be used as personalised financial advice.
