# Part 62: MLOps Pipeline — CI/CD สำหรับ Machine Learning

## บทนำ: ทำไม ML ต้องการ CI/CD แบบพิเศษ?

Machine Learning มีความซับซ้อนที่แตกต่างจาก Software Development ทั่วไป ทั้งการจัดการ data, training code, model artifacts และ hyperparameters ทำให้ CI/CD สำหรับ ML (MLOps) ต้องการ practices พิเศษ

### ความแตกต่างระหว่าง Traditional CI/CD และ MLOps

| ด้าน | Traditional CI/CD | MLOps |
|------|-------------------|-------|
| Input | Source code | Code + Data + Models |
| Testing | Unit/Integration tests | Code tests + Model validation |
| Versioning | Git | Git + DVC + Model Registry |
| Artifacts | Binary/Container | Models + Datasets |
| Quality Gate | Test pass/fail | Model metrics threshold |
| Reproducibility | Dockerfile | Code + Data + Environment |
| Monitoring | Uptime/Latency | Model accuracy drift |

### MLOps Maturity Levels

```
Level 0: Manual ML
  └── Data scientists เทรน model ด้วย Jupyter notebook
  └── Manual deployment ด้วย script
  └── ไม่มี monitoring

Level 1: ML Pipeline Automation
  └── Automated training pipeline
  └── Feature store
  └── Model versioning
  └── Basic monitoring

Level 2: CI/CD Pipeline Automation (สิ่งที่เราเรียนในบทนี้)
  └── Automated testing ของ ML code
  └── Automated model training เมื่อ data/code เปลี่ยน
  └── Automated deployment พร้อม validation
  └── Continuous monitoring + drift detection
```

---

## 1. ML Workflow Overview

### 1.1 โครงสร้าง MLOps Project

```
ml-project/
├── data/                    # Tracked by DVC
│   ├── raw/
│   ├── processed/
│   └── features/
├── models/                  # Tracked by DVC
│   ├── trained/
│   └── deployed/
├── src/
│   ├── data/
│   │   ├── ingest.py
│   │   ├── validate.py
│   │   └── transform.py
│   ├── features/
│   │   ├── engineer.py
│   │   └── select.py
│   ├── training/
│   │   ├── train.py
│   │   ├── evaluate.py
│   │   └── hyperparameter_search.py
│   └── serving/
│       ├── predict.py
│       ├── api.py
│       └── monitoring.py
├── tests/
│   ├── unit/
│   ├── integration/
│   └── model/
├── dvc.yaml                 # DVC pipeline definition
├── params.yaml              # Hyperparameters
├── requirements.txt
├── Dockerfile
├── Makefile
└── .github/
    └── workflows/
        ├── ci.yml
        ├── train.yml
        └── deploy.yml
```

### 1.2 params.yaml

```yaml
# params.yaml - ควบคุม hyperparameters ทั้งหมด
data:
  raw_path: data/raw/transactions.csv
  processed_path: data/processed/
  test_size: 0.2
  random_state: 42
  
features:
  numerical_features:
    - amount
    - frequency
    - recency
  categorical_features:
    - merchant_category
    - payment_method
  target_column: is_fraud
  
training:
  model_type: xgboost  # xgboost, random_forest, neural_network
  n_estimators: 100
  max_depth: 6
  learning_rate: 0.1
  subsample: 0.8
  
evaluation:
  metrics:
    - accuracy
    - precision
    - recall
    - f1_score
    - roc_auc
  threshold:
    min_f1_score: 0.85
    min_roc_auc: 0.90
    max_false_positive_rate: 0.10

serving:
  prediction_threshold: 0.5
  max_latency_ms: 100
  batch_size: 32
```

---

## 2. DVC (Data Version Control)

### 2.1 DVC คืออะไร?

DVC เป็น version control สำหรับ data และ ML models คล้ายกับ Git แต่เหมาะสำหรับ large binary files และ datasets

### 2.2 Setup DVC

```bash
# ติดตั้ง DVC
pip install dvc dvc-s3  # หรือ dvc-gcs, dvc-azure

# Initialize DVC ใน git repo
git init
dvc init

# ตั้งค่า remote storage (S3)
dvc remote add -d myremote s3://my-ml-data/dvc-storage
dvc remote modify myremote region ap-southeast-1

# Add data files
dvc add data/raw/transactions.csv
dvc add models/

# Push data to remote
dvc push

# Commit DVC files ไปยัง Git
git add data/raw/transactions.csv.dvc models/.gitignore .dvcignore
git commit -m "Add training data and initial model"
```

### 2.3 DVC Pipeline (dvc.yaml)

