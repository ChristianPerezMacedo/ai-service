# SageMaker MLOps Pipeline

## Overview

Amazon SageMaker è la piattaforma AWS per il machine learning end-to-end, che fornisce strumenti per costruire, addestrare, distribuire e monitorare modelli ML in produzione.

### Perché SageMaker nel nostro contesto

Nel **AI Technical Support System**, utilizziamo SageMaker per:

1. **Text Classification**: Classificazione automatica dei ticket in categorie tecniche
2. **Sentiment Analysis**: Analisi del tono delle richieste dei clienti
3. **Entity Recognition**: Estrazione di informazioni chiave dai ticket (prodotti, errori, versioni)
4. **Model Retraining**: Miglioramento continuo basato su feedback umano

### Quando usare SageMaker

- **Custom models** che richiedono training su dati proprietari
- **Production ML** con requisiti di scalabilità e affidabilità
- **MLOps automation** per iterazione rapida e continuous learning
- **Compliance** quando serve tracciabilità completa del ciclo di vita del modello

### Architettura High-Level

```
┌─────────────────────────────────────────────────────────────────┐
│                     SageMaker MLOps Pipeline                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  Data Sources        Pipeline Stages           Deployment        │
│  ┌──────────┐       ┌──────────────┐         ┌──────────┐      │
│  │ S3       │──────▶│ Processing   │         │ Endpoint │      │
│  │ DynamoDB │       │   Step       │         │ (RT/BT)  │      │
│  └──────────┘       └──────┬───────┘         └────▲─────┘      │
│                             │                      │             │
│                     ┌───────▼───────┐             │             │
│                     │  Training     │             │             │
│                     │    Step       │             │             │
│                     └───────┬───────┘             │             │
│                             │                      │             │
│                     ┌───────▼───────┐             │             │
│                     │ Evaluation    │             │             │
│                     │    Step       │             │             │
│                     └───────┬───────┘             │             │
│                             │                      │             │
│                     ┌───────▼───────┐             │             │
│                     │ Model         │             │             │
│                     │ Registry      │─────────────┘             │
│                     └───────┬───────┘                           │
│                             │                                    │
│                     ┌───────▼───────┐                           │
│                     │ Monitoring    │                           │
│                     │ (Drift)       │                           │
│                     └───────────────┘                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## Concetti Fondamentali

### SageMaker Pipelines

**SageMaker Pipelines** è il servizio per orchestrare workflow ML, simile a Step Functions ma specializzato per ML.

**Componenti chiave**:
- **Pipeline**: Definizione completa del workflow
- **Steps**: Unità atomiche di lavoro (Processing, Training, Evaluation, etc.)
- **Parameters**: Input dinamici al pipeline
- **Properties**: Output di uno step usati come input per altri

**Step types**:

| Step Type | Uso | Esempio |
|-----------|-----|---------|
| `ProcessingStep` | Data preprocessing, feature engineering | Tokenizzazione testi, bilanciamento dataset |
| `TrainingStep` | Model training | Training BlazingText classifier |
| `TuningStep` | Hyperparameter optimization | Grid search su learning rate |
| `EvaluationStep` | Model evaluation | Calcolo F1-score su test set |
| `ConditionStep` | Branching logic | Deploy solo se accuracy > 0.85 |
| `RegisterModelStep` | Model Registry registration | Versioning del modello |
| `TransformStep` | Batch transform | Inferenza su grandi dataset |
| `CreateModelStep` | Model creation | Preparazione per deployment |
| `LambdaStep` | Custom logic | Notifiche, validazioni custom |

### Model Registry

**Model Registry** è il catalogo centralizzato per i modelli ML.

**Caratteristiche**:
- **Versioning**: Tracking automatico delle versioni del modello
- **Approval workflow**: Stati (Pending, Approved, Rejected)
- **Metadata**: Metriche, hyperparameters, lineage
- **Deployment tracking**: Quali versioni sono in produzione

**Model Package Groups**: Collezioni logiche di versioni del modello (es. "ticket-classifier")

### Training Jobs

**Training jobs** eseguono il codice di training su compute instances gestite.

**Instance types**:

| Tipo | vCPU | GPU | Memoria | Uso |
|------|------|-----|---------|-----|
| `ml.m5.xlarge` | 4 | 0 | 16 GB | Training small models, CPU-only |
| `ml.p3.2xlarge` | 8 | 1 V100 | 61 GB | Deep learning, single GPU |
| `ml.p3.8xlarge` | 32 | 4 V100 | 244 GB | Distributed training |
| `ml.g4dn.xlarge` | 4 | 1 T4 | 16 GB | Cost-effective inference optimization |

**Spot instances**: Risparmio fino al 90% per training interrompibili

### Hyperparameter Tuning

**HPO (Hyperparameter Optimization)** esegue ricerca automatica dei migliori hyperparameters.

**Strategie**:
- **Random Search**: Sampling casuale dello spazio
- **Bayesian Optimization**: Ottimizzazione intelligente (default)
- **Grid Search**: Exhaustive search (non disponibile nativamente)
- **Hyperband**: Early stopping per configurazioni non promettenti

### Model Deployment

**Deployment options**:

| Tipo | Latency | Throughput | Costo | Uso |
|------|---------|------------|-------|-----|
| **Real-time Endpoint** | ms | Alto | $$ | API real-time |
| **Serverless Endpoint** | ms-s | Variabile | $ (pay-per-use) | Traffic intermittente |
| **Batch Transform** | minuti | Batch | $ | Inferenza offline |
| **Async Endpoint** | secondi | Alto | $$ | Long-running inference |

### Model Monitoring

**Model Monitor** traccia la qualità del modello in produzione.

**Tipi di monitoring**:

1. **Data Quality**: Drift nello schema/distribuzione degli input
2. **Model Quality**: Degradazione delle metriche (accuracy, F1)
3. **Bias Drift**: Cambiamenti nel bias del modello
4. **Feature Attribution Drift**: Importanza delle feature

---

## Implementazione Pratica

### 1. Training Pipeline Completo

Pipeline per training del ticket classifier:

```python
"""
SageMaker Pipeline per Text Classification
File: ml/pipelines/ticket_classifier_pipeline.py
"""

import boto3
from sagemaker.workflow.pipeline import Pipeline
from sagemaker.workflow.parameters import (
    ParameterInteger,
    ParameterString,
    ParameterFloat
)
from sagemaker.workflow.steps import (
    ProcessingStep,
    TrainingStep,
    CreateModelStep,
    TransformStep
)
from sagemaker.workflow.properties import PropertyFile
from sagemaker.workflow.conditions import ConditionGreaterThanOrEqualTo
from sagemaker.workflow.condition_step import ConditionStep
from sagemaker.workflow.functions import JsonGet
from sagemaker.sklearn.processing import SKLearnProcessor
from sagemaker.processing import ProcessingInput, ProcessingOutput
from sagemaker.estimator import Estimator
from sagemaker.inputs import TrainingInput
from sagemaker.model import Model
from sagemaker.transformer import Transformer
from sagemaker import get_execution_role

# Configurazione
REGION = "us-east-1"
ROLE = get_execution_role()
BUCKET = "ai-support-ml-artifacts"

# Pipeline Parameters - consentono di riutilizzare il pipeline con input diversi
processing_instance_type = ParameterString(
    name="ProcessingInstanceType",
    default_value="ml.m5.xlarge"
)

training_instance_type = ParameterString(
    name="TrainingInstanceType",
    default_value="ml.p3.2xlarge"
)

training_instance_count = ParameterInteger(
    name="TrainingInstanceCount",
    default_value=1
)

model_approval_status = ParameterString(
    name="ModelApprovalStatus",
    default_value="PendingManualApproval"
)

input_data = ParameterString(
    name="InputDataUrl",
    default_value=f"s3://{BUCKET}/training-data/tickets.jsonl"
)

# Threshold per approvazione automatica
accuracy_threshold = ParameterFloat(
    name="AccuracyThreshold",
    default_value=0.85
)


