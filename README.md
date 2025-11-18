# 🤖 AI Technical Support System

Sistema di assistenza tecnica automatizzata basato su AI/ML per equipaggiamenti industriali (Inverter, EV Charger, HVAC).

## 📋 Quick Start

### Cosa fa questo sistema?

Automatizza il supporto tecnico attraverso:
- **Classificazione** intelligente dei ticket
- **Ricerca** semantica nella knowledge base
- **Generazione** di soluzioni verificabili via LLM
- **Apprendimento** continuo dal feedback operatori

### Stack Tecnologico

- **Cloud**: AWS (Milano - eu-south-1)
- **AI/ML**: Amazon Bedrock, SageMaker, Textract
- **Data**: DynamoDB, OpenSearch, S3
- **Orchestrazione**: Step Functions, Lambda, API Gateway

## 📚 Documentazione

La documentazione è organizzata in moduli tematici nella directory `docs/`:

### 🎯 Getting Started

- **[Overview del Progetto](./docs/01-overview.md)** - Executive summary, obiettivi, stack tecnologico

### 🏗️ Architettura

- **[Principi Architetturali](./docs/02-architecture/overview.md)** - Pattern, componenti, scalabilità
- **[Diagrammi](./docs/02-architecture/diagrams.md)** - Vista d'insieme, data flow, ML pipeline, security
- **[Deployment Topology](./docs/02-architecture/deployment.md)** - Multi-AZ, VPC, CI/CD pipeline
- **[Security](./docs/02-architecture/security.md)** - Network security, IAM, encryption, guardrails

### ☁️ AWS Services

- **[Overview Servizi](./docs/03-aws-services/README.md)** - Mappa completa, costi, limiti, SLA

### 🔄 Data Flows

- **[Ticket Processing](./docs/04-data-flows/ticket-processing.md)** - End-to-end workflow (10 steps)

### 📡 API & Data

- **[API Specification v1](./docs/05-api-specification.md)** - Endpoint REST, autenticazione, esempi
- **[Data Models](./docs/06-data-models.md)** - TypeScript interfaces, DynamoDB schema, OpenSearch mapping

### 💰 Operations

- **[Cost Estimation](./docs/11-cost-estimation.md)** - Breakdown mensile, ottimizzazioni, ROI
- **[Security & Compliance](./docs/10-security-compliance.md)** - GDPR, audit, incident response

### 🚀 Implementation

- **[Roadmap](./docs/12-implementation/roadmap.md)** - 4 fasi, 6 mesi, milestone, metriche
- **[Prompt Templates](./docs/13-prompt-templates.md)** - Prompt engineering per classificazione e RAG

## 🎯 Key Features

### ✅ Implemented

- [x] Infrastructure as Code (CloudFormation)
- [x] API Gateway + Cognito authentication
- [x] Step Functions orchestration
- [x] Bedrock LLM integration (Claude 3)
- [x] OpenSearch knowledge base
- [x] SageMaker classification endpoint
- [x] Textract OCR pipeline
- [x] CloudWatch monitoring

### 🚧 In Progress

- [ ] Advanced RAG (re-ranking, hybrid search)
- [ ] MLOps automation (drift detection, retraining)
- [ ] SSE streaming responses
- [ ] Multi-language support

### 📋 Planned

- [ ] Voice/chat integration
- [ ] Predictive maintenance
- [ ] Playbook automation
- [ ] Mobile app

## 📊 Current Status

**Versione**: 1.0 (MVP)
**Ambiente**: Staging
**Branch**: `claude/reorganize-project-structure-014BdNE8jki34Ce7UPTLEjdB`

### Metrics (Last 30 days)

| Metric | Value | Target | Status |
|--------|-------|--------|--------|
| **Uptime** | 99.2% | 99.5% | 🟡 |
| **Avg Latency (P95)** | 2.8s | < 3s | ✅ |
| **Classification F1** | 0.91 | > 0.90 | ✅ |
| **Groundedness** | 0.86 | > 0.85 | ✅ |
| **Resolution Rate** | 48% | > 50% | 🟡 |

## 🚀 Quick Deploy

### Prerequisites

- AWS Account con accesso a Bedrock
- AWS CLI configurato
- Node.js 18+ (per CDK)
- Python 3.11+

### Deploy Infrastructure