```yaml
# dvc.yaml
stages:
  # Stage 1: Data Ingestion
  ingest_data:
    cmd: python src/data/ingest.py
    deps:
      - src/data/ingest.py
      - ${data.raw_path}
    params:
      - data.raw_path
    outs:
      - data/raw/transactions_validated.csv

  # Stage 2: Data Validation
  validate_data:
    cmd: python src/data/validate.py
    deps:
      - src/data/validate.py
      - data/raw/transactions_validated.csv
    params:
      - data
    outs:
      - data/validation_report.json
    metrics:
      - data/validation_metrics.json:
          cache: false

  # Stage 3: Feature Engineering
  feature_engineering:
    cmd: python src/features/engineer.py
    deps:
      - src/features/engineer.py
      - data/raw/transactions_validated.csv
    params:
      - features
      - data.random_state
    outs:
      - data/features/train.parquet
      - data/features/test.parquet

  # Stage 4: Model Training
  train_model:
    cmd: python src/training/train.py
    deps:
      - src/training/train.py
      - data/features/train.parquet
    params:
      - training
      - features.target_column
    outs:
      - models/trained/model.pkl
      - models/trained/preprocessor.pkl
    metrics:
      - metrics/training_metrics.json:
          cache: false

  # Stage 5: Model Evaluation
  evaluate_model:
    cmd: python src/training/evaluate.py
    deps:
      - src/training/evaluate.py
      - models/trained/model.pkl
      - data/features/test.parquet
    params:
      - evaluation
    metrics:
      - metrics/evaluation_metrics.json:
          cache: false
    plots:
      - metrics/confusion_matrix.csv:
          cache: false
          x: predicted
          y: actual
      - metrics/roc_curve.csv:
          cache: false
          x: fpr
          y: tpr
```

### 2.4 Training Script

```python
# src/training/train.py
import yaml
import pickle
import json
import logging
from pathlib import Path
import pandas as pd
import numpy as np
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler, LabelEncoder
from xgboost import XGBClassifier
import mlflow
import mlflow.xgboost

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def load_params():
    with open('params.yaml') as f:
        return yaml.safe_load(f)

def train(params):
    """Train ML model"""
    mlflow.set_tracking_uri(params.get('mlflow_tracking_uri', './mlruns'))
    
    with mlflow.start_run():
        # Log parameters
        mlflow.log_params({
            'model_type': params['training']['model_type'],
            'n_estimators': params['training']['n_estimators'],
            'max_depth': params['training']['max_depth'],
            'learning_rate': params['training']['learning_rate']
        })
        
        # Load data
        logger.info("Loading training data...")
        train_data = pd.read_parquet('data/features/train.parquet')
        
        target_col = params['features']['target_column']
        feature_cols = (
            params['features']['numerical_features'] +
            params['features']['categorical_features']
        )
        
        X_train = train_data[feature_cols]
        y_train = train_data[target_col]
        
        logger.info(f"Training with {len(X_train)} samples, {len(feature_cols)} features")
        logger.info(f"Class distribution: {y_train.value_counts().to_dict()}")
        
        # Build model
        model = XGBClassifier(
            n_estimators=params['training']['n_estimators'],
            max_depth=params['training']['max_depth'],
            learning_rate=params['training']['learning_rate'],
            subsample=params['training']['subsample'],
            random_state=params['data']['random_state'],
            use_label_encoder=False,
            eval_metric='logloss',
            tree_method='hist'  # สำหรับ faster training
        )
        
        # Train with early stopping
        eval_set = [(X_train, y_train)]
        model.fit(
            X_train, y_train,
            eval_set=eval_set,
            verbose=100
        )
        
        # Save artifacts
        Path('models/trained').mkdir(parents=True, exist_ok=True)
        
        with open('models/trained/model.pkl', 'wb') as f:
            pickle.dump(model, f)
        
        # Log metrics
        training_metrics = {
            'best_iteration': model.best_iteration,
            'best_score': model.best_score
        }
        
        mlflow.log_metrics(training_metrics)
        mlflow.xgboost.log_model(model, 'model')
        
        # Save metrics for DVC
        Path('metrics').mkdir(parents=True, exist_ok=True)
        with open('metrics/training_metrics.json', 'w') as f:
            json.dump(training_metrics, f, indent=2)
        
        logger.info(f"Training complete! Metrics: {training_metrics}")
        
        return model

if __name__ == '__main__':
    params = load_params()
    train(params)
```

### 2.5 Evaluation Script