def create_preprocessing_step():
    """
    Step 1: Data Preprocessing
    - Carica ticket raw da DynamoDB/S3
    - Tokenizzazione e cleaning
    - Split train/validation/test
    - Feature engineering
    """

    sklearn_processor = SKLearnProcessor(
        framework_version="1.2-1",
        instance_type=processing_instance_type,
        instance_count=1,
        role=ROLE,
        base_job_name="ticket-preprocessing"
    )

    # Script di preprocessing
    processing_step = ProcessingStep(
        name="PreprocessTicketData",
        processor=sklearn_processor,
        inputs=[
            ProcessingInput(
                source=input_data,
                destination="/opt/ml/processing/input"
            )
        ],
        outputs=[
            ProcessingOutput(
                output_name="train",
                source="/opt/ml/processing/train",
                destination=f"s3://{BUCKET}/processed/train"
            ),
            ProcessingOutput(
                output_name="validation",
                source="/opt/ml/processing/validation",
                destination=f"s3://{BUCKET}/processed/validation"
            ),
            ProcessingOutput(
                output_name="test",
                source="/opt/ml/processing/test",
                destination=f"s3://{BUCKET}/processed/test"
            )
        ],
        code="preprocessing.py",  # Script custom
        job_arguments=[
            "--min-tokens", "10",
            "--max-tokens", "512",
            "--balance-classes", "true"
        ]
    )

    return processing_step


def create_training_step(preprocessing_step):
    """
    Step 2: Model Training
    - Training BlazingText per text classification
    - Multi-class classification (10 categorie)
    """

    # BlazingText è un algoritmo ottimizzato per text classification
    training_image = sagemaker.image_uris.retrieve(
        framework="blazingtext",
        region=REGION,
        version="latest"
    )

    estimator = Estimator(
        image_uri=training_image,
        role=ROLE,
        instance_type=training_instance_type,
        instance_count=training_instance_count,
        output_path=f"s3://{BUCKET}/models",
        base_job_name="ticket-classifier-training",
        hyperparameters={
            "mode": "supervised",  # Supervised learning
            "epochs": 25,
            "learning_rate": 0.05,
            "word_ngrams": 2,  # Considera bigrammi
            "vector_dim": 100,  # Dimensione embedding
            "min_count": 5,     # Ignora parole rare
            "early_stopping": True,
            "patience": 3
        },
        metric_definitions=[
            {"Name": "train:accuracy", "Regex": "train_accuracy: ([0-9.]+)"},
            {"Name": "validation:accuracy", "Regex": "validation_accuracy: ([0-9.]+)"}
        ],
        enable_sagemaker_metrics=True,
        # Usa spot instances per risparmiare (fino a 90%)
        use_spot_instances=True,
        max_wait=7200,  # 2 ore max
        max_run=3600    # 1 ora max training
    )

    training_step = TrainingStep(
        name="TrainTicketClassifier",
        estimator=estimator,
        inputs={
            "train": TrainingInput(
                s3_data=preprocessing_step.properties.ProcessingOutputConfig.Outputs["train"].S3Output.S3Uri,
                content_type="text/plain"
            ),
            "validation": TrainingInput(
                s3_data=preprocessing_step.properties.ProcessingOutputConfig.Outputs["validation"].S3Output.S3Uri,
                content_type="text/plain"
            )
        }
    )

    return training_step


def create_evaluation_step(training_step, preprocessing_step):
    """
    Step 3: Model Evaluation
    - Valuta performance su test set
    - Genera report con metriche (accuracy, F1, confusion matrix)
    - Salva evaluation.json per conditional step
    """

    sklearn_processor = SKLearnProcessor(
        framework_version="1.2-1",
        instance_type="ml.m5.xlarge",
        instance_count=1,
        role=ROLE,
        base_job_name="ticket-evaluation"
    )

    # PropertyFile per estrarre metriche
    evaluation_report = PropertyFile(
        name="TicketClassifierEvaluationReport",
        output_name="evaluation",
        path="evaluation.json"
    )

    evaluation_step = ProcessingStep(
        name="EvaluateTicketClassifier",
        processor=sklearn_processor,
        inputs=[
            ProcessingInput(
                source=training_step.properties.ModelArtifacts.S3ModelArtifacts,
                destination="/opt/ml/processing/model"
            ),
            ProcessingInput(
                source=preprocessing_step.properties.ProcessingOutputConfig.Outputs["test"].S3Output.S3Uri,
                destination="/opt/ml/processing/test"
            )
        ],
        outputs=[
            ProcessingOutput(
                output_name="evaluation",
                source="/opt/ml/processing/evaluation",
                destination=f"s3://{BUCKET}/evaluation"
            )
        ],
        code="evaluation.py",
        property_files=[evaluation_report]
    )

    return evaluation_step, evaluation_report


def create_model_registration_step(training_step, evaluation_step):
    """
    Step 4: Model Registration
    - Registra modello nel Model Registry
    - Versioning automatico
    - Metadata: metriche, hyperparameters, lineage
    """

    from sagemaker.model_metrics import MetricsSource, ModelMetrics
    from sagemaker.workflow.step_collections import RegisterModel

    # Metriche da associare al modello
    model_metrics = ModelMetrics(
        model_statistics=MetricsSource(
            s3_uri=f"{evaluation_step.properties.ProcessingOutputConfig.Outputs['evaluation'].S3Output.S3Uri}/evaluation.json",
            content_type="application/json"
        )
    )

    register_step = RegisterModel(
        name="RegisterTicketClassifier",
        estimator=training_step.estimator,
        model_data=training_step.properties.ModelArtifacts.S3ModelArtifacts,
        content_types=["text/plain"],
        response_types=["application/json"],
        inference_instances=["ml.t2.medium", "ml.m5.large"],
        transform_instances=["ml.m5.xlarge"],
        model_package_group_name="ticket-classifier-models",
        approval_status=model_approval_status,
        model_metrics=model_metrics
    )

    return register_step


def create_conditional_registration_step(evaluation_step, evaluation_report, register_step):
    """
    Step 5: Conditional Registration
    - Registra modello SOLO se accuracy >= threshold
    - Approval automatico se supera threshold
    """

    # Condizione: accuracy >= 0.85
    cond_gte = ConditionGreaterThanOrEqualTo(
        left=JsonGet(
            step_name=evaluation_step.name,
            property_file=evaluation_report,
            json_path="metrics.accuracy"
        ),
        right=accuracy_threshold
    )

    condition_step = ConditionStep(
        name="CheckAccuracyThreshold",
        conditions=[cond_gte],
        if_steps=[register_step],
        else_steps=[]  # Non fa nulla se accuracy bassa
    )

    return condition_step


def create_pipeline():
    """
    Costruisce il pipeline completo
    """

    # Step 1: Preprocessing
    preprocessing_step = create_preprocessing_step()

    # Step 2: Training
    training_step = create_training_step(preprocessing_step)

    # Step 3: Evaluation
    evaluation_step, evaluation_report = create_evaluation_step(
        training_step,
        preprocessing_step
    )

    # Step 4: Model Registration
    register_step = create_model_registration_step(
        training_step,
        evaluation_step
    )

    # Step 5: Conditional Registration
    condition_step = create_conditional_registration_step(
        evaluation_step,
        evaluation_report,
        register_step
    )

    # Pipeline definition
    pipeline = Pipeline(
        name="TicketClassifierTrainingPipeline",
        parameters=[
            processing_instance_type,
            training_instance_type,
            training_instance_count,
            model_approval_status,
            input_data,
            accuracy_threshold
        ],
        steps=[
            preprocessing_step,
            training_step,
            evaluation_step,
            condition_step
        ],
        sagemaker_session=sagemaker.Session()
    )

    return pipeline


# Deployment del pipeline
if __name__ == "__main__":
    pipeline = create_pipeline()

    # Crea o aggiorna il pipeline
    pipeline.upsert(role_arn=ROLE)

    print(f"Pipeline ARN: {pipeline.describe()['PipelineArn']}")

    # Esegui il pipeline
    execution = pipeline.start(
        parameters={
            "InputDataUrl": f"s3://{BUCKET}/training-data/tickets-2025-01.jsonl",
            "AccuracyThreshold": 0.87
        }
    )

    print(f"Execution ARN: {execution.arn}")

    # Attendi completamento (optional)
    execution.wait()
    print(f"Execution status: {execution.describe()['PipelineExecutionStatus']}")
```

### 2. Script di Preprocessing

```python
"""
Preprocessing script per ticket data
File: ml/scripts/preprocessing.py
"""

import argparse
import json
import os
from pathlib import Path
from typing import List, Dict, Tuple
import pandas as pd
import numpy as np
from sklearn.model_selection import train_test_split
from collections import Counter
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


