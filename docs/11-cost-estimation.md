# Stima Costi AWS

## Costi Mensili per Ambiente

### Production Environment

Basato su **1,000 ticket/mese** elaborati.

| Servizio | Configurazione | Volume Mensile | Costo Unitario | Costo Totale |
|----------|---------------|----------------|----------------|--------------|
| **S3 Storage** | Standard + IA | 100 GB | $0.023/GB | $23 |
| **S3 Requests** | GET/PUT | 1M GET, 100K PUT | $0.0004/1K + $0.005/1K | $5 |
| **Amazon Textract** | Document Analysis | 10,000 pagine | $0.015/pagina | $150 |
| **OpenSearch** | 3x r6g.large.search | 24/7 | $0.166/ora x 3 | $360 |
| **SageMaker Endpoint** | 2x ml.m5.large | 24/7 | $0.138/ora x 2 | $200 |
| **Bedrock - Claude 3** | Input/Output tokens | 10M in, 5M out | $0.003/1K in, $0.015/1K out | $105 |
| **Bedrock - Titan Embeddings** | Embedding tokens | 50M tokens | $0.0001/1K | $5 |
| **Step Functions** | Standard workflows | 50,000 executions | $0.025/1K | $13 |
| **Lambda** | Compute | 5M invocations, 15M GB-sec | $0.20/1M + $0.0000166667/GB-sec | $41 |
| **API Gateway** | REST API | 1M requests | $3.50/M | $35 |
| **DynamoDB** | On-demand | 10GB, 500K WCU, 1M RCU | $1.25/GB + usage | $20 |
| **EventBridge** | Custom events | 100,000 events | $1.00/M | $10 |
| **SNS** | Notifications | 50,000 notifications | $0.50/M | $5 |
| **SQS** | Standard queue | 1M requests | $0.40/M | $5 |
| **CloudWatch Logs** | Ingestion + Storage | 50GB ingestion, 100GB stored | $0.50/GB + $0.03/GB | $28 |
| **CloudWatch Metrics** | Custom metrics | 100 metrics | $0.30/metric | $30 |
| **NAT Gateway** | Data transfer | 100GB | $0.045/GB + $0.045/hour | $80 |
| **Data Transfer Out** | Internet egress | 500GB | $0.09/GB | $45 |
| **KMS** | Key usage | 100,000 requests | $1.00/M | $10 |
| **Secrets Manager** | Secrets stored | 10 secrets | $0.40/secret | $4 |
| **X-Ray** | Tracing | 1M traces | $5.00/M | $5 |
| **VPC** | Endpoints | 5 interface endpoints | $0.01/hour x 5 | $36 |
| | | | **TOTALE MENSILE** | **~$1,215** |

### Staging Environment

Costo ridotto ~40% rispetto a production:

| Categoria | Prod | Staging | Note |
|-----------|------|---------|------|
| **OpenSearch** | $360 | $120 | 1x t3.medium invece di 3x r6g.large |
| **SageMaker** | $200 | $70 | 1x ml.t3.medium |
| **Lambda/API** | $76 | $30 | ~40% traffico |
| **Bedrock** | $110 | $30 | Testing limitato |
| **Altro** | $469 | $250 | Proporzione ridotta |
| **TOTALE** | $1,215 | **~$500** | |

### Development Environment

Costo minimo per sviluppo e test:

- OpenSearch: 1x t3.small ($30)
- Lambda/API: Pay-per-use (~$20)
- S3 + DynamoDB: ~$10
- Bedrock: Testing spot (~$20)
- **TOTALE: ~$80/mese**

## Breakdown per Categoria