```python
# src/training/evaluate.py
import yaml
import pickle
import json
import logging
import pandas as pd
import numpy as np
from sklearn.metrics import (
    accuracy_score, precision_score, recall_score,
    f1_score, roc_auc_score, confusion_matrix,
    roc_curve
)
from pathlib import Path

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def evaluate():
    with open('params.yaml') as f:
        params = yaml.safe_load(f)
    
    # Load model and test data
    with open('models/trained/model.pkl', 'rb') as f:
        model = pickle.load(f)
    
    test_data = pd.read_parquet('data/features/test.parquet')
    
    target_col = params['features']['target_column']
    feature_cols = (
        params['features']['numerical_features'] +
        params['features']['categorical_features']
    )
    
    X_test = test_data[feature_cols]
    y_test = test_data[target_col]
    
    # Predictions
    y_pred = model.predict(X_test)
    y_pred_proba = model.predict_proba(X_test)[:, 1]
    
    # Calculate metrics
    metrics = {
        'accuracy': float(accuracy_score(y_test, y_pred)),
        'precision': float(precision_score(y_test, y_pred)),
        'recall': float(recall_score(y_test, y_pred)),
        'f1_score': float(f1_score(y_test, y_pred)),
        'roc_auc': float(roc_auc_score(y_test, y_pred_proba))
    }
    
    logger.info(f"Evaluation metrics: {metrics}")
    
    # Check thresholds
    thresholds = params['evaluation']['threshold']
    
    issues = []
    if metrics['f1_score'] < thresholds['min_f1_score']:
        issues.append(
            f"F1 score {metrics['f1_score']:.3f} < threshold {thresholds['min_f1_score']}"
        )
    if metrics['roc_auc'] < thresholds['min_roc_auc']:
        issues.append(
            f"ROC AUC {metrics['roc_auc']:.3f} < threshold {thresholds['min_roc_auc']}"
        )
    
    if issues:
        logger.error("Model failed quality gates:")
        for issue in issues:
            logger.error(f"  - {issue}")
        raise ValueError("Model quality gates failed: " + "; ".join(issues))
    
    # Save metrics
    Path('metrics').mkdir(parents=True, exist_ok=True)
    with open('metrics/evaluation_metrics.json', 'w') as f:
        json.dump(metrics, f, indent=2)
    
    # Save confusion matrix for plotting
    cm = confusion_matrix(y_test, y_pred)
    cm_data = []
    for actual, row in enumerate(cm):
        for predicted, count in enumerate(row):
            cm_data.append({'actual': actual, 'predicted': predicted, 'count': count})
    
    pd.DataFrame(cm_data).to_csv('metrics/confusion_matrix.csv', index=False)
    
    # Save ROC curve data
    fpr, tpr, _ = roc_curve(y_test, y_pred_proba)
    pd.DataFrame({'fpr': fpr, 'tpr': tpr}).to_csv('metrics/roc_curve.csv', index=False)
    
    logger.info("Evaluation complete! All quality gates passed.")
    return metrics

if __name__ == '__main__':
    evaluate()
```

---

## 3. MLflow สำหรับ Model Tracking

### 3.1 MLflow Setup

```python
# mlflow_setup.py
import mlflow
from mlflow.tracking import MlflowClient

# ตั้งค่า MLflow Tracking Server
mlflow.set_tracking_uri("http://mlflow-server:5000")
mlflow.set_experiment("fraud-detection")

# สร้าง experiment ด้วย tags
client = MlflowClient()
experiment = client.get_experiment_by_name("fraud-detection")

if experiment is None:
    experiment_id = client.create_experiment(
        name="fraud-detection",
        artifact_location="s3://my-ml-artifacts/mlflow",
        tags={
            "team": "data-science",
            "project": "fraud-detection",
            "version": "1.0"
        }
    )
```

### 3.2 Model Registry และ Lifecycle

```python
# model_registry.py
import mlflow
from mlflow.tracking import MlflowClient
from mlflow.entities.model_registry import ModelVersion

client = MlflowClient()
MODEL_NAME = "fraud-detection-model"

def register_model(run_id: str, model_path: str = "model") -> ModelVersion:
    """Register model ใน Model Registry"""
    model_uri = f"runs:/{run_id}/{model_path}"
    
    # Register model
    model_version = mlflow.register_model(
        model_uri=model_uri,
        name=MODEL_NAME,
        tags={
            "git_commit": os.environ.get("GITHUB_SHA", "unknown"),
            "trained_by": os.environ.get("GITHUB_ACTOR", "unknown"),
            "training_date": datetime.now().isoformat()
        }
    )
    
    print(f"Registered model version: {model_version.version}")
    return model_version

def promote_to_staging(version: str) -> None:
    """Promote model to Staging"""
    client.transition_model_version_stage(
        name=MODEL_NAME,
        version=version,
        stage="Staging",
        archive_existing_versions=True
    )
    
    client.update_model_version(
        name=MODEL_NAME,
        version=version,
        description="Promoted to Staging after passing all quality gates"
    )
    
    print(f"Model version {version} promoted to Staging")

def promote_to_production(version: str) -> None:
    """Promote model to Production"""
    # Verify model is in Staging
    mv = client.get_model_version(MODEL_NAME, version)
    assert mv.current_stage == "Staging", f"Model must be in Staging, got {mv.current_stage}"
    
    client.transition_model_version_stage(
        name=MODEL_NAME,
        version=version,
        stage="Production",
        archive_existing_versions=True  # Archive previous production model
    )
    
    print(f"Model version {version} promoted to Production")

def get_production_model():
    """Load production model"""
    model_uri = f"models:/{MODEL_NAME}/Production"
    return mlflow.pyfunc.load_model(model_uri)

def compare_model_versions(version1: str, version2: str) -> dict:
    """เปรียบเทียบ metrics ระหว่างสอง versions"""
    def get_metrics(version: str) -> dict:
        mv = client.get_model_version(MODEL_NAME, version)
        run = client.get_run(mv.run_id)
        return run.data.metrics
    
    metrics_v1 = get_metrics(version1)
    metrics_v2 = get_metrics(version2)
    
    comparison = {}
    for metric in metrics_v1:
        if metric in metrics_v2:
            diff = metrics_v2[metric] - metrics_v1[metric]
            comparison[metric] = {
                'version1': metrics_v1[metric],
                'version2': metrics_v2[metric],
                'diff': diff,
                'improved': diff > 0
            }
    
    return comparison
```