def load_tickets(input_path: str) -> pd.DataFrame:
    """
    Carica ticket da JSONL format

    Expected format:
    {"ticket_id": "T-001", "content": "...", "category": "networking", "priority": "high"}
    """
    tickets = []

    with open(input_path, 'r', encoding='utf-8') as f:
        for line in f:
            ticket = json.loads(line)
            tickets.append(ticket)

    df = pd.DataFrame(tickets)
    logger.info(f"Loaded {len(df)} tickets")

    return df


def clean_text(text: str) -> str:
    """
    Pulizia del testo
    - Lowercase
    - Rimuove caratteri speciali
    - Rimuove whitespace multipli
    """
    import re

    # Lowercase
    text = text.lower()

    # Mantieni solo alphanumerici e spazi
    text = re.sub(r'[^a-z0-9\s]', ' ', text)

    # Rimuovi whitespace multipli
    text = re.sub(r'\s+', ' ', text).strip()

    return text


def balance_dataset(df: pd.DataFrame, target_col: str = 'category') -> pd.DataFrame:
    """
    Bilanciamento delle classi tramite undersampling/oversampling
    """
    class_counts = df[target_col].value_counts()
    logger.info(f"Original class distribution:\n{class_counts}")

    # Trova la classe con numero medio di samples
    median_count = int(class_counts.median())

    balanced_dfs = []
    for category in df[target_col].unique():
        category_df = df[df[target_col] == category]

        if len(category_df) > median_count:
            # Undersample
            category_df = category_df.sample(n=median_count, random_state=42)
        elif len(category_df) < median_count:
            # Oversample (con replacement)
            category_df = category_df.sample(n=median_count, replace=True, random_state=42)

        balanced_dfs.append(category_df)

    balanced_df = pd.concat(balanced_dfs, ignore_index=True).sample(frac=1, random_state=42)

    logger.info(f"Balanced class distribution:\n{balanced_df[target_col].value_counts()}")

    return balanced_df


def prepare_blazingtext_format(df: pd.DataFrame, output_path: str, label_col: str = 'category'):
    """
    Prepara dati in formato BlazingText

    Format: __label__<category> <text content>
    Example: __label__networking router not responding dhcp issue
    """
    with open(output_path, 'w', encoding='utf-8') as f:
        for _, row in df.iterrows():
            label = f"__label__{row[label_col]}"
            text = clean_text(row['content'])

            # Skip empty texts
            if not text:
                continue

            f.write(f"{label} {text}\n")

    logger.info(f"Saved {len(df)} samples to {output_path}")


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--min-tokens", type=int, default=10)
    parser.add_argument("--max-tokens", type=int, default=512)
    parser.add_argument("--balance-classes", type=str, default="true")
    args = parser.parse_args()

    # Paths
    input_dir = Path("/opt/ml/processing/input")
    output_dir = Path("/opt/ml/processing")

    # Load data
    df = load_tickets(input_dir / "tickets.jsonl")

    # Filter by token length
    df['token_count'] = df['content'].str.split().str.len()
    df = df[
        (df['token_count'] >= args.min_tokens) &
        (df['token_count'] <= args.max_tokens)
    ]
    logger.info(f"After filtering: {len(df)} tickets")

    # Balance classes
    if args.balance_classes.lower() == "true":
        df = balance_dataset(df)

    # Split: 70% train, 15% validation, 15% test
    train_df, temp_df = train_test_split(df, test_size=0.3, random_state=42, stratify=df['category'])
    val_df, test_df = train_test_split(temp_df, test_size=0.5, random_state=42, stratify=temp_df['category'])

    logger.info(f"Split sizes - Train: {len(train_df)}, Val: {len(val_df)}, Test: {len(test_df)}")

    # Save in BlazingText format
    output_dir.mkdir(parents=True, exist_ok=True)
    prepare_blazingtext_format(train_df, output_dir / "train" / "train.txt")
    prepare_blazingtext_format(val_df, output_dir / "validation" / "validation.txt")
    prepare_blazingtext_format(test_df, output_dir / "test" / "test.txt")

    logger.info("Preprocessing completed successfully")


if __name__ == "__main__":
    main()
```

### 3. Script di Evaluation

```python
"""
Model evaluation script
File: ml/scripts/evaluation.py
"""

import json
import tarfile
import logging
from pathlib import Path
import numpy as np
from sklearn.metrics import (
    accuracy_score,
    precision_recall_fscore_support,
    confusion_matrix,
    classification_report
)

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


def load_model(model_path: str):
    """
    Estrae e carica il modello BlazingText
    """
    # Estrai model.tar.gz
    with tarfile.open(model_path, 'r:gz') as tar:
        tar.extractall('/tmp/model')

    # Il modello BlazingText è in formato fastText
    import fasttext
    model = fasttext.load_model('/tmp/model/model.bin')

    return model


def load_test_data(test_path: str):
    """
    Carica test set in formato BlazingText
    """
    texts = []
    labels = []

    with open(test_path, 'r', encoding='utf-8') as f:
        for line in f:
            parts = line.strip().split(' ', 1)
            if len(parts) == 2:
                label = parts[0].replace('__label__', '')
                text = parts[1]
                labels.append(label)
                texts.append(text)

    return texts, labels


def evaluate_model(model, texts: list, true_labels: list):
    """
    Valuta il modello sul test set
    """
    predictions = []

    for text in texts:
        # Predizione
        pred_label, confidence = model.predict(text, k=1)
        pred_label = pred_label[0].replace('__label__', '')
        predictions.append(pred_label)

    # Calcola metriche
    accuracy = accuracy_score(true_labels, predictions)
    precision, recall, f1, support = precision_recall_fscore_support(
        true_labels,
        predictions,
        average='weighted'
    )

    # Confusion matrix
    cm = confusion_matrix(true_labels, predictions)

    # Classification report per classe
    report = classification_report(true_labels, predictions, output_dict=True)

    results = {
        "metrics": {
            "accuracy": float(accuracy),
            "precision": float(precision),
            "recall": float(recall),
            "f1_score": float(f1)
        },
        "confusion_matrix": cm.tolist(),
        "classification_report": report
    }

    logger.info(f"Evaluation Results:")
    logger.info(f"  Accuracy:  {accuracy:.4f}")
    logger.info(f"  Precision: {precision:.4f}")
    logger.info(f"  Recall:    {recall:.4f}")
    logger.info(f"  F1-Score:  {f1:.4f}")

    return results


def main():
    # Paths
    model_path = "/opt/ml/processing/model/model.tar.gz"
    test_path = "/opt/ml/processing/test/test.txt"
    output_dir = Path("/opt/ml/processing/evaluation")
    output_dir.mkdir(parents=True, exist_ok=True)

    # Load model
    logger.info("Loading model...")
    model = load_model(model_path)

    # Load test data
    logger.info("Loading test data...")
    texts, labels = load_test_data(test_path)
    logger.info(f"Test samples: {len(texts)}")

    # Evaluate
    logger.info("Evaluating model...")
    results = evaluate_model(model, texts, labels)

    # Save results
    output_path = output_dir / "evaluation.json"
    with open(output_path, 'w') as f:
        json.dump(results, f, indent=2)

    logger.info(f"Evaluation report saved to {output_path}")


if __name__ == "__main__":
    main()
```

### 4. Hyperparameter Tuning Job

```python
"""
Hyperparameter Optimization per ticket classifier
File: ml/scripts/hpo_job.py
"""

import boto3
from sagemaker.tuner import (
    HyperparameterTuner,
    IntegerParameter,
    ContinuousParameter,
    CategoricalParameter
)
from sagemaker.estimator import Estimator
import sagemaker

REGION = "us-east-1"
ROLE = "arn:aws:iam::123456789012:role/SageMakerRole"
BUCKET = "ai-support-ml-artifacts"

# BlazingText image
training_image = sagemaker.image_uris.retrieve(
    framework="blazingtext",
    region=REGION
)

# Base estimator
estimator = Estimator(
    image_uri=training_image,
    role=ROLE,
    instance_type="ml.p3.2xlarge",
    instance_count=1,
    output_path=f"s3://{BUCKET}/hpo-models",
    base_job_name="ticket-classifier-hpo",
    metric_definitions=[
        {"Name": "validation:accuracy", "Regex": "validation_accuracy: ([0-9.]+)"}
    ],
    use_spot_instances=True,
    max_wait=3600,
    max_run=1800
)

