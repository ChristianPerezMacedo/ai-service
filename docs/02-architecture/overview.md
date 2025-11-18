# Architettura di Sistema

## Principi Architetturali

Il sistema è progettato secondo 5 principi fondamentali:

### 1. Separazione dei Concern
L'architettura è divisa in tre piani distinti:

- **Data Plane**: Storage e gestione dati (S3, DynamoDB, OpenSearch, RDS)
- **Orchestration Plane**: Coordinamento workflow (Step Functions, EventBridge)
- **Model Plane**: Servizi AI/ML (Bedrock, SageMaker, Textract)

Questa separazione garantisce:
- Scalabilità indipendente di ogni piano
- Testing e deployment isolati
- Chiara ownership dei componenti
- Facilità di manutenzione

### 2. LLM-Agnostico
Il sistema non è legato a un modello specifico ma supporta:

- **Astrazione modelli**: Interface comune per tutti i provider
- **Routing intelligente**: Selezione modello basata su task e costi
- **Fallback automatico**: Switch su modello alternativo in caso di errore
- **Multi-model ensemble**: Combinazione risposte da più modelli

Vantaggi:
- Flessibilità nella scelta del provider
- Migrazione semplice tra modelli
- Ottimizzazione costi con modelli specializzati
- Resilienza a failure di singoli provider

### 3. RAG-First (Retrieval Augmented Generation)
Tutte le risposte sono basate su knowledge base esterna:

- **Knowledge Base centralizzata**: Documentazione ufficiale in OpenSearch
- **Retrieval semantico**: Ricerca vettoriale con embedding
- **Grounding**: LLM risponde SOLO basandosi su contesto recuperato
- **Citation**: Ogni affermazione citata con fonte

Benefici:
- **Precisione**: Riduzione allucinazioni dell'80%+
- **Tracciabilità**: Audit trail completo
- **Aggiornabilità**: Modifica KB senza retraining
- **Compliance**: Verificabilità delle risposte

### 4. MLOps Integrato
Pipeline automatizzate per tutto il ciclo di vita ML:

- **Data versioning**: Tracciamento dataset con hash e metadata
- **Model versioning**: Registry centralizzato con lineage
- **Automated training**: Trigger basati su drift detection
- **Canary deployment**: Rollout graduale con monitoring
- **Auto-rollback**: Ripristino automatico se metriche degradano

Componenti:
- SageMaker Pipelines per orchestrazione
- Model Registry per governance
- CloudWatch per monitoring continuo
- EventBridge per trigger automatici

### 5. Async by Default
Operazioni lunghe gestite in modo asincrono:

- **Fire-and-forget**: Client riceve 202 Accepted immediatamente
- **Polling endpoint**: `/tickets/{id}` per stato e risultati
- **Webhook callback**: Notifica opzionale al completamento
- **SSE streaming**: Streaming in tempo reale per risposte lunghe

Pattern implementati:
- Step Functions per workflow complessi
- SQS per buffering richieste
- EventBridge per notifiche
- DynamoDB Streams per change data capture

## Componenti Principali

### Client Layer
```
┌─────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                          │
│  Web UI │ Mobile │ ITSM Integration │ API Clients           │
└────────────────┬────────────────────────────────────────────┘
```

Interfacce verso il sistema:
- **Web UI**: Dashboard per operatori NOC
- **Mobile App**: Accesso field technicians
- **ITSM Integration**: Connettori per ServiceNow, Jira Service Desk
- **API Clients**: Integrazioni custom via REST API

### API Gateway Layer
```
┌────────────────▼────────────────────────────────────────────┐
│                      API GATEWAY (v1)                        │
│  • Auth (Cognito/OAuth2)                                     │
│  • Rate Limiting                                             │
│  • Request Validation                                        │
└────────────────┬────────────────────────────────────────────┘
```

Responsabilità:
- Autenticazione e autorizzazione (Cognito)
- Rate limiting per prevenire abuse
- Validazione payload con JSON Schema
- Request/response transformation
- API versioning

### Orchestration Layer
```
┌────────────────▼────────────────────────────────────────────┐
│              ORCHESTRATION (Step Functions)                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│  │Classify  │→│Retrieve  │→│Generate  │→│Validate  │      │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘      │
└────────────────┬────────────────────────────────────────────┘
```

