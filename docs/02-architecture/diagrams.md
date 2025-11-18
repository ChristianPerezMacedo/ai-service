# Diagrammi Architettura

Questa sezione contiene i diagrammi Mermaid che rappresentano visualmente l'architettura del sistema.

## Vista d'Insieme - Tutti i Servizi

```mermaid
graph TB
    %% External Layer
    subgraph "🌐 External"
        USER[fa:fa-users Users]
        ITSM[fa:fa-ticket ITSM System]
        WIKI[fa:fa-book Corporate Wiki]
    end

    %% Entry Points
    subgraph "🚪 Entry Points"
        CF[CloudFront CDN]
        R53[Route 53 DNS]
        ALB[Application Load Balancer]
        APIGW[API Gateway]
    end

    %% Auth & Security
    subgraph "🔐 Auth & Security"
        COG[Cognito User Pools]
        WAF[AWS WAF]
        SEC[Secrets Manager]
        KMS[KMS Encryption]
    end

    %% Compute Layer
    subgraph "⚙️ Compute"
        LAMBDA1[Lambda: API Handlers]
        LAMBDA2[Lambda: Processors]
        LAMBDA3[Lambda: Validators]
        SFN[Step Functions]
        BATCH[AWS Batch]
    end

    %% ML/AI Services
    subgraph "🤖 ML/AI Services"
        BEDROCK[Amazon Bedrock]
        SM_END[SageMaker Endpoints]
        SM_PIPE[SageMaker Pipelines]
        SM_REG[Model Registry]
        TEXTRACT[Amazon Textract]
        COMPREHEND[Comprehend Medical]
    end

    %% Storage & Database
    subgraph "💾 Storage & Database"
        S3_RAW[S3: Raw Data]
        S3_PROC[S3: Processed]
        S3_MODEL[S3: Models]
        DDB_STATE[DynamoDB: State]
        DDB_IDEM[DynamoDB: Idempotency]
        OS[OpenSearch Domain]
        RDS[RDS PostgreSQL]
    end

    %% Integration & Messaging
    subgraph "📬 Integration"
        SQS1[SQS: Ticket Queue]
        SQS2[SQS: DLQ]
        SNS[SNS Topics]
        EB[EventBridge]
        KINESIS[Kinesis Data Streams]
    end

    %% Monitoring
    subgraph "📈 Monitoring"
        CW[CloudWatch]
        XRAY[X-Ray Tracing]
        CT[CloudTrail]
    end

    %% Connections
    USER --> CF
    CF --> ALB
    ALB --> APIGW
    R53 --> CF

    APIGW --> WAF
    WAF --> LAMBDA1
    LAMBDA1 --> COG

    LAMBDA1 --> SFN
    SFN --> LAMBDA2
    SFN --> LAMBDA3

    ITSM --> SQS1
    SQS1 --> LAMBDA2
    WIKI --> S3_RAW

    LAMBDA2 --> TEXTRACT
    TEXTRACT --> S3_PROC

    SFN --> SM_END
    SFN --> BEDROCK

    SM_PIPE --> SM_REG
    SM_REG --> SM_END

    LAMBDA2 --> DDB_STATE
    LAMBDA1 --> DDB_IDEM

    S3_PROC --> OS
    BEDROCK --> OS

    SFN --> EB
    EB --> SNS
    SNS --> LAMBDA3

    LAMBDA2 --> KINESIS
    KINESIS --> CW

    LAMBDA1 --> XRAY
    LAMBDA2 --> XRAY
    SFN --> CW

    SEC --> LAMBDA1
    KMS --> S3_RAW
    KMS --> DDB_STATE

    SM_END --> CW
    BEDROCK --> CW

    BATCH --> SM_PIPE

    DDB_STATE --> RDS

    SQS1 --> SQS2

    style USER fill:#e8f5e9
    style ITSM fill:#e8f5e9
    style WIKI fill:#e8f5e9
    style BEDROCK fill:#fff3e0
    style SM_END fill:#fff3e0
    style SFN fill:#e3f2fd
    style DDB_STATE fill:#fce4ec
    style OS fill:#fce4ec
    style S3_RAW fill:#fce4ec
```

## Flusso Dati Dettagliato

```mermaid
graph LR
    subgraph "📥 Ingestion"
        TICKET[New Ticket]
        DOC[Document Upload]
        LOG[System Logs]
    end

    subgraph "🔄 Processing Pipeline"
        VAL{Validation}
        ENRICH[Data Enrichment]
        OCR[OCR Processing]
        CLEAN[Cleaning]
        CHUNK[Chunking]
        EMBED[Embedding]
    end

    subgraph "💾 Storage Layers"
        BRONZE[(Bronze: Raw)]
        SILVER[(Silver: Cleaned)]
        GOLD[(Gold: Analytics)]
    end

    subgraph "🎯 Destination"
        KB[Knowledge Base]
        TRAIN[Training Data]
        CACHE[Response Cache]
    end

    TICKET --> VAL
    DOC --> VAL
    LOG --> VAL

    VAL -->|Valid| ENRICH
    VAL -->|Invalid| DLQ[Dead Letter Queue]

    ENRICH --> BRONZE
    BRONZE --> OCR
    OCR --> CLEAN
    CLEAN --> SILVER
    SILVER --> CHUNK
    CHUNK --> EMBED
    EMBED --> GOLD

    GOLD --> KB
    GOLD --> TRAIN
    GOLD --> CACHE

    style BRONZE fill:#cd7f32
    style SILVER fill:#c0c0c0
    style GOLD fill:#ffd700
```

## Pipeline Machine Learning

