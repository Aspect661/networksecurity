# Network Security — Phishing Website Detection

An end-to-end MLOps project that classifies websites as **phishing or legitimate** using 31 numerical features derived from URL structure, SSL properties, and web traffic patterns. The pipeline covers data ingestion from MongoDB, validation, transformation, model training with experiment tracking, and deployment via Docker on AWS EC2 — with full CI/CD through GitHub Actions.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [ML Pipeline](#ml-pipeline)
- [API Endpoints](#api-endpoints)
- [Tech Stack](#tech-stack)
- [CI/CD Pipeline](#cicd-pipeline)
- [Environment Variables](#environment-variables)
- [Running Locally](#running-locally)
- [Running with Docker](#running-with-docker)

---

## Overview

| Attribute | Details |
|-----------|---------|
| **Task** | Binary Classification — Phishing (1) vs Legitimate (0) |
| **Features** | 31 numerical URL / domain / traffic attributes |
| **Data Source** | MongoDB Atlas |
| **Models** | Random Forest, Decision Tree, Gradient Boosting, Logistic Regression, AdaBoost |
| **Hyperparameter Tuning** | GridSearchCV (3-fold CV) |
| **Experiment Tracking** | MLflow + DagsHub |
| **Serving** | FastAPI on port 8000 |
| **Deployment** | Docker on AWS EC2 via ECR |
| **Artifact Storage** | AWS S3 (timestamped versions) |

---

## Architecture

```
MongoDB Atlas
    │
    ▼
Data Ingestion ──► Data Validation ──► Data Transformation ──► Model Trainer
                                                                      │
                                                               MLflow / DagsHub
                                                               (experiment logs)
                                                                      │
                                                               final_model/
                                                               ├── model.pkl
                                                               └── preprocessor.pkl
                                                                      │
                                                             ┌────────┴────────┐
                                                             │   FastAPI App   │
                                                             │   Port: 8000    │
                                                             └────────┬────────┘
                                                                      │
                                                               AWS EC2 Instance
                                                               (Docker Container)
                                                                      │
                                                         ┌────────────┴────────────┐
                                                         │        AWS S3           │
                                                         │  /artifact/{timestamp}  │
                                                         │  /final_model/{timestamp}│
                                                         └─────────────────────────┘
```

---

## Project Structure

```
networksecurity/
├── .github/
│   └── workflows/
│       └── main.yml                        # CI/CD pipeline
├── networksecurity/
│   ├── cloud/
│   │   └── s3_syncer.py                    # S3 sync utilities
│   ├── components/
│   │   ├── data_ingestion.py               # MongoDB → CSV
│   │   ├── data_validation.py              # Schema + drift detection
│   │   ├── data_transformation.py          # KNN imputation + normalization
│   │   └── model_trainer.py                # Multi-model training + MLflow tracking
│   ├── constant/
│   │   └── training_pipeline/__init__.py   # Pipeline constants
│   ├── entity/
│   │   ├── artifact_entity.py              # Artifact dataclasses
│   │   └── config_entity.py                # Config dataclasses
│   ├── exception/
│   │   └── exception.py                    # Custom exception handler
│   ├── logging/
│   │   └── logger.py                       # File-based logging
│   ├── pipeline/
│   │   └── training_pipeline.py            # Full pipeline orchestrator
│   └── utils/
│       ├── main_utils/utils.py             # YAML, pickle, numpy helpers
│       └── ml_utils/
│           ├── model/estimator.py          # NetworkModel wrapper
│           └── metric/classification_metric.py
├── data_schema/
│   └── schema.yaml                         # 31 features + target column
├── app.py                                  # FastAPI entrypoint
├── main.py                                 # CLI pipeline runner
├── Dockerfile
├── requirements.txt
└── push_data.py                            # Upload data to MongoDB
```

---

## ML Pipeline

### 1. Data Ingestion
- Queries the `Network_Data` collection from MongoDB database `TUHINAIR`
- Exports raw data to `Artifacts/data_ingestion/feature_store/phishingData.csv`
- Splits data 80/20 into train and test CSVs

### 2. Data Validation
- Validates that all 31 expected columns are present and numeric
- Runs a **Kolmogorov-Smirnov drift test** on each feature between train and test sets
- Generates a drift report in YAML format

### 3. Data Transformation
- Applies **KNN Imputation** (`k=3`, uniform weights) to handle missing values
- Converts target values: `-1 → 0` for binary classification
- Saves the fitted `preprocessor.pkl` to `final_model/` for inference

### 4. Model Trainer
Five classifiers are trained and evaluated with GridSearchCV:

| Model | Key Hyperparameters |
|-------|---------------------|
| Random Forest | n_estimators: [8, 16, 32, 100, 128, 200, 256] |
| Decision Tree | criterion: [gini, entropy, log_loss] |
| Gradient Boosting | learning_rate, subsample, n_estimators |
| Logistic Regression | — |
| AdaBoost | learning_rate, n_estimators |

The best model is selected by highest R² score on the test set. Metrics (F1, Precision, Recall) are logged to MLflow via DagsHub. The final model is saved as `final_model/model.pkl`.

- **Minimum expected accuracy**: 0.6
- **Max allowed train/test gap** (overfitting threshold): 0.05

### 5. Artifact Sync to S3
After training, both the `Artifacts/` directory and `final_model/` are synced to S3:
```
s3://networksecuritydemo/artifact/{timestamp}
s3://networksecuritydemo/final_model/{timestamp}
```

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/` | Redirects to Swagger UI (`/docs`) |
| `GET` | `/train` | Triggers the full training pipeline |
| `POST` | `/predict` | Accepts a CSV file, returns predictions as an HTML table |

### Prediction Flow
1. Upload a CSV with 31 feature columns
2. The preprocessor applies KNN imputation + scaling
3. The model predicts phishing (1) or legitimate (0)
4. Results are appended as `predicted_column` and returned as an HTML table
5. Output is also saved to `prediction_output/output.csv`

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Web Framework | FastAPI, Uvicorn |
| ML | scikit-learn (classifiers, GridSearchCV, KNN imputer) |
| Data | pandas, numpy |
| Database | MongoDB Atlas (pymongo) |
| Experiment Tracking | MLflow + DagsHub |
| Serialization | dill, pickle |
| Cloud Storage | AWS S3 (boto3 / awscli) |
| Container Registry | AWS ECR |
| Compute | AWS EC2 |
| Containerization | Docker (python:3.10-slim) |
| CI/CD | GitHub Actions |

---

## CI/CD Pipeline

Triggered on every push to `main` (ignores `README.md` changes).

```
Push to main
    │
    ▼
Job 1: Continuous Integration (ubuntu-latest)
    ├── Checkout code
    ├── Lint code
    └── Run unit tests
    │
    ▼
Job 2: Continuous Delivery (ubuntu-latest)
    ├── Configure AWS credentials
    ├── Login to Amazon ECR
    ├── Build Docker image
    └── Push image to ECR
    │
    ▼
Job 3: Continuous Deployment (self-hosted EC2 runner)
    ├── Configure AWS credentials
    ├── Login to ECR
    ├── Pull latest image
    ├── Stop & remove old container
    ├── Run new container on port 8000
    └── Prune old images
```

---

## Environment Variables

These must be set as **GitHub repository secrets** and are injected into the Docker container at runtime:

| Secret | Description |
|--------|-------------|
| `AWS_ACCESS_KEY_ID` | AWS access key |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key |
| `AWS_REGION` | AWS region (e.g. `us-east-1`) |
| `ECR_REPOSITORY_NAME` | ECR repository name |
| `MONGODB_URL_KEY` | MongoDB Atlas connection string |
| `DAGSHUB_USER_TOKEN` | DagsHub personal access token for MLflow |

For local development, create a `.env` file in the project root:
```env
MONGODB_URL_KEY=mongodb+srv://<user>:<password>@cluster0.xxxxx.mongodb.net/
AWS_ACCESS_KEY_ID=your_key
AWS_SECRET_ACCESS_KEY=your_secret
AWS_REGION=us-east-1
DAGSHUB_USER_TOKEN=your_token
```

> **Never commit `.env` to git.** It should be in `.gitignore`.

---

## Running Locally

```bash
# 1. Clone the repo
git clone https://github.com/TuhinDas661/networksecurity.git
cd networksecurity

# 2. Create and activate a virtual environment
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# 3. Install dependencies
pip install -r requirements.txt

# 4. Set up environment variables
cp .env.example .env  # fill in your credentials

# 5. Start the server
python app.py
```

Visit `http://localhost:8000/docs` for the Swagger UI.

---

## Running with Docker

```bash
# Build
docker build -t networksecurity .

# Run
docker run -d -p 8000:8000 \
  -e AWS_ACCESS_KEY_ID=your_key \
  -e AWS_SECRET_ACCESS_KEY=your_secret \
  -e AWS_REGION=us-east-1 \
  -e MONGODB_URL_KEY=your_mongo_uri \
  -e DAGSHUB_USER_TOKEN=your_token \
  --name networksecuritydemo \
  networksecurity
```

Visit `http://localhost:8000`.