Step Functions coordina 4 fasi:
1. **Classify**: Categorizzazione ticket via SageMaker
2. **Retrieve**: Ricerca KB via OpenSearch
3. **Generate**: Creazione risposta via Bedrock
4. **Validate**: Guardrails e safety check

### Service Layer
```
┌────────────────▼────────────────────────────────────────────┐
│                    SERVICE LAYER                             │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │  Classification  │  │   Generation    │                  │
│  │  • SageMaker    │  │   • Bedrock     │                  │
│  │  • BlazingText  │  │   • Claude/Llama│                  │
│  └─────────────────┘  └─────────────────┘                  │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │   Knowledge     │  │   Document      │                  │
│  │   • OpenSearch  │  │   • Textract    │                  │
│  │   • Bedrock KB  │  │   • S3 Storage  │                  │
│  └─────────────────┘  └─────────────────┘                  │
└──────────────────────────────────────────────────────────────┘
```

Servizi specializzati:
- **Classification**: SageMaker endpoint per categorizzazione
- **Generation**: Bedrock con Claude 3 / Llama 2
- **Knowledge**: OpenSearch con k-NN plugin
- **Document**: Textract per OCR e estrazione

## Data Flow

### Flusso Sincrono (Semplice)
```
Client → API Gateway → Lambda → DynamoDB → Response
Latency: < 500ms
```

Usato per:
- Status check ticket
- Ricerca KB diretta
- Operazioni CRUD semplici

### Flusso Asincrono (Complesso)
```
Client → API Gateway → SQS → Lambda → Step Functions → Services
                                            ↓
                                      DynamoDB (status)
                                            ↓
Client ← EventBridge ← SNS ← Step Functions (completed)
```

Usato per:
- Processing ticket completi
- Retraining modelli
- Batch operations

### Flusso Streaming
```
Client → API Gateway → Lambda → Bedrock (streaming)
   ↑──────────────────────────────┘
   (Server-Sent Events)
```

Usato per:
- Generazione risposte in tempo reale
- Feedback progressivo all'utente

## Scalabilità

### Horizontal Scaling
Componenti che scalano orizzontalmente:
- **Lambda**: Concurrency automatica fino a 1000
- **DynamoDB**: On-demand capacity mode
- **API Gateway**: Managed service, auto-scaling
- **OpenSearch**: Aggiunta nodi al cluster

### Vertical Scaling
Componenti che scalano verticalmente:
- **SageMaker endpoints**: Instance type upgrade
- **OpenSearch nodes**: Instance size increase
- **RDS**: Instance class upgrade

### Limiti e Quota
- **Bedrock**: 100 req/min (richiedibile aumento)
- **Lambda concurrent executions**: 1000 (soft limit)
- **API Gateway**: 10K req/sec (soft limit)
- **Step Functions**: 25K simultaneous executions

## Resilienza

### High Availability
- **Multi-AZ deployment**: Tutti i servizi in 3 AZ
- **Load balancing**: ALB distribuisce traffico
- **Auto-healing**: Lambda e ECS auto-restart
- **Health checks**: CloudWatch allarmi su failure

### Disaster Recovery
- **Backup**: Point-in-time recovery DynamoDB
- **Snapshots**: OpenSearch snapshot daily su S3
- **Cross-region replication**: S3 replication attiva
- **RTO target**: 4 ore
- **RPO target**: 1 ora

### Fault Tolerance
- **Retry logic**: Exponential backoff su tutte le chiamate
- **Circuit breaker**: Protezione su servizi esterni
- **Dead Letter Queue**: SQS DLQ per messaggi failed
- **Graceful degradation**: Fallback su cache in caso di failure

## Performance

### Latency Targets
- **API read operations**: < 200ms (p95)
- **Classification**: < 500ms
- **RAG retrieval**: < 800ms
- **Full generation**: < 3s (p95)

### Throughput Targets
- **API requests**: 1000 req/sec sustained
- **Ticket processing**: 50 ticket/min
- **KB indexing**: 100 docs/min

### Optimization Strategy
- **Caching multi-livello**: CloudFront, API Gateway, ElastiCache
- **Connection pooling**: Lambda con keep-alive
- **Batch operations**: DynamoDB batch write
- **Parallel execution**: Step Functions parallel states

## Riferimenti

- [Diagrammi Architettura](./diagrams.md)
- [Deployment Topology](./deployment.md)
- [Security Architecture](./security.md)
- [Servizi AWS](../03-aws-services/README.md)