# Hyperparameter ranges
hyperparameter_ranges = {
    "learning_rate": ContinuousParameter(0.001, 0.1, scaling_type="Logarithmic"),
    "epochs": IntegerParameter(10, 50),
    "word_ngrams": IntegerParameter(1, 3),
    "vector_dim": CategoricalParameter([50, 100, 200]),
    "min_count": IntegerParameter(2, 10)
}

# Fixed hyperparameters
estimator.set_hyperparameters(
    mode="supervised",
    early_stopping=True,
    patience=3
)

# Objective metric
objective_metric_name = "validation:accuracy"
objective_type = "Maximize"

# Tuner configuration
tuner = HyperparameterTuner(
    estimator=estimator,
    objective_metric_name=objective_metric_name,
    hyperparameter_ranges=hyperparameter_ranges,
    metric_definitions=[
        {"Name": "validation:accuracy", "Regex": "validation_accuracy: ([0-9.]+)"}
    ],
    max_jobs=50,        # Numero totale di job
    max_parallel_jobs=5, # Job paralleli
    objective_type=objective_type,
    strategy="Bayesian",  # Bayesian optimization (più efficiente di Random)
    early_stopping_type="Auto"  # Stop early per job non promettenti
)

# Training data
training_data = f"s3://{BUCKET}/processed/train"
validation_data = f"s3://{BUCKET}/processed/validation"

# Start tuning
tuner.fit(
    inputs={
        "train": training_data,
        "validation": validation_data
    },
    job_name="ticket-classifier-hpo-2025-01"
)

print(f"HPO Job started: {tuner.latest_tuning_job.name}")

# Wait for completion (optional)
tuner.wait()

# Get best training job
best_training_job = tuner.best_training_job()
print(f"Best training job: {best_training_job}")

# Best hyperparameters
best_estimator = tuner.best_estimator()
print(f"Best hyperparameters: {best_estimator.hyperparameters()}")
```

### 5. Model Deployment con Blue/Green

```python
"""
Blue/Green deployment per SageMaker endpoint
File: ml/deployment/blue_green_deployment.py
"""

import boto3
import time
from typing import Dict

sagemaker_client = boto3.client('sagemaker')
cloudwatch = boto3.client('cloudwatch')


class BlueGreenDeployment:
    """
    Gestisce deployment blue/green di modelli SageMaker
    """

    def __init__(self, endpoint_name: str):
        self.endpoint_name = endpoint_name
        self.client = sagemaker_client

    def get_current_variant(self) -> Dict:
        """Ottiene configurazione del variant corrente (blue)"""
        response = self.client.describe_endpoint(EndpointName=self.endpoint_name)
        return response['ProductionVariants'][0]

    def create_model_from_package(self, model_package_arn: str, model_name: str):
        """
        Crea modello da Model Registry package
        """
        response = self.client.create_model(
            ModelName=model_name,
            Containers=[{
                'ModelPackageName': model_package_arn
            }],
            ExecutionRoleArn="arn:aws:iam::123456789012:role/SageMakerRole"
        )

        print(f"Model created: {model_name}")
        return response['ModelArn']

    def create_endpoint_config_with_canary(
        self,
        config_name: str,
        blue_model: str,
        green_model: str,
        green_traffic_percentage: int = 10
    ):
        """
        Crea endpoint config con traffic splitting (canary)

        Args:
            green_traffic_percentage: % di traffico sul nuovo modello (green)
        """
        response = self.client.create_endpoint_config(
            EndpointConfigName=config_name,
            ProductionVariants=[
                {
                    'VariantName': 'Blue',
                    'ModelName': blue_model,
                    'InstanceType': 'ml.m5.large',
                    'InitialInstanceCount': 2,
                    'InitialVariantWeight': 100 - green_traffic_percentage
                },
                {
                    'VariantName': 'Green',
                    'ModelName': green_model,
                    'InstanceType': 'ml.m5.large',
                    'InitialInstanceCount': 1,
                    'InitialVariantWeight': green_traffic_percentage
                }
            ]
        )

        print(f"Endpoint config created: {config_name}")
        return response['EndpointConfigArn']

    def update_endpoint(self, new_config_name: str):
        """
        Aggiorna endpoint con nuova configurazione
        """
        response = self.client.update_endpoint(
            EndpointName=self.endpoint_name,
            EndpointConfigName=new_config_name
        )

        print(f"Endpoint update started: {self.endpoint_name}")

        # Attendi completamento
        waiter = self.client.get_waiter('endpoint_in_service')
        waiter.wait(EndpointName=self.endpoint_name)

        print(f"Endpoint update completed")
        return response

    def monitor_canary_metrics(self, duration_minutes: int = 30) -> bool:
        """
        Monitora metriche del canary deployment

        Returns:
            True se metriche OK, False se problemi rilevati
        """
        print(f"Monitoring canary for {duration_minutes} minutes...")

        end_time = time.time() + (duration_minutes * 60)

        while time.time() < end_time:
            # Ottieni metriche per entrambi i variant
            blue_metrics = self._get_variant_metrics('Blue')
            green_metrics = self._get_variant_metrics('Green')

            print(f"\nBlue - Latency: {blue_metrics['latency']:.2f}ms, Errors: {blue_metrics['errors']:.2%}")
            print(f"Green - Latency: {green_metrics['latency']:.2f}ms, Errors: {green_metrics['errors']:.2%}")

            # Check health criteria
            if green_metrics['errors'] > 0.05:  # >5% error rate
                print("❌ Green variant has high error rate")
                return False

            if green_metrics['latency'] > blue_metrics['latency'] * 1.5:  # >50% slower
                print("❌ Green variant has high latency")
                return False

            time.sleep(60)  # Check ogni minuto

        print("✅ Canary metrics look good")
        return True

    def _get_variant_metrics(self, variant_name: str) -> Dict:
        """Ottiene metriche CloudWatch per un variant"""
        # Latency
        latency_response = cloudwatch.get_metric_statistics(
            Namespace='AWS/SageMaker',
            MetricName='ModelLatency',
            Dimensions=[
                {'Name': 'EndpointName', 'Value': self.endpoint_name},
                {'Name': 'VariantName', 'Value': variant_name}
            ],
            StartTime=time.time() - 300,  # Ultimi 5 minuti
            EndTime=time.time(),
            Period=300,
            Statistics=['Average']
        )

        # Errors
        error_response = cloudwatch.get_metric_statistics(
            Namespace='AWS/SageMaker',
            MetricName='ModelInvocation4XXErrors',
            Dimensions=[
                {'Name': 'EndpointName', 'Value': self.endpoint_name},
                {'Name': 'VariantName', 'Value': variant_name}
            ],
            StartTime=time.time() - 300,
            EndTime=time.time(),
            Period=300,
            Statistics=['Sum']
        )

        latency = latency_response['Datapoints'][0]['Average'] if latency_response['Datapoints'] else 0
        errors = error_response['Datapoints'][0]['Sum'] if error_response['Datapoints'] else 0

        return {'latency': latency, 'errors': errors}

    def shift_traffic_incrementally(self, green_model: str, steps: list = [10, 25, 50, 100]):
        """
        Shift graduale del traffico da blue a green

        Args:
            steps: Lista di % di traffico per green (es. [10, 25, 50, 100])
        """
        blue_model = self.get_current_variant()['ModelName']

        for step_percentage in steps:
            print(f"\n{'='*60}")
            print(f"Shifting {step_percentage}% traffic to Green variant")
            print(f"{'='*60}")

            # Crea config con nuovo split
            config_name = f"{self.endpoint_name}-config-green-{step_percentage}pct-{int(time.time())}"
            self.create_endpoint_config_with_canary(
                config_name=config_name,
                blue_model=blue_model,
                green_model=green_model,
                green_traffic_percentage=step_percentage
            )

            # Update endpoint
            self.update_endpoint(config_name)

            # Monitor per 10-30 minuti a seconda dello step
            monitor_duration = 10 if step_percentage < 50 else 30

            if not self.monitor_canary_metrics(duration_minutes=monitor_duration):
                print("❌ Canary failed, rolling back...")
                self.rollback(blue_model)
                return False

            if step_percentage < 100:
                print(f"✅ Step {step_percentage}% successful, proceeding to next step...")
            else:
                print(f"✅ Deployment completed! 100% traffic on Green")

        return True

    def rollback(self, previous_model: str):
        """
        Rollback immediato al modello precedente
        """
        print(f"Rolling back to {previous_model}")

        config_name = f"{self.endpoint_name}-rollback-{int(time.time())}"

        response = self.client.create_endpoint_config(
            EndpointConfigName=config_name,
            ProductionVariants=[{
                'VariantName': 'Blue',
                'ModelName': previous_model,
                'InstanceType': 'ml.m5.large',
                'InitialInstanceCount': 2,
                'InitialVariantWeight': 100
            }]
        )

        self.update_endpoint(config_name)
        print("✅ Rollback completed")