---

## 4. GitHub Actions MLOps Pipeline

### 4.1 CI Pipeline (Unit Tests)

```yaml
# .github/workflows/ci.yml
name: ML CI Pipeline

on:
  push:
    branches: [main, develop]
    paths:
      - 'src/**'
      - 'tests/**'
      - 'requirements.txt'
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'
          cache: 'pip'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov pytest-mock
      
      - name: Run unit tests
        run: |
          pytest tests/unit/ \
            --cov=src \
            --cov-report=xml \
            --cov-report=term-missing \
            --cov-fail-under=80 \
            -v
      
      - name: Run code quality checks
        run: |
          pip install flake8 black isort mypy
          flake8 src/ --max-line-length 120
          black src/ --check
          isort src/ --check-only
          mypy src/ --ignore-missing-imports
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml

  data-validation:
    runs-on: ubuntu-latest
    needs: test
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure DVC remote
        run: |
          pip install dvc dvc-s3
          dvc remote modify myremote access_key_id ${{ secrets.AWS_ACCESS_KEY_ID }}
          dvc remote modify myremote secret_access_key ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      
      - name: Pull validation data sample
        run: dvc pull data/raw/transactions_sample.csv
      
      - name: Run data validation tests
        run: |
          pytest tests/data/ -v \
            --data-path=data/raw/transactions_sample.csv
```

### 4.2 Training Pipeline

```yaml
# .github/workflows/train.yml
name: ML Training Pipeline

on:
  push:
    branches: [main]
    paths:
      - 'src/**'
      - 'params.yaml'
      - 'dvc.yaml'
  schedule:
    - cron: '0 2 * * 1'  # ทุกวันจันทร์ 02:00 UTC - weekly retraining
  workflow_dispatch:
    inputs:
      force_retrain:
        description: 'Force retrain even if no changes'
        type: boolean
        default: false

jobs:
  train:
    runs-on: ubuntu-latest
    timeout-minutes: 120
    
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history สำหรับ DVC
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.10'
          cache: 'pip'
      
      - name: Install dependencies
        run: pip install -r requirements.txt
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ap-southeast-1
      
      - name: Setup DVC
        run: |
          pip install dvc dvc-s3
          dvc remote modify myremote access_key_id ${{ secrets.AWS_ACCESS_KEY_ID }}
          dvc remote modify myremote secret_access_key ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      
      - name: Pull data
        run: dvc pull data/
      
      - name: Run DVC pipeline
        env:
          MLFLOW_TRACKING_URI: ${{ secrets.MLFLOW_TRACKING_URI }}
          MLFLOW_TRACKING_USERNAME: ${{ secrets.MLFLOW_USER }}
          MLFLOW_TRACKING_PASSWORD: ${{ secrets.MLFLOW_PASSWORD }}
        run: |
          dvc repro --force=${{ github.event.inputs.force_retrain || 'false' }}
      
      - name: Check model quality gates
        id: quality_check
        run: |
          python scripts/check_quality_gates.py
          
          # Export metrics สำหรับ GitHub Actions
          F1=$(cat metrics/evaluation_metrics.json | python3 -c "import sys,json; print(json.load(sys.stdin)['f1_score'])")
          ROC=$(cat metrics/evaluation_metrics.json | python3 -c "import sys,json; print(json.load(sys.stdin)['roc_auc'])")
          
          echo "f1_score=$F1" >> $GITHUB_OUTPUT
          echo "roc_auc=$ROC" >> $GITHUB_OUTPUT
      
      - name: Push artifacts to DVC remote
        if: success()
        run: dvc push
      
      - name: Register model in MLflow
        if: success()
        env:
          MLFLOW_TRACKING_URI: ${{ secrets.MLFLOW_TRACKING_URI }}
        run: |
          python scripts/register_model.py \
            --run-id=$(cat metrics/run_id.txt) \
            --promote-to=Staging
      
      - name: Create GitHub Release for model
        if: success()
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: model-v${{ github.run_number }}
          release_name: Model v${{ github.run_number }}
          body: |
            ## Model Training Results
            
            - **F1 Score**: ${{ steps.quality_check.outputs.f1_score }}
            - **ROC AUC**: ${{ steps.quality_check.outputs.roc_auc }}
            - **Training Date**: ${{ github.event.head_commit.timestamp }}
            - **Commit**: ${{ github.sha }}
            
            See full metrics in MLflow: ${{ secrets.MLFLOW_TRACKING_URI }}
      
      - name: Notify Slack
        if: always()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Model training ${{ job.status }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*ML Training Pipeline* ${{ job.status }}\nF1: ${{ steps.quality_check.outputs.f1_score }}\nROC AUC: ${{ steps.quality_check.outputs.roc_auc }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

### 4.3 Deployment Pipeline

```yaml
# .github/workflows/deploy.yml
name: ML Model Deployment

on:
  workflow_run:
    workflows: ["ML Training Pipeline"]
    types: [completed]
  workflow_dispatch:
    inputs:
      model_version:
        description: 'MLflow model version to deploy'
        required: true
      deployment_strategy:
        description: 'Deployment strategy'
        type: choice
        options: [shadow, canary, blue_green]
        default: canary

