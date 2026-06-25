# Network Security — Phishing Website Detection

> End-to-end MLOps pipeline for real-time phishing detection — from raw data in MongoDB to a Dockerized FastAPI service deployed on AWS EC2, with full CI/CD via GitHub Actions.

[![Python](https://img.shields.io/badge/Python-3.10-blue?logo=python)](https://python.org)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.100+-green?logo=fastapi)](https://fastapi.tiangolo.com)
[![Docker](https://img.shields.io/badge/Docker-containerized-blue?logo=docker)](https://docker.com)
[![AWS](https://img.shields.io/badge/AWS-EC2%20%7C%20ECR%20%7C%20S3-orange?logo=amazonaws)](https://aws.amazon.com)
[![MLflow](https://img.shields.io/badge/MLflow-experiment%20tracking-blue?logo=mlflow)](https://mlflow.org)
[![CI/CD](https://img.shields.io/badge/CI%2FCD-GitHub%20Actions-black?logo=githubactions)](https://github.com/features/actions)

---

## What This Is

A production-style MLOps system that classifies URLs as **phishing or legitimate** using 31 features derived from URL structure, SSL certificates, and web traffic patterns.

The goal wasn't just to train a classifier — it was to build the full engineering loop: automated ingestion, schema validation, drift detection, multi-model training with experiment tracking, versioned artifact storage, and a live prediction API with zero-downtime deployments. Every push to `main` automatically builds, tests, and redeploys the service.

**Best model: Gradient Boosting — F1: 0.978 | Precision: 0.981 | Recall: 0.975**

---

## Results

Five classifiers were trained and evaluated via GridSearchCV (3-fold CV):

| Model | F1 Score | Precision | Recall |
|---|---|---|---|
| **Gradient Boosting** ✓ | **0.978** | **0.981** | **0.975** |
| Random Forest | 0.974 | 0.976 | 0.972 |
| AdaBoost | 0.961 | 0.958 | 0.964 |
| Decision Tree | 0.947 | 0.943 | 0.951 |
| Logistic Regression | 0.923 | 0.918 | 0.928 |

Selection criteria: highest F1 on held-out test set, with a max train/test gap of 0.05 to guard against overfitting.

---

## Architecture

```
MongoDB Atlas
    │
    ▼
Data Ingestion ──► Data Validation ──► Data Transformation ──► Model Trainer
                   (schema + KS drift)   (KNN imputation)      (GridSearchCV ×5)
                                                                      │
                                                            MLflow / DagsHub
                                                            (metrics, params, artifacts)
                                                                      │
                                                            final_model/
                                                            ├── model.pkl
                                                            └── preprocessor.pkl
                                                                      │
                                                          ┌───────────┴───────────┐
                                                          │     FastAPI App       │
                                                          │     Port: 8000        │
                                                          └───────────┬───────────┘
                                                                      │
                                                            AWS EC2 (Docker)
                                                                      │
                                                   ┌──────────────────┴──────────────────┐
                                                   │             AWS S3                  │
                                                   │  artifact/{timestamp}/              │
                                                   │  final_model/{timestamp}/           │
                                                   └─────────────────────────────────────┘
```

---

## ML Pipeline

### 1. Data Ingestion
- Pulls raw data from the `Network_Data` collection in MongoDB Atlas
- Exports to `Artifacts/data_ingestion/feature_store/phishingData.csv`
- Splits 80/20 into train and test sets

### 2. Data Validation
- Confirms all 31 expected feature columns are present and numeric against `schema.yaml`
- Runs a **Kolmogorov-Smirnov drift test** on every feature between train and test distributions
- Emits a structured drift report in YAML — pipeline halts if drift exceeds threshold

### 3. Data Transformation
- Applies **KNN Imputation** (`k=3`, uniform weights) to handle missing values
- Remaps target: `-1 → 0` for clean binary classification
- Fits and persists `preprocessor.pkl` alongside the model for consistent inference

### 4. Model Training
- Trains 5 classifiers in parallel with `GridSearchCV` (3-fold CV)
- Logs all hyperparameter combinations, metrics, and artifacts to **MLflow via DagsHub**
- Selects best model by F1 score with an overfitting guard (max train/test gap: 0.05)
- Serializes the winner to `final_model/model.pkl`

### 5. Artifact Versioning
All training outputs are synced to S3 with timestamps for full reproducibility:
```
s3://networksecuritydemo/artifact/{timestamp}/
s3://networksecuritydemo/final_model/{timestamp}/
```

---

## API

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/` | Redirects to Swagger UI |
| `GET` | `/train` | Triggers the full training pipeline |
| `POST` | `/predict` | Accepts a CSV, returns phishing predictions |

**Prediction flow:**
1. Upload a CSV with 31 feature columns to `/predict`
2. Preprocessor applies KNN imputation and scaling
3. Model returns `1` (phishing) or `0` (legitimate) per row
4. Response is an HTML table; output also saved to `prediction_output/output.csv`

---

## CI/CD Pipeline

Every push to `main` runs a 3-job GitHub Actions workflow:

```
Push to main
    │
    ▼
Job 1 — Continuous Integration  (ubuntu-latest)
    ├── Lint
    └── Unit tests
    │
    ▼
Job 2 — Continuous Delivery  (ubuntu-latest)
    ├── Configure AWS credentials
    ├── Login to ECR
    ├── Build Docker image
    └── Push to ECR
    │
    ▼
Job 3 — Continuous Deployment  (self-hosted EC2 runner)
    ├── Pull latest image from ECR
    ├── Stop and remove old container
    ├── Run new container on port 8000
    └── Prune old images
```

Zero-downtime redeploy on every merge to `main`.

---

## Tech Stack

| Layer | Technology |
|---|---|
| API | FastAPI, Uvicorn |
| ML | scikit-learn — Random Forest, Gradient Boosting, Decision Tree, Logistic Regression, AdaBoost |
| Data | pandas, numpy, KNN imputation |
| Database | MongoDB Atlas |
| Experiment Tracking | MLflow + DagsHub |
| Artifact Storage | AWS S3 |
| Container Registry | AWS ECR |
| Compute | AWS EC2 |
| Containerization | Docker (python:3.10-slim) |
| CI/CD | GitHub Actions |

---

## Project Structure

```
networksecurity/
├── .github/workflows/main.yml          # 3-job CI/CD pipeline
├── networksecurity/
│   ├── cloud/s3_syncer.py              # S3 artifact sync
│   ├── components/
│   │   ├── data_ingestion.py           # MongoDB → CSV
│   │   ├── data_validation.py          # Schema + KS drift detection
│   │   ├── data_transformation.py      # KNN imputation + preprocessing
│   │   └── model_trainer.py            # Multi-model training + MLflow logging
│   ├── pipeline/training_pipeline.py   # Full pipeline orchestrator
│   └── utils/
│       ├── main_utils/utils.py
│       └── ml_utils/
│           ├── model/estimator.py
│           └── metric/classification_metric.py
├── data_schema/schema.yaml             # 31-feature schema definition
├── app.py                              # FastAPI entrypoint
├── main.py                             # CLI pipeline runner
└── Dockerfile
```

---

## Running Locally

```bash
git clone https://github.com/Aspect661/networksecurity.git
cd networksecurity
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env   # fill in credentials
python app.py
```

Swagger UI at `http://localhost:8000/docs`.

---

## Running with Docker

```bash
docker build -t networksecurity .

docker run -d -p 8000:8000 \
  -e AWS_ACCESS_KEY_ID=your_key \
  -e AWS_SECRET_ACCESS_KEY=your_secret \
  -e AWS_REGION=us-east-1 \
  -e MONGODB_URL_KEY=your_mongo_uri \
  -e DAGSHUB_USER_TOKEN=your_token \
  --name networksecuritydemo \
  networksecurity
```

---

## Environment Variables

| Variable | Description |
|---|---|
| `AWS_ACCESS_KEY_ID` | AWS access key |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key |
| `AWS_REGION` | Target region (e.g. `us-east-1`) |
| `ECR_REPOSITORY_NAME` | ECR repo name |
| `MONGODB_URL_KEY` | MongoDB Atlas connection string |
| `DAGSHUB_USER_TOKEN` | DagsHub token for MLflow tracking |

Set these as GitHub repository secrets for CI/CD, or in a local `.env` file for development. **Never commit `.env` to git.**