# Example usage
if __name__ == "__main__":
    deployer = BlueGreenDeployment(endpoint_name="ticket-classifier-prod")

    # Deploy nuovo modello approvato dal Model Registry
    new_model_package = "arn:aws:sagemaker:us-east-1:123456789012:model-package/ticket-classifier-models/5"

    # Crea modello da package
    green_model_name = f"ticket-classifier-green-{int(time.time())}"
    deployer.create_model_from_package(new_model_package, green_model_name)

    # Deploy con shift graduale: 10% -> 25% -> 50% -> 100%
    success = deployer.shift_traffic_incrementally(
        green_model=green_model_name,
        steps=[10, 25, 50, 100]
    )

    if success:
        print("🎉 Deployment successful!")
    else:
        print("💥 Deployment failed and rolled back")
```

### 6. Model Monitoring per Drift Detection

```python
"""
SageMaker Model Monitor per drift detection
File: ml/monitoring/model_monitor.py
"""

import boto3
from sagemaker.model_monitor import (
    DataCaptureConfig,
    DefaultModelMonitor,
    CronExpressionGenerator
)
from sagemaker.model_monitor.dataset_format import DatasetFormat
import sagemaker

REGION = "us-east-1"
ROLE = "arn:aws:iam::123456789012:role/SageMakerRole"
BUCKET = "ai-support-ml-artifacts"

session = sagemaker.Session()


def enable_data_capture(endpoint_name: str):
    """
    Abilita data capture per un endpoint
    Cattura input/output delle predizioni per monitoring
    """
    data_capture_config = DataCaptureConfig(
        enable_capture=True,
        sampling_percentage=100,  # Cattura 100% delle richieste (ridurre in prod)
        destination_s3_uri=f"s3://{BUCKET}/model-monitor/data-capture/{endpoint_name}",
        capture_options=["REQUEST", "RESPONSE"],  # Cattura sia input che output
        csv_content_types=["text/csv"],
        json_content_types=["application/json"]
    )

    # Update endpoint per abilitare data capture
    client = boto3.client('sagemaker')

    # Ottieni current config
    endpoint_desc = client.describe_endpoint(EndpointName=endpoint_name)
    current_config = endpoint_desc['EndpointConfigName']

    # Crea nuova config con data capture
    new_config_name = f"{endpoint_name}-with-capture-{int(time.time())}"

    endpoint_config = client.describe_endpoint_config(EndpointConfigName=current_config)

    client.create_endpoint_config(
        EndpointConfigName=new_config_name,
        ProductionVariants=endpoint_config['ProductionVariants'],
        DataCaptureConfig={
            'EnableCapture': True,
            'InitialSamplingPercentage': 100,
            'DestinationS3Uri': f"s3://{BUCKET}/model-monitor/data-capture/{endpoint_name}",
            'CaptureOptions': [
                {'CaptureMode': 'Input'},
                {'CaptureMode': 'Output'}
            ]
        }
    )

    # Update endpoint
    client.update_endpoint(
        EndpointName=endpoint_name,
        EndpointConfigName=new_config_name
    )

    print(f"Data capture enabled for {endpoint_name}")


def create_baseline_job(
    endpoint_name: str,
    baseline_dataset_path: str,
    output_path: str
):
    """
    Crea baseline statistics e constraints dal training dataset

    Baseline serve come riferimento per rilevare drift
    """
    monitor = DefaultModelMonitor(
        role=ROLE,
        instance_count=1,
        instance_type="ml.m5.xlarge",
        volume_size_in_gb=20,
        max_runtime_in_seconds=3600,
        sagemaker_session=session
    )

    # Crea baseline
    baseline_job = monitor.suggest_baseline(
        baseline_dataset=baseline_dataset_path,  # S3 path del training set
        dataset_format=DatasetFormat.csv(header=False),
        output_s3_uri=output_path,
        wait=True
    )

    print(f"Baseline job completed: {baseline_job.job_name}")
    print(f"Baseline statistics: {output_path}/statistics.json")
    print(f"Baseline constraints: {output_path}/constraints.json")

    return monitor


def schedule_monitoring_job(
    endpoint_name: str,
    monitor: DefaultModelMonitor,
    baseline_statistics_path: str,
    baseline_constraints_path: str,
    schedule_expression: str = "cron(0 * * * ? *)"  # Ogni ora
):
    """
    Schedule monitoring job che gira periodicamente

    Confronta traffico corrente con baseline e genera report
    """
    monitor.create_monitoring_schedule(
        monitor_schedule_name=f"{endpoint_name}-monitor-schedule",
        endpoint_input=endpoint_name,
        output_s3_uri=f"s3://{BUCKET}/model-monitor/reports/{endpoint_name}",
        statistics=baseline_statistics_path,
        constraints=baseline_constraints_path,
        schedule_cron_expression=CronExpressionGenerator.hourly(),  # Ogni ora
        enable_cloudwatch_metrics=True
    )

    print(f"Monitoring schedule created for {endpoint_name}")
    print(f"Schedule: {schedule_expression}")


def analyze_monitoring_violations(report_path: str):
    """
    Analizza violation report generato dal monitor
    """
    import json

    s3 = boto3.client('s3')

    # Parse S3 path
    bucket = report_path.replace("s3://", "").split("/")[0]
    key = "/".join(report_path.replace("s3://", "").split("/")[1:])

    # Download constraint violations
    violations_key = f"{key}/constraint_violations.json"

    try:
        obj = s3.get_object(Bucket=bucket, Key=violations_key)
        violations = json.loads(obj['Body'].read())

        if violations.get('violations'):
            print(f"⚠️  {len(violations['violations'])} violations detected:")

            for v in violations['violations']:
                print(f"\n  - Metric: {v['metric_name']}")
                print(f"    Description: {v['description']}")
                print(f"    Constraint: {v['constraint_check_type']}")

            return violations
        else:
            print("✅ No violations detected")
            return None

    except s3.exceptions.NoSuchKey:
        print("✅ No violations file (all constraints met)")
        return None


def create_cloudwatch_alarm_for_drift(endpoint_name: str):
    """
    Crea CloudWatch alarm che notifica quando drift è rilevato
    """
    cloudwatch = boto3.client('cloudwatch')
    sns = boto3.client('sns')

    # Crea SNS topic per notifiche
    topic_response = sns.create_topic(Name=f"{endpoint_name}-drift-alerts")
    topic_arn = topic_response['TopicArn']

    # Subscribe email
    sns.subscribe(
        TopicArn=topic_arn,
        Protocol='email',
        Endpoint='ml-team@company.com'
    )

    # Crea alarm
    cloudwatch.put_metric_alarm(
        AlarmName=f"{endpoint_name}-data-drift",
        AlarmDescription="Alert when data drift is detected",
        ActionsEnabled=True,
        AlarmActions=[topic_arn],
        MetricName='feature_baseline_drift_total_count',
        Namespace='aws/sagemaker/Endpoints/data-metrics',
        Statistic='Sum',
        Dimensions=[
            {'Name': 'Endpoint', 'Value': endpoint_name}
        ],
        Period=3600,  # 1 ora
        EvaluationPeriods=1,
        Threshold=1.0,  # Alert se >= 1 violation
        ComparisonOperator='GreaterThanOrEqualToThreshold'
    )

    print(f"CloudWatch alarm created: {endpoint_name}-data-drift")
    print(f"Notifications will be sent to: {topic_arn}")


