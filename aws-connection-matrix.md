# 🗺️ Mappa Dettagliata Collegamenti AWS - Matrice di Connessione

## 📊 Matrice di Interconnessione Servizi

### Legenda
- **→** Chiamata diretta/sincrona
- **⇒** Chiamata asincrona/evento
- **↔** Bidirezionale
- **📥** Input/Trigger
- **📤** Output/Target
- **🔄** Polling/Stream

| Servizio | Input Da | Output Verso | Tipo Connessione | Protocollo | Note |
|----------|----------|--------------|------------------|------------|------|
| **API Gateway** | ALB, CloudFront | Lambda, Step Functions, SQS, DynamoDB | → Sync/Async | HTTPS, WebSocket | Request validation, throttling |
| **Lambda (API)** | API Gateway, ALB | DynamoDB, S3, Step Functions | → Sync | SDK Calls | 29s timeout limit |
| **Lambda (Process)** | SQS, EventBridge, S3 | Textract, Bedrock, OpenSearch | → Sync/Async | SDK/HTTPS | Batch processing |
| **Lambda (Stream)** | DynamoDB Streams, Kinesis | OpenSearch, S3 | 🔄 Polling | Stream API | Parallelization factor |
| **Step Functions** | API Gateway, EventBridge | Lambda, SageMaker, Bedrock, SNS | ⇒ Orchestration | States Language | Express/Standard |
| **SQS** | ITSM, Lambda | Lambda | 🔄 Poll-based | SQS API | Visibility timeout |
| **EventBridge** | Step Functions, S3, Custom | Lambda, SNS, Step Functions | ⇒ Event-driven | Events | Rule-based routing |
| **DynamoDB** | Lambda, API Gateway | Lambda (Streams) | ↔ Read/Write + Stream | SDK | Global tables |
| **S3** | Lambda, Textract, Users | Lambda, Athena, SageMaker | 📥📤 Storage | S3 API | Event notifications |
| **OpenSearch** | Lambda, Kinesis | Lambda, Bedrock | ↔ Index/Search | REST/HTTP | k-NN enabled |
| **Bedrock** | Step Functions, Lambda | CloudWatch | → Inference | SDK | Rate limited |
| **SageMaker** | Step Functions, S3 | Model Registry, CloudWatch | ⇒ Training/Inference | SDK/REST | Async jobs |
| **Textract** | Lambda, S3 | S3, SNS | ⇒ Async OCR | SDK | Job-based |
| **SNS** | EventBridge, Step Functions | Lambda, Email, Webhook | ⇒ Pub/Sub | HTTPS/Email | Fan-out |
| **Cognito** | Users | API Gateway | → Auth | OAuth2/JWT | Token management |
| **CloudWatch** | All Services | SNS, Lambda | 📤 Monitoring | Metrics API | Alarms |
| **Kinesis** | Lambda | OpenSearch, S3 | 🔄 Streaming | Kinesis API | Sharding |
| **Secrets Manager** | - | Lambda, ECS | → Secrets | SDK | Rotation |
| **KMS** | - | S3, DynamoDB, Lambda | → Encryption | SDK | Key policies |

## 🔀 Routing e Load Distribution

### Model Routing Strategy

```yaml
Primary Route (90% traffic):
  Trigger: Step Functions
  Decision: Category-based
  Routes:
    - Technical Issues → SageMaker Classifier → Bedrock Claude
    - Documentation → OpenSearch Only
    - Complex Analysis → Bedrock Claude + RAG
    - Simple FAQ → Cache Layer

Fallback Route (10% traffic):
  Trigger: Error/Timeout
  Routes:
    - Bedrock Claude → Bedrock Llama
    - SageMaker → Bedrock Classifier
    - OpenSearch → DynamoDB Cache
```

### Request Flow Patterns

| Pattern | Path | Latency | Use Case |
|---------|------|---------|----------|
| **Sync Simple** | API → Lambda → DynamoDB | <500ms | Status check |
| **Sync Complex** | API → Lambda → Bedrock → Response | 2-5s | Direct generation |
| **Async Pipeline** | API → SQS → Lambda → Step Functions | 10-60s | Full processing |
| **Stream** | API → Lambda → Kinesis → OpenSearch | Continuous | Real-time indexing |
| **Batch** | EventBridge → Lambda → Batch → S3 | Minutes-Hours | Retraining |

## 🏗️ Deployment Topology

### Multi-AZ Architecture