```mermaid
graph TB
    subgraph "📊 Data Preparation"
        HIST[Historical Tickets]
        LABEL[Manual Labels]
        AUG[Data Augmentation]
    end

    subgraph "🔬 Training Pipeline"
        SPLIT[Train/Val/Test Split]
        FE[Feature Engineering]
        TRAIN_CLS[Train Classifier]
        TRAIN_EMB[Train Embeddings]
        EVAL[Evaluation]
        REG[Model Registry]
    end

    subgraph "🚀 Deployment"
        STAGING[Staging Endpoint]
        CANARY[Canary Deploy]
        PROD[Production Endpoint]
        AB[A/B Testing]
    end

    subgraph "🔄 Continuous Learning"
        FEEDBACK[User Feedback]
        MONITOR[Performance Monitor]
        DRIFT[Drift Detection]
        RETRAIN[Auto Retrain]
    end

    HIST --> SPLIT
    LABEL --> SPLIT
    AUG --> SPLIT

    SPLIT --> FE
    FE --> TRAIN_CLS
    FE --> TRAIN_EMB

    TRAIN_CLS --> EVAL
    TRAIN_EMB --> EVAL

    EVAL -->|Pass| REG
    EVAL -->|Fail| FE

    REG --> STAGING
    STAGING --> CANARY
    CANARY --> AB
    AB --> PROD

    PROD --> MONITOR
    FEEDBACK --> MONITOR
    MONITOR --> DRIFT
    DRIFT -->|Detected| RETRAIN
    RETRAIN --> SPLIT

    style REG fill:#4caf50
    style PROD fill:#2196f3
    style DRIFT fill:#ff9800
```

## Security & Network Architecture

```mermaid
graph TB
    subgraph "🌍 Internet"
        INET[Internet Traffic]
    end

    subgraph "🛡️ Edge Security"
        SHIELD[AWS Shield]
        CFWAF[CloudFront + WAF]
    end

    subgraph "🏰 VPC Architecture"
        subgraph "Public Subnets"
            NAT[NAT Gateway]
            ALBPUB[ALB]
        end

        subgraph "Private Subnets"
            LAMBDA_PR[Lambda Functions]
            ECS[ECS Fargate]
        end

        subgraph "Data Subnets"
            RDS_PR[RDS Multi-AZ]
            OS_PR[OpenSearch]
            CACHE[ElastiCache]
        end
    end

    subgraph "🔑 Security Services"
        IAM[IAM Roles]
        SSM[Systems Manager]
        GUARD[GuardDuty]
        MACIE[Macie]
    end

    subgraph "🔐 Encryption"
        KMS_DATA[KMS Data Keys]
        KMS_APP[KMS App Keys]
        ACM[ACM Certificates]
    end

    INET --> SHIELD
    SHIELD --> CFWAF
    CFWAF --> ALBPUB

    ALBPUB --> LAMBDA_PR
    LAMBDA_PR --> NAT
    NAT --> INET

    LAMBDA_PR --> RDS_PR
    LAMBDA_PR --> OS_PR
    LAMBDA_PR --> CACHE

    IAM --> LAMBDA_PR
    SSM --> ECS

    GUARD --> LAMBDA_PR
    MACIE --> RDS_PR

    KMS_DATA --> RDS_PR
    KMS_DATA --> OS_PR
    KMS_APP --> LAMBDA_PR
    ACM --> ALBPUB

    style SHIELD fill:#ff5722
    style CFWAF fill:#ff9800
    style IAM fill:#4caf50
    style KMS_DATA fill:#9c27b0
```

## Pipeline Ticket Processing

```mermaid
graph LR
    A[Nuovo Ticket] --> B[Classificazione]
    B --> C{Categoria?}
    C -->|EV Charger| D1[Workflow EV]
    C -->|Inverter| D2[Workflow Inverter]
    C -->|HVAC| D3[Workflow HVAC]
    D1 --> E[RAG Retrieval]
    D2 --> E
    D3 --> E
    E --> F[LLM Generation]
    F --> G[Guardrails Check]
    G --> H{Sicuro?}
    H -->|Sì| I[Pubblica Soluzione]
    H -->|No| J[Flag per Revisione]
    I --> K[Update ITSM]
    J --> K
```

## Pipeline Knowledge Base

```mermaid
graph LR
    A[Upload Document] --> B[S3 Event]
    B --> C[Lambda Trigger]
    C --> D{File Type?}
    D -->|PDF| E[Textract OCR]
    D -->|HTML| F[HTML Parser]
    D -->|TXT| G[Direct Read]
    E --> H[Text Extraction]
    F --> H
    G --> H
    H --> I[Chunking Strategy]
    I --> J[Metadata Extraction]
    J --> K[Bedrock Embeddings]
    K --> L[OpenSearch Index]
    L --> M[Validation Check]
    M --> N{Quality OK?}
    N -->|Yes| O[Activate Index]
    N -->|No| P[Manual Review]
```

## Continuous Learning Loop

```mermaid
graph TB
    A[Production Inference] --> B[Collect Feedback]
    B --> C[Store Labels]
    C --> D{Enough Data?}
    D -->|No| A
    D -->|Yes| E[Prepare Dataset]
    E --> F[Train New Model]
    F --> G[Evaluate Metrics]
    G --> H{Better than Current?}
    H -->|No| I[Discard]
    H -->|Yes| J[Register Model]
    J --> K[Canary Deploy]
    K --> L{Monitor Metrics}
    L -->|Degraded| M[Rollback]
    L -->|Improved| N[Full Rollout]
    M --> A
    N --> A
```

## Riferimenti

- [Overview Architettura](./overview.md)
- [Deployment Topology](./deployment.md)
- [Security Architecture](./security.md)