# Complete monitoring setup
if __name__ == "__main__":
    ENDPOINT_NAME = "ticket-classifier-prod"
    BASELINE_DATA = f"s3://{BUCKET}/processed/train/train.csv"
    BASELINE_OUTPUT = f"s3://{BUCKET}/model-monitor/baseline/{ENDPOINT_NAME}"

    # Step 1: Enable data capture
    enable_data_capture(ENDPOINT_NAME)

    # Step 2: Create baseline
    monitor = create_baseline_job(
        endpoint_name=ENDPOINT_NAME,
        baseline_dataset_path=BASELINE_DATA,
        output_path=BASELINE_OUTPUT
    )

    # Step 3: Schedule monitoring
    schedule_monitoring_job(
        endpoint_name=ENDPOINT_NAME,
        monitor=monitor,
        baseline_statistics_path=f"{BASELINE_OUTPUT}/statistics.json",
        baseline_constraints_path=f"{BASELINE_OUTPUT}/constraints.json"
    )

    # Step 4: Create CloudWatch alarm
    create_cloudwatch_alarm_for_drift(ENDPOINT_NAME)

    print("\n✅ Model monitoring setup completed!")
```

### 7. Automated Retraining con EventBridge

```python
"""
Automated retraining trigger con EventBridge
File: ml/automation/retraining_trigger.py
"""

import boto3
import json
from datetime import datetime

events = boto3.client('events')
sagemaker = boto3.client('sagemaker')
lambda_client = boto3.client('lambda')


def create_retraining_lambda():
    """
    Lambda function che triggera il retraining pipeline
    """
    lambda_code = '''
import boto3
import json

sagemaker = boto3.client('sagemaker')

def lambda_handler(event, context):
    """
    Triggera SageMaker pipeline per retraining

    Event sources:
    1. Scheduled (weekly)
    2. Data drift detected
    3. Model performance degradation
    4. Manual trigger
    """

    pipeline_name = "TicketClassifierTrainingPipeline"

    # Trigger reason
    trigger_source = event.get('source', 'manual')

    print(f"Retraining triggered by: {trigger_source}")

    # Parametri dinamici basati sul trigger
    pipeline_params = {
        "InputDataUrl": f"s3://ai-support-ml-artifacts/training-data/tickets-latest.jsonl",
        "AccuracyThreshold": 0.87
    }

    # Se triggered da drift, abbassa threshold
    if trigger_source == "model-monitor":
        pipeline_params["AccuracyThreshold"] = 0.85
        print("Drift detected, lowering accuracy threshold to 0.85")

    # Start pipeline execution
    response = sagemaker.start_pipeline_execution(
        PipelineName=pipeline_name,
        PipelineParameters=[
            {"Name": k, "Value": str(v)} for k, v in pipeline_params.items()
        ],
        PipelineExecutionDescription=f"Automated retraining triggered by {trigger_source}"
    )

    execution_arn = response['PipelineExecutionArn']

    print(f"Pipeline execution started: {execution_arn}")

    # Send notification
    sns = boto3.client('sns')
    sns.publish(
        TopicArn="arn:aws:sns:us-east-1:123456789012:ml-team-notifications",
        Subject="SageMaker Retraining Started",
        Message=f"Pipeline execution: {execution_arn}\\nTrigger: {trigger_source}"
    )

    return {
        'statusCode': 200,
        'body': json.dumps({
            'execution_arn': execution_arn,
            'trigger_source': trigger_source
        })
    }
'''

    # Crea Lambda function
    response = lambda_client.create_function(
        FunctionName='trigger-retraining-pipeline',
        Runtime='python3.11',
        Role='arn:aws:iam::123456789012:role/LambdaSageMakerRole',
        Handler='index.lambda_handler',
        Code={'ZipFile': lambda_code.encode()},
        Timeout=60,
        MemorySize=256
    )

    return response['FunctionArn']


def create_scheduled_retraining_rule(lambda_arn: str):
    """
    EventBridge rule per retraining schedulato (ogni settimana)
    """
    rule_name = "weekly-model-retraining"

    # Crea rule
    events.put_rule(
        Name=rule_name,
        Description="Trigger model retraining every Sunday at 2 AM",
        ScheduleExpression="cron(0 2 ? * SUN *)",  # Domenica alle 2 AM UTC
        State='ENABLED'
    )

    # Add Lambda target
    events.put_targets(
        Rule=rule_name,
        Targets=[
            {
                'Id': '1',
                'Arn': lambda_arn,
                'Input': json.dumps({
                    'source': 'scheduled',
                    'reason': 'Weekly automated retraining'
                })
            }
        ]
    )

    print(f"Scheduled retraining rule created: {rule_name}")


def create_drift_trigger_rule(lambda_arn: str):
    """
    EventBridge rule che triggera retraining quando drift è rilevato
    """
    rule_name = "data-drift-retraining-trigger"

    # Pattern per catturare CloudWatch alarm
    event_pattern = {
        "source": ["aws.cloudwatch"],
        "detail-type": ["CloudWatch Alarm State Change"],
        "detail": {
            "alarmName": [{
                "prefix": "ticket-classifier-prod-data-drift"
            }],
            "state": {
                "value": ["ALARM"]
            }
        }
    }

    # Crea rule
    events.put_rule(
        Name=rule_name,
        Description="Trigger retraining when data drift alarm fires",
        EventPattern=json.dumps(event_pattern),
        State='ENABLED'
    )

    # Add Lambda target
    events.put_targets(
        Rule=rule_name,
        Targets=[
            {
                'Id': '1',
                'Arn': lambda_arn,
                'Input': json.dumps({
                    'source': 'model-monitor',
                    'reason': 'Data drift detected'
                })
            }
        ]
    )

    print(f"Drift trigger rule created: {rule_name}")


def create_performance_trigger_rule(lambda_arn: str):
    """
    Trigger retraining quando performance degrada
    """
    rule_name = "model-performance-degradation-trigger"

    event_pattern = {
        "source": ["aws.cloudwatch"],
        "detail-type": ["CloudWatch Alarm State Change"],
        "detail": {
            "alarmName": [{
                "prefix": "ticket-classifier-prod-accuracy"
            }],
            "state": {
                "value": ["ALARM"]
            }
        }
    }

    events.put_rule(
        Name=rule_name,
        Description="Trigger retraining when model accuracy drops",
        EventPattern=json.dumps(event_pattern),
        State='ENABLED'
    )

    events.put_targets(
        Rule=rule_name,
        Targets=[
            {
                'Id': '1',
                'Arn': lambda_arn,
                'Input': json.dumps({
                    'source': 'performance-degradation',
                    'reason': 'Model accuracy below threshold'
                })
            }
        ]
    )

    print(f"Performance trigger rule created: {rule_name}")


# Setup automation
if __name__ == "__main__":
    print("Setting up automated retraining...")

    # Create Lambda function
    lambda_arn = create_retraining_lambda()
    print(f"Lambda created: {lambda_arn}")

    # Create EventBridge rules
    create_scheduled_retraining_rule(lambda_arn)
    create_drift_trigger_rule(lambda_arn)
    create_performance_trigger_rule(lambda_arn)

    print("\n✅ Automated retraining setup completed!")
    print("\nRetraining will be triggered by:")
    print("  1. Schedule: Every Sunday at 2 AM UTC")
    print("  2. Data drift: When drift alarm fires")
    print("  3. Performance: When accuracy drops below threshold")
```

---

## Best Practices

### Pipeline Design

**✅ DO**:
- **Parametrizzare tutto**: Pipeline parameters per input dinamici
- **Idempotenza**: Gli step devono essere safely re-runnable
- **Conditional steps**: Deploy solo se metriche soddisfano threshold
- **Metadata tracking**: Collega execution a ticket/issue per traceability
- **Caching**: Usa `CacheConfig` per step costosi che non cambiano

**❌ DON'T**:
- **Hardcode paths**: Usa parameters per S3 paths e configurazioni
- **Ignora failures**: Sempre gestire errori con retry o fallback
- **Skip evaluation**: Mai deployare senza evaluation step
- **Mix concerns**: Un pipeline = un obiettivo (training, batch inference, etc.)

### Instance Selection

**Training instances**:

```python
# Small models (< 100MB, < 1M params)
instance_type = "ml.m5.xlarge"  # CPU-only, cost-effective

# Medium models (100MB-1GB, 1M-50M params)
instance_type = "ml.p3.2xlarge"  # 1 GPU, good balance

# Large models (> 1GB, > 50M params)
instance_type = "ml.p3.8xlarge"  # 4 GPUs for distributed training

# Budget-conscious (non-urgent training)
use_spot_instances = True
max_wait = 2 * max_run  # Consenti 2x il tempo per gestire interruzioni
```

**Inference instances**:

```python
# Low latency (<100ms), high throughput
instance_type = "ml.c5.2xlarge"  # Compute-optimized