jobs:
  validate-model:
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' || github.event_name == 'workflow_dispatch' }}
    
    outputs:
      model_version: ${{ steps.get_version.outputs.version }}
      approved: ${{ steps.approval_check.outputs.approved }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Get latest staging model version
        id: get_version
        env:
          MLFLOW_TRACKING_URI: ${{ secrets.MLFLOW_TRACKING_URI }}
        run: |
          VERSION=$(python scripts/get_model_version.py --stage=Staging)
          echo "version=$VERSION" >> $GITHUB_OUTPUT
      
      - name: Run A/B validation against production
        id: approval_check
        env:
          MLFLOW_TRACKING_URI: ${{ secrets.MLFLOW_TRACKING_URI }}
        run: |
          python scripts/compare_models.py \
            --challenger-version=${{ steps.get_version.outputs.version }} \
            --champion-stage=Production \
            --output=comparison_report.json
          
          # Check if challenger is better
          APPROVED=$(python3 -c "
          import json
          with open('comparison_report.json') as f:
              report = json.load(f)
          print('true' if report['challenger_better'] else 'false')
          ")
          echo "approved=$APPROVED" >> $GITHUB_OUTPUT
      
      - name: Upload comparison report
        uses: actions/upload-artifact@v4
        with:
          name: model-comparison-report
          path: comparison_report.json

  deploy-canary:
    needs: validate-model
    runs-on: ubuntu-latest
    if: needs.validate-model.outputs.approved == 'true'
    environment: staging
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy with Canary strategy
        env:
          MLFLOW_TRACKING_URI: ${{ secrets.MLFLOW_TRACKING_URI }}
        run: |
          python scripts/deploy_model.py \
            --version=${{ needs.validate-model.outputs.model_version }} \
            --strategy=canary \
            --canary-traffic=10 \
            --canary-duration=30m
      
      - name: Monitor canary deployment
        run: |
          python scripts/monitor_deployment.py \
            --duration=30m \
            --error-threshold=0.05 \
            --latency-threshold=200
      
      - name: Promote to full production
        if: success()
        run: |
          python scripts/deploy_model.py \
            --version=${{ needs.validate-model.outputs.model_version }} \
            --strategy=full_traffic
          
          # Update MLflow stage
          python scripts/promote_model.py \
            --version=${{ needs.validate-model.outputs.model_version }} \
            --to-stage=Production
```

---

## 5. Shadow Deployment สำหรับ ML Models

### 5.1 Shadow Mode ทำงานอย่างไร

```python
# serving/shadow_predictor.py
import logging
import asyncio
from typing import Optional
import numpy as np

logger = logging.getLogger(__name__)

class ShadowPredictor:
    """
    Shadow deployment: ส่ง request ไปทั้ง production และ shadow model
    แต่ return result จาก production เท่านั้น
    Collect shadow predictions เพื่อ comparison
    """
    
    def __init__(self, production_model, shadow_model, metrics_collector):
        self.production_model = production_model
        self.shadow_model = shadow_model
        self.metrics = metrics_collector
    
    def predict(self, features: np.ndarray) -> dict:
        """ทำ prediction ด้วย production model และ shadow ไปพร้อมกัน"""
        
        # Production prediction (blocking - return to user)
        prod_prediction = self._predict_production(features)
        
        # Shadow prediction (async - don't block user)
        asyncio.create_task(
            self._predict_shadow(features, prod_prediction)
        )
        
        return prod_prediction
    
    def _predict_production(self, features: np.ndarray) -> dict:
        """Production prediction พร้อม timing"""
        import time
        start = time.time()
        
        try:
            prediction = self.production_model.predict_proba(features)[0]
            latency_ms = (time.time() - start) * 1000
            
            self.metrics.record('production_latency', latency_ms)
            self.metrics.record('production_prediction', prediction)
            
            return {
                'probability': float(prediction[1]),
                'is_fraud': bool(prediction[1] > 0.5),
                'model': 'production'
            }
        except Exception as e:
            logger.error(f"Production prediction error: {e}")
            self.metrics.record('production_errors', 1)
            raise
    
    async def _predict_shadow(self, features: np.ndarray, prod_result: dict):
        """Shadow prediction - ไม่ block user request"""
        import time
        start = time.time()
        
        try:
            shadow_prediction = self.shadow_model.predict_proba(features)[0]
            latency_ms = (time.time() - start) * 1000
            
            self.metrics.record('shadow_latency', latency_ms)
            self.metrics.record('shadow_prediction', shadow_prediction)
            
            # Compare predictions
            prod_proba = prod_result['probability']
            shadow_proba = float(shadow_prediction[1])
            
            difference = abs(prod_proba - shadow_proba)
            
            if difference > 0.2:  # ความแตกต่างมากกว่า 20%
                logger.warning(
                    f"Large prediction difference: "
                    f"prod={prod_proba:.3f}, shadow={shadow_proba:.3f}"
                )
                self.metrics.record('prediction_disagreements', 1)
            
            # Log for analysis
            self.metrics.log_comparison({
                'production': prod_proba,
                'shadow': shadow_proba,
                'difference': difference,
                'prod_label': prod_result['is_fraud'],
                'shadow_label': bool(shadow_proba > 0.5)
            })
            
        except Exception as e:
            logger.error(f"Shadow prediction error: {e}")
```

---

## 6. Model Monitoring และ Drift Detection

### 6.1 Data Drift Detection

```python
# monitoring/drift_detector.py
import numpy as np
import pandas as pd
from scipy import stats
from typing import Dict, Tuple
import logging

logger = logging.getLogger(__name__)

class DataDriftDetector:
    """ตรวจจับ data drift ระหว่าง reference data กับ current data"""
    
    def __init__(self, reference_data: pd.DataFrame, features: list):
        self.reference_data = reference_data
        self.features = features
        self.baseline_stats = self._compute_baseline_stats()
    
    def _compute_baseline_stats(self) -> Dict:
        """คำนวณ statistics จาก reference data"""
        stats_dict = {}
        
        for feature in self.features:
            if self.reference_data[feature].dtype in ['int64', 'float64']:
                stats_dict[feature] = {
                    'type': 'numerical',
                    'mean': self.reference_data[feature].mean(),
                    'std': self.reference_data[feature].std(),
                    'min': self.reference_data[feature].min(),
                    'max': self.reference_data[feature].max(),
                    'distribution': self.reference_data[feature].values
                }
            else:
                value_counts = self.reference_data[feature].value_counts(normalize=True)
                stats_dict[feature] = {
                    'type': 'categorical',
                    'distribution': value_counts.to_dict()
                }
        
        return stats_dict
    
    def detect_drift(self, current_data: pd.DataFrame) -> Dict:
        """ตรวจจับ drift ใน current data"""
        drift_report = {
            'timestamp': pd.Timestamp.now().isoformat(),
            'total_features': len(self.features),
            'drifted_features': [],
            'drift_scores': {}
        }
        
        for feature in self.features:
            baseline = self.baseline_stats[feature]
            
            if baseline['type'] == 'numerical':
                # KS Test สำหรับ numerical features
                ks_stat, p_value = stats.ks_2samp(
                    baseline['distribution'],
                    current_data[feature].values
                )
                
                drift_score = ks_stat
                is_drifted = p_value < 0.05  # p-value threshold
                
                drift_report['drift_scores'][feature] = {
                    'type': 'ks_test',
                    'statistic': float(ks_stat),
                    'p_value': float(p_value),
                    'is_drifted': is_drifted,
                    'current_mean': float(current_data[feature].mean()),
                    'baseline_mean': float(baseline['mean'])
                }
                
            else:
                # Chi-square test สำหรับ categorical features
                current_dist = current_data[feature].value_counts(normalize=True)
                
                # Align distributions
                all_categories = set(baseline['distribution'].keys()) | set(current_dist.index)
                
                baseline_probs = [baseline['distribution'].get(cat, 0) for cat in all_categories]
                current_probs = [current_dist.get(cat, 0) for cat in all_categories]
                
                # PSI (Population Stability Index)
                psi = self._calculate_psi(baseline_probs, current_probs)
                is_drifted = psi > 0.2  # PSI threshold
                
                drift_report['drift_scores'][feature] = {
                    'type': 'psi',
                    'psi_value': float(psi),
                    'is_drifted': is_drifted
                }
            
            if is_drifted:
                drift_report['drifted_features'].append(feature)
        
        drift_report['drift_ratio'] = (
            len(drift_report['drifted_features']) / len(self.features)
        )
        
        if drift_report['drift_ratio'] > 0.3:
            drift_report['alert_level'] = 'HIGH'
            logger.warning(f"High data drift detected! {len(drift_report['drifted_features'])} features drifted")
        elif drift_report['drift_ratio'] > 0.1:
            drift_report['alert_level'] = 'MEDIUM'
            logger.info(f"Medium data drift detected: {len(drift_report['drifted_features'])} features")
        else:
            drift_report['alert_level'] = 'LOW'
        
        return drift_report
    
    def _calculate_psi(self, baseline: list, current: list) -> float:
        """Calculate Population Stability Index"""
        psi = 0
        for b, c in zip(baseline, current):
            if b > 0 and c > 0:
                psi += (c - b) * np.log(c / b)
        return psi

class ModelPerformanceMonitor:
    """Monitor model performance ใน production"""
    
    def __init__(self, model_name: str, metrics_store):
        self.model_name = model_name
        self.metrics_store = metrics_store
    
    def log_prediction(self, features, prediction, actual=None):
        """Log prediction และ actual label (ถ้ามี)"""
        record = {
            'timestamp': pd.Timestamp.now().isoformat(),
            'model_name': self.model_name,
            'prediction': float(prediction),
            'actual': actual
        }
        self.metrics_store.append(record)
    
    def calculate_performance_metrics(self, window_hours: int = 24) -> Dict:
        """คำนวณ performance metrics จาก predictions ที่มี labels"""
        recent_data = self.metrics_store.get_recent(hours=window_hours)
        
        # Filter records ที่มี actual labels
        labeled_data = [r for r in recent_data if r.get('actual') is not None]
        
        if len(labeled_data) < 100:
            return {'status': 'insufficient_data', 'count': len(labeled_data)}
        
        predictions = [r['prediction'] for r in labeled_data]
        actuals = [r['actual'] for r in labeled_data]
        
        binary_preds = [1 if p > 0.5 else 0 for p in predictions]
        
        from sklearn.metrics import f1_score, roc_auc_score
        
        return {
            'f1_score': float(f1_score(actuals, binary_preds)),
            'roc_auc': float(roc_auc_score(actuals, predictions)),
            'sample_count': len(labeled_data),
            'window_hours': window_hours
        }
```

---

## 7. Automated Retraining Pipeline

### 7.1 Retraining Trigger Logic

```python
# scripts/check_retraining_needed.py
import json
import logging
from datetime import datetime, timedelta
import sys

logger = logging.getLogger(__name__)

class RetrainingDecisionEngine:
    
    def __init__(self, config: dict):
        self.config = config
    
    def should_retrain(self) -> Tuple[bool, str]:
        """
        Returns: (should_retrain: bool, reason: str)
        """
        checks = [
            self._check_performance_degradation(),
            self._check_data_drift(),
            self._check_time_since_last_training(),
            self._check_new_data_volume()
        ]
        
        for should_retrain, reason in checks:
            if should_retrain:
                return True, reason
        
        return False, "No retraining needed"
    
    def _check_performance_degradation(self) -> Tuple[bool, str]:
        """ตรวจสอบ model performance ลดลงหรือไม่"""
        try:
            with open('metrics/production_performance.json') as f:
                current_metrics = json.load(f)
            
            with open('metrics/baseline_metrics.json') as f:
                baseline_metrics = json.load(f)
            
            f1_degradation = (
                baseline_metrics['f1_score'] - current_metrics['f1_score']
            )
            
            threshold = self.config.get('performance_degradation_threshold', 0.05)
            
            if f1_degradation > threshold:
                return True, f"F1 score degraded by {f1_degradation:.3f} (threshold: {threshold})"
            
        except FileNotFoundError:
            pass
        
        return False, ""
    
    def _check_data_drift(self) -> Tuple[bool, str]:
        """ตรวจสอบ data drift"""
        try:
            with open('metrics/drift_report.json') as f:
                drift_report = json.load(f)
            
            if drift_report.get('alert_level') == 'HIGH':
                return True, f"High data drift: {drift_report['drift_ratio']:.1%} features drifted"
            
        except FileNotFoundError:
            pass
        
        return False, ""
    
    def _check_time_since_last_training(self) -> Tuple[bool, str]:
        """ตรวจสอบว่า train นานเกินไปหรือไม่"""
        max_days = self.config.get('max_days_without_retraining', 30)
        
        try:
            with open('models/trained/metadata.json') as f:
                metadata = json.load(f)
            
            training_date = datetime.fromisoformat(metadata['training_date'])
            days_since_training = (datetime.now() - training_date).days
            
            if days_since_training > max_days:
                return True, f"Last training was {days_since_training} days ago (max: {max_days})"
            
        except (FileNotFoundError, KeyError):
            return True, "No training metadata found"
        
        return False, ""

if __name__ == '__main__':
    config = {
        'performance_degradation_threshold': 0.05,
        'max_days_without_retraining': 30,
        'min_new_data_volume': 1000
    }
    
    engine = RetrainingDecisionEngine(config)
    should_retrain, reason = engine.should_retrain()
    
    print(f"Should retrain: {should_retrain}")
    print(f"Reason: {reason}")
    
    # Export สำหรับ GitHub Actions
    with open(os.environ.get('GITHUB_OUTPUT', '/dev/null'), 'a') as f:
        f.write(f"should_retrain={str(should_retrain).lower()}\n")
        f.write(f"reason={reason}\n")
    
    sys.exit(0 if not should_retrain else 0)  # Always exit 0, use output variable
```

---

## 8. Unit Tests สำหรับ ML Code

### 8.1 Testing Feature Engineering

```python
# tests/unit/test_feature_engineering.py
import pytest
import pandas as pd
import numpy as np
from src.features.engineer import FeatureEngineer

@pytest.fixture
def sample_transactions():
    return pd.DataFrame({
        'transaction_id': range(100),
        'user_id': np.random.choice(['user1', 'user2', 'user3'], 100),
        'amount': np.random.exponential(100, 100),
        'timestamp': pd.date_range('2024-01-01', periods=100, freq='H'),
        'merchant_category': np.random.choice(['food', 'retail', 'travel'], 100),
        'is_fraud': np.random.binomial(1, 0.1, 100)
    })

class TestFeatureEngineer:
    
    def test_creates_velocity_features(self, sample_transactions):
        """ทดสอบว่า velocity features ถูกสร้างอย่างถูกต้อง"""
        engineer = FeatureEngineer()
        result = engineer.create_velocity_features(sample_transactions)
        
        assert 'tx_count_1h' in result.columns
        assert 'tx_count_24h' in result.columns
        assert 'amount_sum_24h' in result.columns
        
        # Values should be non-negative
        assert (result['tx_count_1h'] >= 0).all()
        assert (result['tx_count_24h'] >= 0).all()
    
    def test_handles_missing_values(self, sample_transactions):
        """ทดสอบว่าจัดการ missing values ได้"""
        # เพิ่ม missing values
        sample_transactions.loc[0:10, 'amount'] = np.nan
        
        engineer = FeatureEngineer()
        result = engineer.transform(sample_transactions)
        
        assert result.isnull().sum().sum() == 0, "Should have no null values after transform"
    
    def test_categorical_encoding(self, sample_transactions):
        """ทดสอบ categorical encoding"""
        engineer = FeatureEngineer()
        result = engineer.encode_categoricals(sample_transactions)
        
        # merchant_category should be encoded
        assert result['merchant_category'].dtype != object
        
        # Check encoded values are valid
        unique_vals = result['merchant_category'].unique()
        assert len(unique_vals) <= 3  # มีแค่ 3 categories
    
    def test_feature_scaling(self, sample_transactions):
        """ทดสอบ feature scaling"""
        engineer = FeatureEngineer()
        engineer.fit(sample_transactions)
        result = engineer.transform(sample_transactions)
        
        # Scaled features should be roughly in [-3, 3] range
        assert result['amount'].abs().max() < 10, "Amount should be scaled"
    
    def test_reproducibility(self, sample_transactions):
        """ทดสอบว่า transform ให้ผลเหมือนกันทุกครั้ง"""
        engineer = FeatureEngineer(random_state=42)
        
        result1 = engineer.transform(sample_transactions.copy())
        result2 = engineer.transform(sample_transactions.copy())
        
        pd.testing.assert_frame_equal(result1, result2)

class TestModelTraining:
    
    def test_model_trains_without_error(self):
        """ทดสอบว่า model train ได้โดยไม่มี error"""
        from src.training.train import train
        import yaml
        
        # Use small test params
        params = {
            'data': {'random_state': 42},
            'features': {
                'numerical_features': ['amount'],
                'categorical_features': ['merchant_category'],
                'target_column': 'is_fraud'
            },
            'training': {
                'model_type': 'xgboost',
                'n_estimators': 10,
                'max_depth': 3,
                'learning_rate': 0.1,
                'subsample': 0.8
            }
        }
        
        # Train with mock data
        model = train(params, use_mock_data=True)
        
        assert model is not None
        assert hasattr(model, 'predict')
        assert hasattr(model, 'predict_proba')
    
    def test_model_passes_quality_gates(self):
        """ทดสอบ quality gates"""
        from src.training.evaluate import evaluate_mock
        
        metrics = evaluate_mock()
        
        assert metrics['f1_score'] >= 0.85
        assert metrics['roc_auc'] >= 0.90
```

---

## 9. แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Build Complete MLOps Pipeline

```
Task: สร้าง end-to-end MLOps pipeline สำหรับ customer churn prediction

Requirements:
1. Dataset: Telco Customer Churn dataset (Kaggle)
2. DVC สำหรับ data versioning
3. MLflow สำหรับ experiment tracking
4. GitHub Actions สำหรับ automated training
5. Model quality gates:
   - Accuracy >= 80%
   - F1 Score >= 0.75
   - ROC AUC >= 0.85
6. Automated retraining เมื่อ:
   - New data volume > 1000 records
   - Model performance ลดลง > 5%

Deliverables:
- dvc.yaml pipeline
- params.yaml
- GitHub Actions workflows
- Unit tests สำหรับ feature engineering
- MLflow experiment dashboard
```

### แบบฝึกหัดที่ 2: Implement Shadow Deployment

```python
# TODO: Implement shadow deployment

class ShadowDeploymentTest:
    """
    สร้าง shadow deployment test scenario:
    
    1. Load production model (v1) และ challenger model (v2)
    2. Generate 1000 test requests
    3. Compare predictions between models
    4. Generate comparison report:
       - Agreement rate
       - Cases where models disagree
       - Performance difference (ถ้ามี labels)
    5. Decide whether to promote challenger to production
    """
    pass
```

### แบบฝึกหัดที่ 3: Drift Detection Alert

```
สร้าง drift detection system ที่:
1. Calculate KS test สำหรับ numerical features
2. Calculate PSI สำหรับ categorical features  
3. ส่ง alert ไปยัง Slack เมื่อ drift level = HIGH
4. Trigger automated retraining เมื่อ drift ratio > 30%
5. สร้าง dashboard แสดง drift metrics over time
```

### สรุปบทที่ 62

ในบทนี้เราได้เรียนรู้:
- **ML Workflow**: ความแตกต่างระหว่าง Traditional CI/CD และ MLOps
- **DVC**: Version control สำหรับ data และ ML artifacts
- **MLflow**: Experiment tracking และ model registry
- **GitHub Actions**: Automated training และ deployment pipelines
- **Shadow Deployment**: วิธีทดสอบ model ใหม่ใน production อย่างปลอดภัย
- **Model Monitoring**: Data drift detection และ performance monitoring
- **Automated Retraining**: Logic สำหรับตัดสินใจว่าต้อง retrain หรือไม่

บทถัดไปเราจะเรียนรู้ DataOps และ Data Pipeline CI/CD
