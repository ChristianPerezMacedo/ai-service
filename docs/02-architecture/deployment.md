# Deployment Topology

## Multi-AZ Architecture

### Regione e Availability Zones

**Regione AWS**: `eu-south-1` (Milano)
- **Motivo**: Compliance GDPR, latenza ridotta per utenti EU
- **Backup Region**: `eu-west-1` (Irlanda) per disaster recovery
- **Availability Zones**: 3 AZ attive per high availability

### Topologia di Rete

```
Region: eu-south-1 (Milano)
├── AZ 1 (euw1-az1)
│   ├── Public Subnet:  10.0.1.0/24
│   │   ├── ALB
│   │   └── NAT Gateway
│   ├── Private Subnet: 10.0.11.0/24
│   │   ├── Lambda Functions
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
│   │   ├── Lambda Functions
│   │   └── ECS Tasks
│   └── Data Subnet:    10.0.22.0/24
│       ├── RDS Standby
│       └── OpenSearch Node 2
│
└── AZ 3 (euw1-az3)
    ├── Public Subnet:  10.0.3.0/24
    │   └── NAT Gateway
    ├── Private Subnet: 10.0.13.0/24
    │   └── Lambda Functions
    └── Data Subnet:    10.0.23.0/24
        └── OpenSearch Node 3
```

### VPC Configuration

**VPC CIDR**: `10.0.0.0/16`

#### Subnet Strategy

| Layer | CIDR Range | Purpose | Services |
|-------|------------|---------|----------|
| **Public** | 10.0.0.0/20 | Internet-facing resources | ALB, NAT Gateway |
| **Private** | 10.0.16.0/20 | Application layer | Lambda, ECS, Fargate |
| **Data** | 10.0.32.0/20 | Data persistence | RDS, OpenSearch, ElastiCache |
| **Management** | 10.0.48.0/20 | Ops and maintenance | Bastion, SSM endpoints |

#### Security Groups

**ALB Security Group**
- Inbound: 443/TCP from 0.0.0.0/0 (internet)
- Outbound: Ephemeral ports to Lambda SG

**Lambda Security Group**
- Inbound: Ephemeral ports from ALB SG
- Outbound: 443/TCP to VPC endpoints, data layer

**Data Layer Security Group**
- Inbound: Service ports from Lambda SG only
  - RDS: 5432/TCP (PostgreSQL)
  - OpenSearch: 443/TCP
  - ElastiCache: 6379/TCP (Redis)
- Outbound: None (isolated)

#### Network ACLs

Public Subnets:
- Inbound: 80/TCP, 443/TCP from anywhere
- Outbound: Ephemeral ports to internet

Private Subnets:
- Inbound: All traffic from VPC CIDR
- Outbound: 443/TCP to internet via NAT

Data Subnets:
- Inbound: Service ports from private subnets only
- Outbound: None (no internet access)

## Service Distribution

### Cross-AZ Services

**Managed Services (AWS-managed Multi-AZ)**:
- API Gateway (regional endpoint)
- Step Functions (regional)
- DynamoDB (global tables)
- S3 (automatic replication)
- Lambda (automatic placement)
- Bedrock (regional)
- SageMaker (endpoint in multiple AZ)

**Self-managed Multi-AZ**:
- OpenSearch: 3 nodes, 1 per AZ
- RDS PostgreSQL: Primary + Standby
- Application Load Balancer: Nodes in all AZ

### Single-AZ Services

**Stateless** (può rimanere in singola AZ):
- EventBridge (regional, no AZ preference)
- SNS Topics (regional)
- CloudWatch (regional)

## Traffic Routing

### Ingress Path

```
Internet
  ↓
Route 53 (DNS)
  ↓
CloudFront (CDN) [Edge Locations globally]
  ↓
AWS Shield + WAF [DDoS protection]
  ↓
Application Load Balancer [Multi-AZ]
  ↓
API Gateway [Regional Endpoint]
  ↓
Lambda / Step Functions [Auto-distributed]
```

### Egress Path