# GPU-accelerated (deep learning models)
instance_type = "ml.g4dn.xlarge"  # Cost-effective GPU

# Variable traffic
endpoint_type = "Serverless"  # Pay-per-use, auto-scaling

# Predictable traffic
endpoint_type = "Real-time"
auto_scaling_enabled = True
```

### Performance Optimization

**1. Spot Instances**:
```python
estimator = Estimator(
    # ... altre config
    use_spot_instances=True,
    max_wait=7200,      # Max wait time (include interruzioni)
    max_run=3600,       # Max training time
    checkpoint_s3_uri=f"s3://{BUCKET}/checkpoints"  # Per resume
)
```

**2. Distributed Training**:
```python
estimator = Estimator(
    # ... altre config
    instance_type="ml.p3.8xlarge",
    instance_count=4,  # 4 instances = 16 GPUs totali
    distribution={
        "smdistributed": {
            "dataparallel": {
                "enabled": True
            }
        }
    }
)
```

**3. Pipe Mode** (per grandi dataset):
```python
# Streaming di dati da S3 invece di download completo
training_input = TrainingInput(
    s3_data=train_data_path,
    input_mode="Pipe"  # Default è "File"
)
```

### Security

**1. Encryption**:
```python
estimator = Estimator(
    # ... altre config
    encrypt_inter_container_traffic=True,  # TLS tra container
    volume_kms_key="arn:aws:kms:...",     # Encrypt EBS volumes
    output_kms_key="arn:aws:kms:..."       # Encrypt model artifacts
)
```

**2. Network Isolation**:
```python
estimator = Estimator(
    # ... altre config
    enable_network_isolation=True,  # No internet access durante training
    subnets=["subnet-xxx"],
    security_group_ids=["sg-xxx"]
)
```

**3. IAM Least Privilege**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::ai-support-ml-artifacts/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sagemaker:CreateTrainingJob",
        "sagemaker:DescribeTrainingJob"
      ],
      "Resource": "arn:aws:sagemaker:*:*:training-job/ticket-classifier-*"
    }
  ]
}
```

### Cost Optimization

**1. Right-sizing**:
```python
# Test con instance piccola prima di scaling
instance_type = "ml.m5.xlarge"  # Start small
# Monitor CloudWatch metrics (CPUUtilization, GPUUtilization)
# Scale up solo se necessary
```

**2. Spot Instances** (fino a 90% risparmio):
```python
use_spot_instances = True
# Funziona bene per training non urgenti
```

**3. Model Endpoint Auto-scaling**:
```python
client = boto3.client('application-autoscaling')

# Register scalable target
client.register_scalable_target(
    ServiceNamespace='sagemaker',
    ResourceId=f'endpoint/{endpoint_name}/variant/AllTraffic',
    ScalableDimension='sagemaker:variant:DesiredInstanceCount',
    MinCapacity=1,
    MaxCapacity=10
)

# Target tracking policy
client.put_scaling_policy(
    PolicyName='InvocationsPerInstance',
    ServiceNamespace='sagemaker',
    ResourceId=f'endpoint/{endpoint_name}/variant/AllTraffic',
    ScalableDimension='sagemaker:variant:DesiredInstanceCount',
    PolicyType='TargetTrackingScaling',
    TargetTrackingScalingPolicyConfiguration={
        'TargetValue': 1000.0,  # Target 1000 invocations/instance
        'PredefinedMetricSpecification': {
            'PredefinedMetricType': 'SageMakerVariantInvocationsPerInstance'
        },
        'ScaleInCooldown': 300,   # 5 min
        'ScaleOutCooldown': 60    # 1 min
    }
)
```

**4. Serverless Endpoints** (per traffic variabile):
```python
endpoint_config = sagemaker_client.create_endpoint_config(
    EndpointConfigName=config_name,
    ProductionVariants=[{
        'VariantName': 'AllTraffic',
        'ModelName': model_name,
        'ServerlessConfig': {
            'MemorySizeInMB': 2048,  # 1-6 GB
            'MaxConcurrency': 50      # Max concurrent invocations
        }
    }]
)
# Pay solo per invocations, no idle cost
```

---

## Troubleshooting

### Training Job Failures

**Problem**: Training job failed with `ResourceLimitExceeded`

```
ClientError: An error occurred (ResourceLimitExceeded) when calling the
CreateTrainingJob operation: The account-level service limit 'ml.p3.2xlarge
for training job usage' is 0 Instances
```

**Solution**:
1. Verifica service quotas: `aws service-quotas list-service-quotas --service-code sagemaker`
2. Richiedi aumento: AWS Console → Service Quotas → SageMaker
3. Temporary workaround: usa instance type diverso (es. `ml.m5.xlarge` se CPU-only)

---

**Problem**: Out of Memory (OOM) durante training

```
Killed - Training job terminated due to OOM error
```

**Solution**:
```python
# 1. Riduci batch size
hyperparameters = {
    "batch_size": 16  # Era 64, riduci a 16 o 32
}

# 2. Usa gradient accumulation
hyperparameters = {
    "batch_size": 8,
    "gradient_accumulation_steps": 4  # Effective batch size = 8*4 = 32
}

# 3. Scale up instance
instance_type = "ml.p3.8xlarge"  # 244 GB RAM invece di 61 GB
```

---

**Problem**: Training troppo lento

**Diagnosi**:
```python
# Controlla CloudWatch metrics
cloudwatch = boto3.client('cloudwatch')

response = cloudwatch.get_metric_statistics(
    Namespace='AWS/SageMaker',
    MetricName='GPUUtilization',
    Dimensions=[{'Name': 'TrainingJobName', 'Value': training_job_name}],
    StartTime=...,
    EndTime=...,
    Period=300,
    Statistics=['Average']
)

# Se GPU util < 50%: I/O bottleneck
# Se GPU util > 90%: Model compute-bound
```

**Solutions**:
```python
# A. I/O bottleneck → Usa Pipe mode
training_input = TrainingInput(s3_data=..., input_mode="Pipe")

# B. Compute-bound → Distributed training
instance_count = 4
distribution = {"smdistributed": {"dataparallel": {"enabled": True}}}

# C. Data preprocessing bottleneck → Più workers
hyperparameters = {
    "num_data_workers": 8  # Default spesso 4
}
```

### Endpoint Issues

**Problem**: High latency (>1s per prediction)

**Diagnosi**:
```python
# Check model size
import boto3
s3 = boto3.client('s3')
model_size = s3.head_object(Bucket=bucket, Key=model_key)['ContentLength']
print(f"Model size: {model_size / 1e6:.2f} MB")

# Check instance CPU/Memory
cloudwatch.get_metric_statistics(
    Namespace='AWS/SageMaker',
    MetricName='CPUUtilization',
    Dimensions=[
        {'Name': 'EndpointName', 'Value': endpoint_name},
        {'Name': 'VariantName', 'Value': 'AllTraffic'}
    ],
    ...
)
```

**Solutions**:
```python
# 1. Scale up instance
instance_type = "ml.c5.4xlarge"  # Era ml.t2.medium

# 2. Enable model compilation (SageMaker Neo)
compiled_model = model.compile(
    target_instance_family='ml_c5',
    input_shape={'data': [1, 512]},  # Adatta al tuo input
    output_path=f"s3://{BUCKET}/compiled-models",
    framework='pytorch',
    framework_version='1.13'
)

# 3. Batch predictions
predictor.predict(batch_data, initial_args={'ContentType': 'application/json'})

# 4. Usa Multi-Model Endpoint per molti modelli piccoli
```

---

**Problem**: Endpoint ritorna 5xx errors

**Diagnosi**:
```bash
# Check CloudWatch logs
aws logs tail /aws/sagemaker/Endpoints/{endpoint-name} --follow

# Common errors:
# - Model deserialization failed
# - Container health check failed
# - Memory exhausted
```

**Solutions**:
```python
# 1. Verifica formato input
# Test localmente prima
from sagemaker.predictor import Predictor
from sagemaker.serializers import JSONSerializer
from sagemaker.deserializers import JSONDeserializer

predictor = Predictor(
    endpoint_name=endpoint_name,
    serializer=JSONSerializer(),
    deserializer=JSONDeserializer()
)

# Payload corretto per BlazingText
payload = {"instances": ["router not working dhcp issue"]}
response = predictor.predict(payload)

# 2. Aumenta memory/timeout
instance_type = "ml.m5.xlarge"  # 16 GB invece di 8 GB

# 3. Check container logs per dettagli
```