```
┌─────────────────────────────────────────────┐
│ AI/ML Services:          $470 (39%)         │
│ ├─ Bedrock:             $110                │
│ ├─ SageMaker:           $200                │
│ ├─ Textract:            $150                │
│ └─ Comprehend:          $10                 │
├─────────────────────────────────────────────┤
│ Compute & API:           $289 (24%)         │
│ ├─ Lambda:              $41                 │
│ ├─ API Gateway:         $35                 │
│ ├─ Step Functions:      $13                 │
│ └─ NAT Gateway:         $80                 │
├─────────────────────────────────────────────┤
│ Data Storage:            $403 (33%)         │
│ ├─ OpenSearch:          $360                │
│ ├─ DynamoDB:            $20                 │
│ ├─ S3:                  $23                 │
│ └─ RDS (future):        $0                  │
├─────────────────────────────────────────────┤
│ Monitoring:              $63 (5%)           │
│ ├─ CloudWatch:          $58                 │
│ └─ X-Ray:               $5                  │
├─────────────────────────────────────────────┤
│ Networking:              $161 (13%)         │
│ ├─ NAT Gateway:         $80                 │
│ ├─ Data Transfer:       $45                 │
│ └─ VPC Endpoints:       $36                 │
├─────────────────────────────────────────────┤
│ Security:                $14 (1%)           │
│ ├─ KMS:                 $10                 │
│ └─ Secrets Manager:     $4                  │
└─────────────────────────────────────────────┘
```

## Scalabilità dei Costi

### Scenario di Crescita

| Ticket/Mese | Costo Totale | Costo/Ticket | Note |
|-------------|--------------|--------------|------|
| **100** | $850 | $8.50 | Minimum viable |
| **500** | $1,050 | $2.10 | Current estimate |
| **1,000** | $1,215 | $1.22 | Target |
| **5,000** | $2,400 | $0.48 | Economy of scale |
| **10,000** | $3,800 | $0.38 | Full optimization |

**Componenti a costo fisso**:
- OpenSearch: $360 (fisso fino a 10K ticket)
- SageMaker Endpoint: $200 (fisso)
- NAT Gateway: $80 (base)
- VPC Endpoints: $36 (fisso)

**Componenti variabili**:
- Bedrock tokens: scala linearmente
- Lambda: scala con invocations
- DynamoDB: on-demand scaling
- S3: storage incrementale

## Ottimizzazione Costi

### Quick Wins (risparmio immediato)

1. **Reserved Instances** (-35%)
   - OpenSearch: $360 → $235/mese (1 year RI)
   - Risparmio: $125/mese = $1,500/anno

2. **Savings Plans** (-20%)
   - SageMaker: $200 → $160/mese (1 year)
   - Risparmio: $40/mese = $480/anno

3. **S3 Intelligent-Tiering** (-30%)
   - S3 Storage: $23 → $16/mese
   - Risparmio: $7/mese = $84/anno

4. **CloudWatch Logs Retention** (-40%)
   - Ridurre retention da 30 a 7 giorni: $28 → $17/mese
   - Risparmio: $11/mese = $132/anno

**Risparmio totale Quick Wins: ~$2,200/anno (18%)**

### Mid-term Optimizations

1. **Bedrock Provisioned Throughput** (per high volume)
   - Se >1M token/giorno: provisioned model units
   - Risparmio potenziale: 30-50%

2. **Lambda Provisioned Concurrency** (selective)
   - Solo per critical paths
   - Trade-off latency vs cost

3. **OpenSearch Auto-Scaling**
   - Scale down durante off-hours (se applicabile)
   - Risparmio potenziale: 20-30%

4. **VPC Endpoint consolidation**
   - Rimuovere endpoint poco usati
   - Risparmio: $7/endpoint/mese

### Long-term Optimizations

1. **Model Self-Hosting**
   - Deploy open-source LLM su SageMaker
   - Cost: $800/mese (ml.g5.xlarge 24/7)
   - Break-even a ~5,000 ticket/mese

2. **Caching Aggressivo**
   - ElastiCache per risposte frequenti
   - Cost: $50/mese
   - Risparmio Bedrock: potenziale 20-40%