```bash
# Install dependencies
npm install -g aws-cdk
pip install -r requirements.txt

# Bootstrap CDK (first time only)
cdk bootstrap aws://ACCOUNT-ID/eu-south-1

# Deploy all stacks
cdk deploy --all

# Deploy specific environment
cdk deploy -c env=staging
```

### Configure Secrets

```bash
# Create OpenSearch credentials
aws secretsmanager create-secret \
  --name prod/opensearch/credentials \
  --secret-string '{"username":"admin","password":"STRONG_PASSWORD"}'

# Create RDS credentials (if using)
aws secretsmanager create-secret \
  --name prod/rds/credentials \
  --secret-string '{"username":"app_user","password":"STRONG_PASSWORD"}'
```

### Deploy Application

```bash
# Package Lambda functions
./scripts/package-lambdas.sh

# Upload to S3
aws s3 sync ./build s3://deployment-bucket/artifacts/

# Deploy via CodePipeline
aws codepipeline start-pipeline-execution \
  --name ai-support-pipeline
```

## 🧪 Testing

### Run Unit Tests

```bash
# Python tests
pytest tests/unit/ -v --cov=src

# Integration tests (requires deployed stack)
pytest tests/integration/ -v
```

### Load Testing

```bash
# Install Locust
pip install locust

# Run load test (1000 users, 100/sec ramp)
locust -f tests/load/locustfile.py \
  --host https://api.example.com \
  --users 1000 \
  --spawn-rate 100
```

## 📞 Support

### Team Contacts

- **Tech Lead**: [Nome] - tech.lead@example.com
- **Product Owner**: [Nome] - product@example.com
- **AWS TAM**: [Nome] - aws-tam@example.com

### Channels

- **Slack**: #ai-tech-support
- **Jira**: [Project Board](https://jira.example.com/ai-support)
- **Confluence**: [Wiki](https://confluence.example.com/ai-support)
- **PagerDuty**: On-call rotation

### Incident Response

```
P1 (Critical): Page on-call immediately
P2 (High): Slack alert + email
P3 (Medium): Ticket only
P4 (Low): Backlog
```

## 🔒 Security

### Reporting Vulnerabilities

**DO NOT** open public issues for security vulnerabilities.

Email: security@example.com (PGP key available)

### Compliance

- ✅ GDPR compliant (data residency EU)
- ✅ SOC 2 Type II (in progress)
- ✅ ISO 27001 (planned Q2 2025)

## 📄 License

Proprietary - Internal use only

---

## 🗂️ Project Structure

```
ai-service/
├── README.md                    # This file
├── docs/                        # Documentation modules
│   ├── 01-overview.md
│   ├── 02-architecture/
│   ├── 03-aws-services/
│   ├── 04-data-flows/
│   ├── 05-api-specification.md
│   ├── 06-data-models.md
│   ├── 10-security-compliance.md
│   ├── 11-cost-estimation.md
│   ├── 12-implementation/
│   └── 13-prompt-templates.md
├── src/                         # Source code (Lambda, Step Functions)
├── infrastructure/              # CloudFormation / CDK
├── tests/                       # Unit, integration, load tests
├── scripts/                     # Deployment, maintenance scripts
└── .github/                     # CI/CD workflows
```

## 📖 Documentation Index

### By Role

**For Product Managers**:
- [Overview](./docs/01-overview.md)
- [API Specification](./docs/05-api-specification.md)
- [Cost Estimation](./docs/11-cost-estimation.md)
- [Roadmap](./docs/12-implementation/roadmap.md)

**For Developers**:
- [Architecture](./docs/02-architecture/overview.md)
- [Data Models](./docs/06-data-models.md)
- [Data Flows](./docs/04-data-flows/ticket-processing.md)
- [AWS Services](./docs/03-aws-services/README.md)

**For ML Engineers**:
- [Prompt Templates](./docs/13-prompt-templates.md)
- [SageMaker Setup](./docs/03-aws-services/sagemaker.md)
- [Bedrock Integration](./docs/03-aws-services/bedrock.md)

**For DevOps**:
- [Deployment](./docs/02-architecture/deployment.md)
- [Security](./docs/02-architecture/security.md)
- [Monitoring](./docs/09-operations/monitoring.md)

**For Compliance/Legal**:
- [Security & Compliance](./docs/10-security-compliance.md)
- [Data Models](./docs/06-data-models.md#gdpr-compliance)

---

**Last Updated**: 2025-11-18
**Version**: 1.1
**Status**: ACTIVE DEVELOPMENT