### Model Monitor Issues

**Problem**: Nessuna violation rilevata nonostante drift evidente

**Diagnosi**:
```python
# Verifica baseline constraints
import json
import boto3

s3 = boto3.client('s3')
obj = s3.get_object(Bucket=bucket, Key=f"{baseline_path}/constraints.json")
constraints = json.loads(obj['Body'].read())

print(json.dumps(constraints, indent=2))

# Check monitoring execution status
client = boto3.client('sagemaker')
executions = client.list_monitoring_executions(
    MonitoringScheduleName=schedule_name,
    MaxResults=10
)

for exec in executions['MonitoringExecutionSummaries']:
    print(f"Execution: {exec['MonitoringExecutionStatus']}")
    if exec['MonitoringExecutionStatus'] == 'Failed':
        print(f"  Failure: {exec.get('FailureReason')}")
```

**Solutions**:
```python
# 1. Rigenera baseline con più dati
monitor.suggest_baseline(
    baseline_dataset=larger_dataset,  # Usa dataset più grande
    dataset_format=DatasetFormat.csv(header=False)
)

# 2. Adjust constraints manualmente
constraints['features'][0]['numerical_constraints']['upper_bound'] = 10.0

# 3. Increase monitoring frequency
schedule_expression = CronExpressionGenerator.daily()  # Era hourly
```

---

**Problem**: Data capture non funziona

```
No data captured in S3 bucket
```

**Diagnosi**:
```python
# Verifica che data capture sia abilitato
endpoint_desc = sagemaker_client.describe_endpoint(EndpointName=endpoint_name)
config_name = endpoint_desc['EndpointConfigName']

config = sagemaker_client.describe_endpoint_config(EndpointConfigName=config_name)
print(config.get('DataCaptureConfig'))

# Check IAM permissions per S3 write
```

**Solutions**:
```python
# 1. Verifica permissions
# SageMaker execution role deve avere s3:PutObject sul bucket

# 2. Check sampling percentage
data_capture_config = DataCaptureConfig(
    enable_capture=True,
    sampling_percentage=100,  # Era 0?
    destination_s3_uri=...
)

# 3. Aspetta un po'
# Data capture può richiedere 5-10 minuti per apparire
```

---

## Esempi Reali dal Progetto

### Esempio 1: Ticket Classification Pipeline

Nel nostro sistema AI Technical Support, il **ticket classifier** è il primo modello nella catena di processing.

**Obiettivo**: Classificare ticket in 10 categorie tecniche (networking, storage, compute, database, etc.)

**Dataset**:
- 50,000 ticket storici da DynamoDB
- Bilanciati per categoria (5,000 per categoria)
- Testo preprocessato: 50-500 token

**Pipeline**:
```
Input (S3) → Preprocessing → Training (BlazingText) → Evaluation →
Conditional Registration (accuracy >= 0.85) → Model Registry →
Approval → Deployment (Blue/Green) → Monitoring
```

**Risultati**:
- **Accuracy**: 0.89 su test set
- **Training time**: 15 minuti (ml.p3.2xlarge con spot)
- **Cost per training**: ~$0.50 (spot instances)
- **Inference latency**: 25ms (ml.c5.xlarge)
- **Retraining frequency**: Weekly + on-drift

**File di riferimento**: `docs/04-data-flows/ticket-processing.md:45-67`

### Esempio 2: Sentiment Analysis Model

**Obiettivo**: Analizzare sentiment del customer per prioritizzazione intelligente

**Approccio**:
```python
# Usa Amazon Comprehend (managed) + custom model per domain-specific terms

from sagemaker.huggingface import HuggingFace

# Fine-tune BERT per technical support sentiment
huggingface_estimator = HuggingFace(
    entry_point='train.py',
    source_dir='./scripts',
    instance_type='ml.p3.2xlarge',
    instance_count=1,
    role=role,
    transformers_version='4.26',
    pytorch_version='1.13',
    py_version='py39',
    hyperparameters={
        'model_name_or_path': 'bert-base-uncased',
        'num_labels': 5,  # Very Negative, Negative, Neutral, Positive, Very Positive
        'epochs': 3,
        'train_batch_size': 32,
        'learning_rate': 5e-5
    }
)
```

**Integrazione con Step Functions**:
```json
{
  "StartAt": "ClassifyTicket",
  "States": {
    "ClassifyTicket": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sagemaker:createEndpoint.sync",
      "Parameters": {
        "EndpointName": "ticket-classifier-prod",
        "Input.$": "$.ticket_content"
      },
      "Next": "AnalyzeSentiment"
    },
    "AnalyzeSentiment": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sagemaker:createEndpoint.sync",
      "Parameters": {
        "EndpointName": "sentiment-analyzer-prod",
        "Input.$": "$.ticket_content"
      },
      "Next": "DeterminePriority"
    }
  }
}
```

**Risultati**:
- **F1-Score**: 0.87 (balanced across 5 sentiment classes)
- **Impact**: Riduzione del 30% nei ticket escalated impropriamente

### Esempio 3: Automated Retraining su Concept Drift

**Scenario**: Dopo 2 settimane in produzione, il Model Monitor rileva drift nella distribuzione delle categorie.

**Drift rilevato**:
```json
{
  "violations": [
    {
      "metric_name": "category_distribution",
      "description": "Distribution of 'kubernetes' category increased from 8% to 18%",
      "constraint_check_type": "data_distribution_check"
    }
  ]
}
```

**Azione automatica**:
1. **CloudWatch Alarm** fires
2. **EventBridge rule** cattura l'alarm
3. **Lambda function** triggera retraining pipeline
4. **Pipeline** runs con dati updated (include nuovi ticket Kubernetes)
5. **New model** (v1.5) registered con accuracy 0.91 (migliorata da 0.89)
6. **Approval automatico** (supera threshold)
7. **Blue/Green deployment**: 10% → 25% → 50% → 100% in 2 ore
8. **Monitoring** conferma: latency invariata, accuracy migliorata

**Risultato**: Drift risolto automaticamente senza intervento manuale

---

## Riferimenti

### AWS Documentation

- [SageMaker Pipelines Developer Guide](https://docs.aws.amazon.com/sagemaker/latest/dg/pipelines.html)
- [SageMaker Model Registry](https://docs.aws.amazon.com/sagemaker/latest/dg/model-registry.html)
- [SageMaker Model Monitor](https://docs.aws.amazon.com/sagemaker/latest/dg/model-monitor.html)
- [SageMaker Endpoints](https://docs.aws.amazon.com/sagemaker/latest/dg/deploy-model.html)
- [BlazingText Algorithm](https://docs.aws.amazon.com/sagemaker/latest/dg/blazingtext.html)
- [Hyperparameter Tuning](https://docs.aws.amazon.com/sagemaker/latest/dg/automatic-model-tuning.html)

### Blog Posts & Tutorials

- [MLOps with SageMaker Pipelines](https://aws.amazon.com/blogs/machine-learning/building-automating-managing-and-scaling-ml-workflows-using-amazon-sagemaker-pipelines/)
- [Blue/Green Deployments](https://aws.amazon.com/blogs/machine-learning/amazon-sagemaker-now-supports-deploying-machine-learning-models-with-automated-rollback/)
- [Spot Instances Best Practices](https://docs.aws.amazon.com/sagemaker/latest/dg/model-managed-spot-training.html)
- [Cost Optimization for SageMaker](https://aws.amazon.com/blogs/machine-learning/ensure-efficient-compute-resources-on-amazon-sagemaker/)

### Files del Progetto

- `docs/12-implementation/roadmap.md#mlops-automation` - MLOps implementation roadmap
- `docs/03-aws-services/README.md#sagemaker` - SageMaker service overview
- `docs/04-data-flows/ticket-processing.md` - Ticket processing workflow

### Tools & Libraries

- [SageMaker Python SDK](https://github.com/aws/sagemaker-python-sdk)
- [SageMaker Examples](https://github.com/aws/amazon-sagemaker-examples)
- [MLflow](https://mlflow.org/) - Experiment tracking (alternative)
- [Weights & Biases](https://wandb.ai/) - Experiment tracking (alternative)

---

**Maintainer**: ML Engineering Team
**Last Updated**: 2025-11-18
**Status**: Production Ready
**Version**: 1.0
