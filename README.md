# IntelliSupply

[![API Docs](https://img.shields.io/badge/API-Docs-green)](https://ayrasaqib.github.io/supply-chain-risk-api-docs/)
[![CI](https://github.com/zayan-farazi/SENG3011/actions/workflows/terraform-ci.yml/badge.svg)](https://github.com/zayan-farazi/SENG3011/actions/workflows/terraform-ci.yml)

Supply chain risk intelligence platform that scores any global transport hub using weather ML models and geopolitical news sentiment. Features a live risk dashboard, 7-day forecasts, optimal route pathfinding, and watchlist email alerts.

---

## Overview

IntelliSupply transforms raw weather forecasts and geopolitical news into a single, unified disruption risk score for each supply chain hub. Instead of manually monitoring dozens of locations, users get colour-coded risk indicators, 7-day predictive forecasts, and real-time alerts all from one dashboard.

**Live frontend:** https://supply-chain-risk-m4mbhukg6-ayras-projects.vercel.app

---

## Features

- **Risk dashboard**: interactive map of ~1,800 monitored hubs colour-coded by risk level (Low / Elevated / High / Critical)
- **7-day forecast**: ML model trained on ERA5 historical data predicts weather-driven disruption risk up to 7 days ahead
- **Combined risk scoring**: 65% weather + 35% geopolitical sentiment, with graceful fallback when news data is unavailable
- **Optimal path finding**: Dijkstra's algorithm over a risk-weighted graph of hubs to recommend the safest shipping route
- **Custom locations**: analyse any latitude/longitude, not just predefined hubs
- **Watchlist alerts**: subscribe to hubs and receive email notifications when risk reaches Critical

---

## Architecture

The backend is a Python microservices architecture on AWS, deployed via Terraform and triggered by an event-driven pipeline.

```
EventBridge (daily) → Ingestion → S3 → Processing → S3 → Analytics → DynamoDB / S3
                                                              ↑
                                                    ML model + News Sentiment API
```

| Layer | Technology |
|---|---|
| Compute | AWS Lambda (Python 3.12) |
| API | Amazon API Gateway (HTTP API) |
| Storage | Amazon S3 (data lake), DynamoDB (locations, watchlist, scores, messages) |
| Scheduling | Amazon EventBridge |
| Email | Amazon SES |
| IaC | Terraform |
| Frontend | Next.js, deployed on Vercel |
| CI/CD | GitHub Actions |

### Microservices

| Service | Responsibility |
|---|---|
| `ingestion` | Fetches raw weather forecasts from PirateWeather API and stores in S3 |
| `retrieval` | Reads raw or processed weather data from S3 |
| `processing` | Transforms raw forecasts into 6-hour snapshot schema for the ML model |
| `analytics` | Runs ML model, fetches geopolitical sentiment, computes combined risk score |
| `location` | Manages monitored and user-defined dynamic hubs |
| `hub_sync` | Syncs ~1,800 hub records from PortWatch API and builds the pathfinding graph |
| `pathfinding` | Returns optimal risk-weighted route between two hubs using Dijkstra's algorithm |
| `watchlist` | Manages hub subscriptions and dispatches SES email alerts |
| `auth` | Handles Cognito-backed user profile and password management |
| `testing` | Dedicated Lambda for running E2E tests against the deployed environment |

---

## ML Model

The analytics service uses a supervised regression model (Random Forest or XGBoost, whichever achieves lower MAE) to predict a continuous risk score in [0, 1] from six meteorological features.

**Features:** wind gust, precipitation intensity, surface pressure, wind speed, temperature, humidity

**Training data:** ERA5 historical reanalysis (2015–2026) across 28 global supply chain hubs

**Evaluation results:**

| Metric | Threshold | Achieved |
|---|---|---|
| MAE | < 0.08 | 0.0015 |
| R² | > 0.80 | 0.9996 |
| High-risk Recall | ≥ 0.60 | 1.00 |
| Critical-risk Recall | ≥ 0.60 | 0.96 |

---

## API

Base URL (production): `https://ljtwsbvd8l.execute-api.ap-southeast-2.amazonaws.com/prod`

Full OpenAPI spec: [`docs/openapi.yaml`](docs/openapi.yaml): rendered at the [API Docs](https://ayrasaqib.github.io/supply-chain-risk-api-docs/) link above.

Key endpoints:

```
GET  /ese/v1/location/list                         List all monitored and dynamic hubs
GET  /ese/v1/location/{hub_id}                     Get a hub by ID
POST /ese/v1/location                              Create a dynamic hub
POST /ese/v1/ingest/weather/{hub_id}               Ingest weather data for a hub
GET  /ese/v1/retrieve/raw/weather/{hub_id}         Retrieve raw weather data
GET  /ese/v1/retrieve/processed/weather/{hub_id}   Retrieve processed weather data
POST /ese/v1/process/weather                       Process raw weather data
GET  /ese/v1/risk/location/{hub_id}                Get combined risk score for a hub
GET  /ese/v1/pathfinding/{hub_id_1}/{hub_id_2}     Get optimal route between two hubs
POST /ese/v1/watchlist/{hub_id}/{email}            Subscribe to hub alerts
GET  /ese/v1/watchlist/{email}                     List subscribed hubs
```

---

## Repositories

| Repo | Description |
|---|---|
| [Backend](https://github.com/zayan-farazi/SENG3011) | This repo: Lambdas, Terraform, tests |
| [Frontend](https://github.com/ayrasaqib/supply-chain-risk-frontend) | Next.js dashboard |

---

## Local Development

### Prerequisites

- Python 3.12
- Terraform ≥ 1.2
- AWS CLI configured with appropriate credentials
- Node.js (for frontend)

### Backend setup

```bash
# Install dependencies
pip install -r requirements.txt
pip install -r requirements-dev.txt

# Run unit and integration tests
pytest tests/unit tests/integration

# Run stress tests (local only)
pytest tests/stress

# Lint and type check
ruff check .
mypy .
```

### Infrastructure

```bash
# Build Lambda artifacts
bash scripts/build_lambda_artifacts.sh

# Initialise Terraform (replace placeholders)
terraform -chdir=terraform init \
  -backend-config="bucket=<your-state-bucket>" \
  -backend-config="key=dev/terraform.tfstate" \
  -backend-config="region=ap-southeast-2"

# Deploy to dev
terraform -chdir=terraform apply \
  -var="data_bucket_name=<your-app-bucket>" \
  -var="pirate_weather_api_key=<your-key>"
```

See [`docs/aws/README.md`](docs/aws/README.md) for full AWS setup instructions including IAM roles and GitHub OIDC configuration.

---

## CI/CD Pipeline

| Workflow | Trigger | Purpose |
|---|---|---|
| `terraform-ci.yml` | PR / branch push | Lint, type check, Terraform validate, unit + integration tests (≥85% coverage) |
| `terraform-deploy-dev.yml` | Non-`main` push | Deploy to `dev` environment |
| `terraform-deploy-staging.yml` | Push to `main` | Deploy to `staging`, run system, contract, and E2E tests |
| `terraform-deploy-prod.yml` | Manual | Deploy to `prod` with manual approval gate |
| `combined-quality-report.yml` | After staging | Merge CI and staging artifacts into final PDF test and coverage report |

---

## Testing

Over 200 test cases across 7 test types:

| Type | Environment | Purpose |
|---|---|---|
| Unit | Local / CI | Individual Lambda function logic, mocked infrastructure |
| Integration | Local / CI | Multi-Lambda interactions, mocked infrastructure |
| Contract | Staging | API response schema validation against OpenAPI spec |
| System | Staging | End-to-end endpoint behaviour in a deployed environment |
| Stress | Local | Performance and fault tolerance under 300 simulated users |
| Backend E2E | Staging | Full user workflow via dedicated testing Lambda |
| Frontend E2E | Local | UI flows across Chromium, Firefox, and WebKit via Playwright |

---

## Deployment Environments

| Environment | Base URL |
|---|---|
| Development | `https://1oj9t7sjq5.execute-api.ap-southeast-2.amazonaws.com/dev` |
| Staging | `https://yn139tzyx8.execute-api.ap-southeast-2.amazonaws.com/staging` |
| Production | `https://ljtwsbvd8l.execute-api.ap-southeast-2.amazonaws.com/prod` |
