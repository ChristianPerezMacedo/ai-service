# Servizi AWS - Overview

## Mappa Servizi

Il sistema utilizza 19 servizi AWS distribuiti su 6 categorie funzionali.

### Compute & Orchestration
- **[API Gateway](./api-gateway.md)**: Entry point per tutte le API REST
- **[Lambda](./lambda.md)**: Compute serverless per business logic
- **[Step Functions](./step-functions.md)**: Orchestrazione workflow asincroni
- AWS Batch: Job batch per training ML (future)

### AI/ML Services
- **[Amazon Bedrock](./bedrock.md)**: LLM per generazione risposte e embeddings
- **[SageMaker](./sagemaker.md)**: Training e deployment modelli classificazione
- **[Textract](./textract.md)**: OCR per estrazione testo da PDF
- Comprehend (future): NER e sentiment analysis

### Data & Storage
- **[DynamoDB](./dynamodb.md)**: NoSQL database per stato ticket
- **[OpenSearch](./opensearch.md)**: Vector database per knowledge base
- **[S3](./s3.md)**: Object storage per documenti e modelli
- RDS PostgreSQL: Analytics e reporting (future)

### Integration & Messaging
- **[EventBridge](./eventbridge.md)**: Event bus e scheduling
- SQS: Code messaggi per integrazione ITSM
- SNS: Notifiche e pub/sub
- Kinesis: Streaming dati real-time

### Security & Identity
- **[Cognito](./cognito.md)**: User authentication e authorization
- Secrets Manager: Gestione credenziali
- KMS: Encryption key management
- IAM: Access control

### Monitoring & Operations
- **[CloudWatch](./cloudwatch.md)**: Logging, metriche, allarmi
- X-Ray: Distributed tracing
- CloudTrail: Audit logging
- AWS Config: Compliance monitoring

## Mapping Funzionale

| Funzione | Servizio AWS | Alternative Considerate | Note Scelta |
|----------|--------------|-------------------------|-------------|
| **Storage** | S3 | MinIO, EFS | Costo-efficienza, integrazione AWS |
| **OCR** | Textract | Tesseract, Google Vision | Qualità output, gestione tabelle |
| **Vector DB** | OpenSearch | Pinecone, Weaviate | Controllo fine-grained, costi |
| **Classification** | SageMaker | Vertex AI, Azure ML | Ecosistema AWS, MLOps integrato |
| **LLM** | Bedrock | OpenAI API, Azure OpenAI | Data residency EU, compliance |
| **Orchestration** | Step Functions | Airflow, Temporal | Serverless, integrazione nativa |
| **API** | API Gateway | Kong, Apigee | Managed service, auth integrata |
| **Auth** | Cognito | Auth0, Keycloak | Costo, integrazione API Gateway |
| **Monitoring** | CloudWatch | Datadog, New Relic | Zero configurazione, costi inclusi |
| **NoSQL** | DynamoDB | MongoDB, Cassandra | Performance, scalabilità automatica |

## Costi Mensili Stimati

| Servizio | Volume | Costo Mensile |
|----------|--------|---------------|
| **S3 Storage** | 100 GB | $25 |
| **S3 Requests** | 1M GET, 100K PUT | $5 |
| **Textract** | 10K pagine/mese | $150 |
| **OpenSearch** | 3x r6g.large | $360 |
| **SageMaker Endpoint** | 2x ml.m5.large | $200 |
| **Bedrock (Claude)** | 15M tokens/mese | $300 |
| **Bedrock (Embeddings)** | 50M tokens/mese | $10 |
| **Step Functions** | 50K executions | $50 |
| **Lambda** | 5M invocations, 3GB-sec | $40 |
| **API Gateway** | 1M requests | $35 |
| **DynamoDB** | On-demand, 10GB | $15 |
| **EventBridge** | 100K events | $10 |
| **CloudWatch** | Logs 50GB, metrics | $30 |
| **NAT Gateway** | 100GB transfer | $50 |
| **Data Transfer** | 500GB out | $45 |
| **RDS (future)** | db.t3.medium | $80 |
| **TOTALE** | | **~$1,405/mese** |

**Note**:
- Costi variabili basati su utilizzo effettivo
- Possibilità di Reserved Instances per -40% su OpenSearch e RDS
- S3 Intelligent-Tiering per ottimizzare storage
- Stima per ~1000 ticket/mese elaborati

## Limiti di Servizio