```
Lambda (Private Subnet)
  ↓
NAT Gateway (Public Subnet)
  ↓
Internet Gateway
  ↓
Internet / AWS Services
```

**Alternative per AWS Services**: VPC Endpoints (PrivateLink)
- S3 Gateway Endpoint
- DynamoDB Gateway Endpoint
- Bedrock Interface Endpoint
- SageMaker Runtime Interface Endpoint
- Secrets Manager Interface Endpoint

## Load Balancing Strategy

### Application Load Balancer

**Configuration**:
- Cross-zone load balancing: Enabled
- Deregistration delay: 30 seconds
- Health check interval: 30 seconds
- Healthy threshold: 2 consecutive checks
- Unhealthy threshold: 3 consecutive checks

**Target Groups**:
- API Gateway (HTTP/HTTPS targets)
- ECS Fargate (for async workers)

**Routing Rules**:
```
/api/v1/*     → API Gateway invoke URL
/health       → Lambda health check
/static/*     → S3 via CloudFront
```

### Lambda Distribution

**Concurrency allocation**:
- **Reserved**: 100 per critical APIs (create ticket, get solution)
- **Provisioned**: 50 for latency-sensitive endpoints
- **Unreserved**: Pool condiviso per altri workload

**Placement**:
- Automatic across all AZ in private subnets
- ENI creation per subnet

### OpenSearch Cluster

**Node Distribution**:
- 3x r6g.large.search (1 per AZ)
- 5 primary shards, 1 replica
- Shard allocation awareness per AZ

**Routing**:
- Client-side load balancing (SDK)
- Health-aware routing
- Preference per local shard

## Deployment Pipeline

### CI/CD Flow

```
GitHub Repository
  ↓
[CodePipeline Trigger]
  ↓
CodeBuild
  ├─ Unit Tests
  ├─ Linting
  ├─ Security Scan (Snyk)
  └─ Build Artifacts
  ↓
[Store in S3]
  ↓
CloudFormation ChangeSet
  ↓
[Manual Approval Gate]
  ↓
Deploy to Dev Environment
  ↓
Integration Tests
  ↓
Deploy to Staging (full stack)
  ↓
Load Testing (Locust)
  ↓
[Manual Approval Gate]
  ↓
Deploy to Production
  ├─ Blue/Green Deployment
  └─ Canary 10% → 50% → 100%
  ↓
Post-deployment Validation
  ↓
CloudWatch Alarms Monitoring
```

### Infrastructure as Code

**Tool**: AWS CloudFormation / CDK (TypeScript)

**Stack Organization**:
```
├── network-stack.yaml          # VPC, subnets, SG, NACL
├── data-stack.yaml             # DynamoDB, RDS, OpenSearch
├── compute-stack.yaml          # Lambda, Step Functions
├── ml-stack.yaml               # SageMaker, Bedrock config
├── api-stack.yaml              # API Gateway, Cognito
├── monitoring-stack.yaml       # CloudWatch, X-Ray
└── pipeline-stack.yaml         # CI/CD resources
```

**Deployment Sequence**:
1. Network (VPC foundation)
2. Data (databases, storage)
3. Compute (Lambda, containers)
4. ML (endpoints, models)
5. API (gateway, auth)
6. Monitoring (observability)

### Deployment Strategies

#### Blue/Green Deployment

Per servizi stateful (RDS, OpenSearch):
1. Provision nuova versione (Green)
2. Replica dati da Blue a Green
3. Test su Green environment
4. Switch DNS/routing da Blue a Green
5. Monitor per 24h
6. Decommission Blue

#### Canary Deployment

Per Lambda e SageMaker endpoints:
1. Deploy nuova versione con alias `canary`
2. Route 10% traffico su canary
3. Monitor metriche per 30min
4. Se OK → 50% traffico
5. Monitor per 1h
6. Se OK → 100% traffico
7. Promuovi canary a production

**Rollback automatico se**:
- Error rate > 1%
- Latency p99 > 5s
- Custom metric (accuracy) degrada

#### Rolling Deployment

