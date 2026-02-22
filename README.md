# Getaround Data Platform


End-to-end data product implementation: exploratory analytics, machine-learning inference API, and production-oriented deployment on a VPS.

Video walkthrough: https://youtu.be/QUoh_RfaAc8

## Executive summary

This repository addresses a real product decision for Getaround: how to reduce rental-chain friction caused by late checkouts without harming utilization.

The implementation includes:

- analytical workflow for delay-threshold and scope decisions,
- ML inference API for rental price suggestion,
- model lifecycle infrastructure with MLflow,
- containerized deployment stack with hardened runtime defaults.

## Business context

Late checkout events can cascade into customer dissatisfaction and cancellations for the next rental. Product and operations need to tune:

- threshold: minimum delay between consecutive rentals,
- scope: all cars or selective rollout (for example Connect only),
- tradeoff: reliability of handoff vs inventory efficiency.

## Deliverables

- Dashboard: [https://streamlit.pryda.dev](https://streamlit.pryda.dev)
- MLflow tracking: [https://mlflow.pryda.dev](https://mlflow.pryda.dev)
- API docs: [https://api.pryda.dev/docs](https://api.pryda.dev/docs)

## Architecture

### Runtime topology

```text
Client traffic
  -> Traefik (TLS termination + routing)
      -> FastAPI (pricing inference)
      -> Streamlit (analytics dashboard)
      -> MLflow (tracking + registry)

FastAPI -> PostgreSQL
MLflow  -> SQLite backend + MinIO artifact storage
```

### FastAPI service design

Hexagonal boundaries are enforced in the API codebase:

```text
adapters -> application -> domain
         \-> composition root (dependency wiring)
```

This keeps business logic isolated from infrastructure concerns and improves maintainability and testability.

## Why this project is technically non-trivial

- Combines analytics, ML serving, MLOps, and infrastructure in a single coherent system.
- Uses explicit architecture boundaries (hexagonal) instead of framework-centric code coupling.
- Ships as a multi-service deployment target, not only notebooks/scripts.
- Includes production hardening work: exposed-surface reduction, dependency remediation, and deploy verification.

## Stack

| Concern | Technologies |
|---|---|
| Data analysis | pandas, numpy, matplotlib, seaborn, plotly |
| ML | scikit-learn, xgboost |
| Model lifecycle | MLflow |
| API serving | FastAPI, uvicorn, pydantic |
| Dashboard | Streamlit |
| Storage | PostgreSQL, SQLite (MLflow backend), MinIO |
| Routing and platform | Traefik, Docker, Docker Compose |
| Security and verification | pinned dependencies, Safety/pip-audit scans, Docker smoke checks |

## Repository structure

- `containers/getaround/`: production deployment stack and service composition.
- `containers/getaround/app/fastapi/app/`: FastAPI service with hexagonal layering.
- `containers/getaround/app/streamlit/`: production dashboard service.
- `containers/getaround/app/mlflow/`: MLflow service container files.
- `streamlit_dev/`: local dashboard environment.
- `data/`: source datasets.
- `ml_models/`: model experimentation artifacts.
- `model_final.py`: training/logging workflow script.

## Deployment and operations

### Local production-like run

```bash
cd containers/getaround
cp .env.example .env
# replace every change-me value
docker compose build
docker compose up -d
```

### VPS quickstart

1. Create `containers/getaround/.env` from `containers/getaround/.env.example`.
2. Set strong secrets for all sensitive variables.
3. Configure DNS records for service hostnames.
4. Build and start:

```bash
cd containers/getaround
docker compose build
docker compose up -d
```

### Security posture (current)

- Traefik insecure dashboard mode disabled.
- MinIO initialized without anonymous download policy.
- Secrets externalized through environment variables.
- Dependency vulnerability workflow integrated into maintenance process.

## API example

Python:

```python
import requests

payload = {
    "model_key": "Citroen",
    "mileage": 150000,
    "engine_power": 100,
    "fuel": "diesel",
    "paint_color": "green",
    "car_type": "convertible",
    "private_parking_available": True,
    "has_gps": True,
    "has_air_conditioning": True,
    "automatic_car": True,
    "has_getaround_connect": True,
    "has_speed_regulator": True,
    "winter_tires": True,
}

response = requests.post(
    "https://api.pryda.dev/prediction",
    json=payload,
    timeout=30,
)
print(response.json())
```

curl:

```bash
curl -X POST "https://api.pryda.dev/prediction" \
  -H "accept: application/json" \
  -H "Content-Type: application/json" \
  -d '{
    "model_key": "Citroen",
    "mileage": 150000,
    "engine_power": 100,
    "fuel": "diesel",
    "paint_color": "green",
    "car_type": "convertible",
    "private_parking_available": true,
    "has_gps": true,
    "has_air_conditioning": true,
    "automatic_car": true,
    "has_getaround_connect": true,
    "has_speed_regulator": true,
    "winter_tires": true
  }'
```

## Notes for reviewers

- The prediction endpoint requires the configured `MODEL_URI` artifact to be present in MLflow.
- The project is intentionally scoped as a portfolio-grade production system, not a notebook-only demonstration.