### Limiti Soft (richiedibili aumenti)

| Servizio | Limite Default | Limite Richiesto | Status |
|----------|----------------|------------------|--------|
| Lambda Concurrent Executions | 1,000 | 1,500 | ✅ Approvato |
| API Gateway Requests/sec | 10,000 | 10,000 | Default OK |
| Bedrock Claude RPM | 100 | 300 | 📋 In richiesta |
| SageMaker Endpoints | 10 | 10 | Default OK |
| DynamoDB Table Count | 2,500 | 10 | Default OK |

### Limiti Hard (non modificabili)

| Servizio | Limite | Workaround |
|----------|--------|------------|
| Lambda Max Execution Time | 15 min | Step Functions per workflow lunghi |
| API Gateway Timeout | 29 sec | Async pattern con SQS |
| Lambda Deployment Package | 250 MB | Lambda Layers, ECR |
| DynamoDB Item Size | 400 KB | Compressione, reference a S3 |
| Step Functions Execution Time | 1 anno | N/A (sufficiente) |

## Service Level Agreements (SLA)

| Servizio | SLA AWS | Target Nostro | Impatto se Down |
|----------|---------|---------------|-----------------|
| API Gateway | 99.95% | 99.9% | No nuovi ticket |
| Lambda | 99.95% | 99.9% | Processing interrotto |
| DynamoDB | 99.99% | 99.95% | Data loss potenziale |
| S3 | 99.9% | 99.9% | Impossibile leggere KB |
| Step Functions | 99.9% | 99.5% | Workflow manuali |
| Bedrock | 99.5% | 99% | Fallback su cache |
| SageMaker | 99.9% | 99% | Degraded classification |
| OpenSearch | 99.9% | 99.5% | Degraded search |

## Dipendenze tra Servizi

### Tier 0 (Fondamentali)
```
VPC, Subnets, Security Groups
IAM Roles
KMS Keys
S3 Buckets
```

### Tier 1 (Core Data)
```
DynamoDB Tables ← Tier 0
Secrets Manager ← KMS
CloudWatch Log Groups ← IAM
```

### Tier 2 (Compute)
```
Lambda Functions ← Tier 0, Tier 1
API Gateway ← IAM, Lambda
EventBridge ← IAM
```

### Tier 3 (ML/AI)
```
SageMaker ← S3, IAM
Bedrock Configuration ← IAM
OpenSearch ← VPC, Tier 0
Textract ← IAM
```

### Tier 4 (Orchestration)
```
Step Functions ← Lambda, SageMaker, Bedrock
SNS ← IAM
```

## Quick Reference

### Identificatori Risorsa

```
Region: eu-south-1 (Milano primary)
Account ID: XXXXXXXXXXXX

Naming Convention:
{env}-{service}-{function}-{region}

Examples:
  prod-lambda-ticket-processor-eus1
  prod-dynamodb-tickets-eus1
  prod-opensearch-kb-eus1
```

### Endpoint URLs

```
API Gateway:
  https://api.example.com/v1/

S3 Buckets:
  s3://prod-raw-data-eus1
  s3://prod-processed-data-eus1
  s3://prod-ml-models-eus1

OpenSearch:
  https://vpc-prod-opensearch-kb-xxxxx.eu-south-1.es.amazonaws.com

RDS:
  prod-rds-analytics.xxxxx.eu-south-1.rds.amazonaws.com
```

## Riferimenti Dettagliati

### Compute
- [API Gateway](./api-gateway.md) - REST API management
- [Lambda](./lambda.md) - Serverless compute
- [Step Functions](./step-functions.md) - Workflow orchestration

### AI/ML
- [Bedrock](./bedrock.md) - LLM inference
- [SageMaker](./sagemaker.md) - ML training/deployment
- [Textract](./textract.md) - Document OCR

### Data
- [DynamoDB](./dynamodb.md) - NoSQL database
- [OpenSearch](./opensearch.md) - Vector search
- [S3](./s3.md) - Object storage

### Integration
- [EventBridge](./eventbridge.md) - Event bus
- SQS - Message queuing
- SNS - Pub/Sub notifications

### Security
- [Cognito](./cognito.md) - Authentication
- Secrets Manager - Credential storage
- KMS - Encryption

### Monitoring
- [CloudWatch](./cloudwatch.md) - Observability
- X-Ray - Distributed tracing
- CloudTrail - Audit logs