3. **Batch Processing**
   - Accumulate e processa in batch (trade-off latency)
   - Risparmio Lambda/API: 15-25%

## Breakdown per Ticket

### Costo Medio per Ticket Elaborato

Assumendo 1,000 ticket/mese:

| Fase | Servizi | Costo |
|------|---------|-------|
| **Ingestion** | API Gateway, Lambda, DynamoDB | $0.08 |
| **OCR** | Textract (1 page avg) | $0.15 |
| **Classification** | SageMaker inference | $0.20 |
| **RAG Retrieval** | OpenSearch, Titan embeddings | $0.36 |
| **Generation** | Bedrock Claude (2K tokens) | $0.11 |
| **Storage** | DynamoDB, S3 | $0.03 |
| **Monitoring** | CloudWatch, X-Ray | $0.06 |
| **Networking** | Data transfer | $0.04 |
| | **TOTALE PER TICKET** | **~$1.03** |

**Fixed costs ammortizzati**: $0.19/ticket
**TOTALE**: $1.22/ticket

## Confronto Alternative

### Cloud Provider Comparison

| Provider | Stack Equivalente | Costo/Mese | Note |
|----------|------------------|------------|------|
| **AWS (current)** | Bedrock + SageMaker | $1,215 | Migliore integrazione |
| **Azure** | OpenAI + ML Studio | $1,450 | +19% costo |
| **GCP** | Vertex AI + PaLM | $1,350 | +11% costo |
| **Hybrid** | AWS + OpenAI API | $1,100 | -9% ma compliance issue |

### LLM Provider Comparison

| Provider | Model | Cost/1M tokens | Monthly (5M tokens) |
|----------|-------|----------------|---------------------|
| **Bedrock Claude** | claude-3-sonnet | $3/1M in, $15/1M out | $105 |
| **OpenAI** | gpt-4-turbo | $10/1M in, $30/1M out | $200 |
| **Azure OpenAI** | gpt-4 | $10/1M in, $30/1M out | $200 |
| **Self-hosted Llama** | llama-2-70b | Compute only | ~$800 (infra) |

**Winner**: Bedrock per costi e compliance (data in EU)

## Budget e Allarmi

### Budget Mensili

```
Development:    $100   (alert @ $80)
Staging:        $600   (alert @ $500)
Production:     $1,500 (alert @ $1,300)
```

### CloudWatch Alarms

```yaml
Alarms:
  - Name: MonthlyBudgetExceeded
    Threshold: $1,300
    Action: SNS → Email + Lambda (scale down non-critical)

  - Name: DailySpendAnomaly
    Threshold: 2x daily average
    Action: SNS alert

  - Name: BedrocktokenOverage
    Threshold: 500K tokens/day
    Action: Rate limiting

  - Name: OpenSearchHighCPU
    Threshold: >80% for 30min
    Action: Consider scaling (cost alert)
```

### Cost Allocation Tags

```
Environment: [dev, staging, prod]
Service: [api, ml, storage, monitoring]
Team: [engineering, ml-ops, data]
CostCenter: [RD-001, OPS-002]
```

## ROI Analysis

### Costo vs Alternativa Manuale

**Scenario**: 1,000 ticket/mese

| Metodo | Costo Mensile | Tempo/Ticket | Note |
|--------|---------------|--------------|------|
| **Manuale (L1 support)** | $15,000 | 30 min | $30/hour × 500 hours |
| **AI-Assisted (current)** | $1,215 + $5,000 | 5 min | Operator reviews AI |
| **Fully Automated** | $1,215 | 2 min | 60% auto-resolution |

**Savings**:
- AI-assisted: $9,000/mese (60% saving)
- Fully automated: $13,785/mese (92% saving su auto-resolved)

**Payback Period**: 2 mesi di sviluppo

## Riferimenti

- [Architecture](./02-architecture/overview.md)
- [AWS Services](./03-aws-services/README.md)
- [AWS Pricing Calculator](https://calculator.aws)