Per ECS tasks:
1. Deploy 1 task alla volta
2. Health check nuova task
3. Drain connections da vecchia task
4. Terminate vecchia task
5. Repeat per tutte le task

## Disaster Recovery

### RTO/RPO Targets

| Servizio | RTO | RPO | Strategia |
|----------|-----|-----|-----------|
| **API Gateway** | 0 (managed) | 0 | Multi-AZ automatico |
| **Lambda** | 0 (managed) | 0 | Multi-AZ automatico |
| **DynamoDB** | 0 (global tables) | < 1s | Active-Active |
| **S3** | 0 (managed) | 0 | Cross-region replication |
| **RDS** | 5 min | 5 min | Multi-AZ failover |
| **OpenSearch** | 30 min | 1h | Snapshot + restore |
| **SageMaker** | 1h | N/A | Re-deploy da registry |

### Backup Strategy

**Automated Backups**:
- **DynamoDB**: Point-in-time recovery (35 days)
- **RDS**: Automated daily + transaction logs (7 days)
- **OpenSearch**: Automated snapshots daily (14 days)
- **S3**: Versioning enabled + lifecycle policies

**Manual Backups**:
- Pre-deployment snapshot di RDS e OpenSearch
- Export DynamoDB per audit mensili
- Model artifacts backup su S3 Glacier

### Cross-Region Replication

**Active-Passive Standby**:
- **Primary**: eu-south-1 (Milano)
- **Standby**: eu-west-1 (Irlanda)

**Replicated Resources**:
- S3 buckets (CRR enabled)
- DynamoDB global tables
- Lambda code (cross-region deployment)
- CloudFormation templates

**Failover Process**:
1. Detect primary region failure (Route 53 health check)
2. Route 53 failover to Irlanda endpoint
3. Promote RDS standby in Irlanda (se necessario)
4. Scale up Lambda concurrency in Irlanda
5. Update API clients (automatic via DNS)

**Failback Process**:
1. Restore primary region
2. Sync data da standby a primary
3. Test primary region
4. Route 53 switch back (gradual canary)

## Scaling Strategy

### Auto-scaling Configuration

**Lambda**:
- Target: 70% concurrent execution utilization
- Scale out: +10 concurrent per 1 min sustained load
- Scale in: -5 concurrent after 5 min low load

**DynamoDB**:
- On-demand mode (no pre-provisioning)
- Burst capacity per spike handling

**OpenSearch**:
- Manual scaling (CloudWatch alarm triggers SNS)
- Target: 60% CPU, 70% JVM heap
- Scale out: Add 1 node (takes ~20 min)

**SageMaker Endpoints**:
- Target tracking: 70% invocations per instance
- Min instances: 2
- Max instances: 10
- Scale out cooldown: 300s
- Scale in cooldown: 600s

### Cost Optimization

**Reserved Capacity**:
- RDS: 1 year reserved instance (40% saving)
- OpenSearch: 1 year reserved nodes (30% saving)

**Spot Instances**:
- SageMaker training: 70% spot, 30% on-demand
- Batch processing: 100% spot con fallback

**Lifecycle Policies**:
- S3 raw data: IA after 30 days, Glacier after 90 days
- CloudWatch logs: Retention 30 days per default
- DynamoDB TTL: Session data, idempotency tokens

## Monitoring Deployment

### Health Checks

**Endpoint Health**:
- `/health` → Lambda basic check (200 OK)
- `/health/deep` → DynamoDB + OpenSearch connectivity

**Service Health**:
- API Gateway: 5xx error rate < 0.1%
- Lambda: Error invocation rate < 1%
- Step Functions: Failed execution rate < 2%

**Synthetic Monitoring**:
- CloudWatch Synthetics canaries
- Test end-to-end ogni 5 minuti
- Alert se 2 consecutive failures

## Riferimenti

- [Overview Architettura](./overview.md)
- [Security Architecture](./security.md)
- [Monitoring](../09-operations/monitoring.md)
- [CI/CD Pipeline](../09-operations/cicd.md)