```
Region: eu-south-1 (Milano)
├── AZ 1 (euw1-az1)
│   ├── Public Subnet:  10.0.1.0/24
│   │   ├── ALB
│   │   └── NAT Gateway
│   ├── Private Subnet: 10.0.11.0/24
│   │   ├── Lambda
│   │   └── ECS Tasks
│   └── Data Subnet:    10.0.21.0/24
│       ├── RDS Primary
│       └── OpenSearch Node 1
│
├── AZ 2 (euw1-az2)
│   ├── Public Subnet:  10.0.2.0/24
│   │   ├── ALB
│   │   └── NAT Gateway
│   ├── Private Subnet: 10.0.12.0/24
│   │   ├── Lambda
│   │   └── ECS Tasks
│   └── Data Subnet:    10.0.22.0/24
│       ├── RDS Standby
│       └── OpenSearch Node 2
│
└── AZ 3 (euw1-az3)
    ├── Public Subnet:  10.0.3.0/24
    │   └── NAT Gateway
    ├── Private Subnet: 10.0.13.0/24
    │   └── Lambda
    └── Data Subnet:    10.0.23.0/24
        └── OpenSearch Node 3
```

## 📈 Traffic Flow Analysis

### Ingress Points

| Entry Point | Traffic Type | Volume/Day | Peak/Hour | Protection |
|-------------|--------------|------------|-----------|------------|
| CloudFront | Static/API | 1M requests | 50K | WAF + Shield |
| API Gateway | REST API | 500K calls | 25K | Throttling |
| API Gateway | WebSocket | 10K connections | 500 | Connection limits |
| S3 Direct | Upload | 1000 files | 100 | Presigned URLs |
| SQS | ITSM Integration | 50K messages | 3K | DLQ |

### Service Communication Matrix

```
         │ Lambda │ DDB │ S3  │ OS  │ Bedrock │ SM  │
─────────┼────────┼─────┼─────┼─────┼─────────┼─────┤
Lambda   │   -    │ 10K │ 5K  │ 3K  │  2K     │ 1K  │ calls/hour
DDB      │  500   │  -  │ 100 │  0  │   0     │  0  │ streams/hour
S3       │  2K    │  0  │  -  │ 500 │   0     │ 200 │ events/hour
OS       │  1K    │  0  │ 100 │  -  │  500    │  0  │ queries/hour
Bedrock  │   0    │ 100 │ 500 │ 1K  │   -     │  0  │ results/hour
SM       │  100   │ 50  │ 300 │  0  │   0     │  -  │ jobs/hour
```

## 🔐 Security Boundaries

### Network Segmentation

| Layer | CIDR | Services | Access Control |
|-------|------|----------|----------------|
| **Public** | 10.0.0.0/20 | ALB, NAT | Security Groups + NACLs |
| **Private** | 10.0.16.0/20 | Lambda, ECS | SG only, no internet |
| **Data** | 10.0.32.0/20 | RDS, OpenSearch | SG + IAM auth |
| **Management** | 10.0.48.0/20 | Bastion, SSM | Session Manager only |

### IAM Role Relationships

```yaml
Roles:
  APIGatewayRole:
    Assumes: Lambda, StepFunctions
    Permissions: Invoke, StartExecution
  
  LambdaExecutionRole:
    Assumes: AWS Lambda Service
    Permissions:
      - DynamoDB: Read/Write
      - S3: Read/Write specific buckets
      - Bedrock: InvokeModel
      - OpenSearch: HTTP calls
      - KMS: Decrypt
      - Secrets: GetSecretValue
  
  StepFunctionsRole:
    Assumes: States Service
    Permissions:
      - Lambda: Invoke
      - SageMaker: CreateTrainingJob
      - Bedrock: InvokeModel
      - SNS: Publish
  
  SageMakerRole:
    Assumes: SageMaker Service
    Permissions:
      - S3: Full on ml-bucket
      - ECR: Pull images
      - CloudWatch: PutMetrics
```

## 🔄 Data Flow Patterns

### Pattern 1: Ticket Processing
```
[ITSM] --HTTPS--> [API Gateway] --sync--> [Lambda Auth]
                          |
                          v
                  [Step Functions]
                    |    |    |
        +-----------+    |    +-----------+
        v                v                v
  [Classify:SM]    [Retrieve:OS]    [Generate:Bedrock]
        |                |                |
        v                v                v
  [DynamoDB]  <---- [Merge Results] ----> [S3 Logs]
                          |
                          v
                    [SNS Notification]
```

### Pattern 2: Knowledge Base Update
```
[S3 Upload] --event--> [Lambda Processor]
                            |
                            v
                      [Textract Job]
                            |
                            v
                    [Lambda Parser]
                            |
                +-----------+-----------+
                v                       v
          [Chunk Text]            [Extract Meta]
                |                       |
                v                       v
          [Bedrock Embed]          [Validate]
                |                       |
                v                       v
          [OpenSearch]             [DynamoDB]
```

### Pattern 3: Model Retraining
```
[EventBridge Cron] --trigger--> [Step Functions Pipeline]
                                          |
                    +---------------------+---------------------+
                    v                                           v
            [Athena Query]                              [Check Drift]
                    |                                           |
                    v                                           v
            [S3 Dataset]                                 [Trigger?:Yes]
                    |                                           |
                    v                                           v
        [SageMaker Pipeline] <----------------------------------+
                    |
        +-----------+-----------+
        v           v           v
    [Train]    [Evaluate]   [Register]
        |           |           |
        v           v           v
    [Artifact]  [Metrics]   [Deploy]
```

## 📊 Performance Optimization Points

### Caching Layers

| Cache Level | Service | TTL | Hit Rate Target | Invalidation |
|-------------|---------|-----|-----------------|--------------|
| **CDN** | CloudFront | 24h | 80% | Manual/API |
| **API** | API Gateway | 5min | 60% | Auto |
| **Application** | ElastiCache | 1h | 70% | LRU |
| **Query** | OpenSearch | 30min | 50% | Index update |
| **Response** | DynamoDB | 7d | 40% | TTL |

### Bottleneck Analysis

| Service | Current Limit | Peak Usage | Scaling Strategy | Cost Impact |
|---------|--------------|------------|------------------|-------------|
| **API Gateway** | 10K req/s | 2K req/s | Auto | Linear |
| **Lambda Concurrent** | 1000 | 300 | Reserved + On-demand | Step function |
| **DynamoDB WCU** | On-demand | 500/s | Auto | Usage-based |
| **OpenSearch** | 3 nodes | 60% CPU | Add nodes | +$120/node |
| **Bedrock Claude** | 100 req/min | 80 req/min | Request increase | Token-based |
| **SageMaker Endpoint** | 10 instances | 8 instances | Auto-scaling | +$50/instance |

## 🚀 Deployment Pipeline

### CI/CD Flow
```
[GitHub] --> [CodePipeline] --> [CodeBuild]
                |                    |
                v                    v
          [Validate]            [Unit Tests]
                |                    |
                v                    v
          [CloudFormation]      [Integration Tests]
                |                    |
                +--------------------+
                          |
                          v
                    [Deploy Dev]
                          |
                          v
                   [Smoke Tests]
                          |
                          v
                  [Deploy Staging]
                          |
                          v
                   [Load Tests]
                          |
                          v
                  [Manual Approval]
                          |
                          v
                   [Deploy Prod]
                          |
                          v
                [Canary Deployment]
                     (10% → 100%)
```

## 📋 Service Dependencies

### Critical Path Dependencies
```
Tier 0 (Foundation):
├── VPC, Subnets, Security Groups
├── IAM Roles and Policies
├── KMS Keys
└── S3 Buckets

Tier 1 (Core Services):
├── DynamoDB Tables
├── Secrets Manager
├── Cognito User Pool
└── CloudWatch Log Groups

Tier 2 (Compute):
├── Lambda Functions
├── API Gateway
├── SQS Queues
└── EventBridge Rules

Tier 3 (ML/AI):
├── SageMaker Endpoints
├── Bedrock Configuration
├── OpenSearch Domain
└── Textract Configuration

Tier 4 (Orchestration):
├── Step Functions
├── SNS Topics
└── CloudWatch Alarms

Tier 5 (Edge):
├── CloudFront Distribution
├── WAF Rules
└── Route 53
```

## 🎯 SLA Targets per Connection

| Connection | SLA Target | Current | Impact if Down |
|------------|------------|---------|----------------|
| API → Lambda | 99.99% | 99.95% | No new requests |
| Lambda → DynamoDB | 99.999% | 99.99% | Data loss |
| Lambda → Bedrock | 99.9% | 99.5% | Fallback to cache |
| Step Functions → All | 99.95% | 99.9% | Manual processing |
| OpenSearch cluster | 99.95% | 99.9% | Degraded search |
| S3 availability | 99.999999999% | ✓ | Critical failure |

---

## 📌 Note Implementative Chiave

1. **Ogni connessione Lambda-to-service deve avere**:
   - Retry logic con exponential backoff
   - Circuit breaker pattern
   - Timeout configurato
   - Error handling con DLQ

2. **Monitoring obbligatorio su**:
   - Ogni integration point
   - Cross-service latency
   - Error rates per connection
   - Token/rate consumption

3. **Security su ogni hop**:
   - TLS in transit
   - IAM per service-to-service
   - VPC endpoints dove possibile
   - Secrets rotation automatica

4. **Cost optimization**:
   - Reserved capacity dove prevedibile
   - Spot instances per training
   - S3 lifecycle policies
   - DynamoDB auto-scaling

---

*Ultimo aggiornamento: 2025-11-06*