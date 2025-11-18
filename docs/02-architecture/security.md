# Security Architecture

## Security Layers

Il sistema implementa un modello di sicurezza a più livelli (defense in depth):

```
┌─────────────────────────────────────────────┐
│ Layer 7: Application Security              │ ← Guardrails, Input Validation
├─────────────────────────────────────────────┤
│ Layer 6: Identity & Access Management      │ ← Cognito, IAM Roles
├─────────────────────────────────────────────┤
│ Layer 5: Data Encryption                   │ ← KMS, TLS
├─────────────────────────────────────────────┤
│ Layer 4: Network Security                  │ ← Security Groups, NACLs
├─────────────────────────────────────────────┤
│ Layer 3: Edge Protection                   │ ← WAF, Shield
├─────────────────────────────────────────────┤
│ Layer 2: Monitoring & Detection            │ ← GuardDuty, CloudTrail
├─────────────────────────────────────────────┤
│ Layer 1: Compliance & Governance           │ ← Config, Audit
└─────────────────────────────────────────────┘
```

## Network Security

### Network Segmentation

| Layer | CIDR | Services | Internet Access | Purpose |
|-------|------|----------|-----------------|---------|
| **Public** | 10.0.0.0/20 | ALB, NAT Gateway | Yes (bidirectional) | Internet-facing |
| **Private** | 10.0.16.0/20 | Lambda, ECS | Outbound only (via NAT) | Application |
| **Data** | 10.0.32.0/20 | RDS, OpenSearch, Cache | No | Data persistence |
| **Management** | 10.0.48.0/20 | Bastion, Systems Manager | No | Operations |

### Security Groups

#### API Layer Security Group
```
Name: api-lambda-sg
Inbound:
  - 443/TCP from alb-sg
  - 443/TCP from vpc-endpoints-sg
Outbound:
  - 443/TCP to 0.0.0.0/0 (HTTPS to AWS services)
  - 5432/TCP to data-layer-sg (RDS)
  - 443/TCP to data-layer-sg (OpenSearch)
  - 6379/TCP to data-layer-sg (ElastiCache)
```

#### Data Layer Security Group
```
Name: data-layer-sg
Inbound:
  - 5432/TCP from api-lambda-sg (PostgreSQL)
  - 443/TCP from api-lambda-sg (OpenSearch)
  - 6379/TCP from api-lambda-sg (Redis)
Outbound:
  - None (fully isolated)
```

#### ALB Security Group
```
Name: alb-sg
Inbound:
  - 443/TCP from 0.0.0.0/0 (Internet HTTPS)
  - 80/TCP from 0.0.0.0/0 (HTTP redirect to HTTPS)
Outbound:
  - 443/TCP to api-lambda-sg
```

### Network ACLs

**Public Subnet NACL**:
```
Inbound:
  100: Allow 443/TCP from 0.0.0.0/0
  110: Allow 80/TCP from 0.0.0.0/0
  120: Allow 1024-65535/TCP from 0.0.0.0/0 (ephemeral)
  *: Deny all
Outbound:
  100: Allow all traffic to 0.0.0.0/0
  *: Deny all
```

**Private Subnet NACL**:
```
Inbound:
  100: Allow all from 10.0.0.0/16 (VPC CIDR)
  *: Deny all
Outbound:
  100: Allow 443/TCP to 0.0.0.0/0 (HTTPS)
  110: Allow all to 10.0.0.0/16 (VPC CIDR)
  *: Deny all
```

**Data Subnet NACL**:
```
Inbound:
  100: Allow 5432/TCP from 10.0.16.0/20 (private subnets)
  110: Allow 443/TCP from 10.0.16.0/20
  120: Allow 6379/TCP from 10.0.16.0/20
  *: Deny all
Outbound:
  100: Allow all to 10.0.16.0/20 (responses)
  *: Deny all
```

### VPC Endpoints (PrivateLink)

Per evitare traffico internet e ridurre costi:

```
Gateway Endpoints (no cost):
  - S3
  - DynamoDB

Interface Endpoints ($0.01/hour + data):
  - Bedrock Runtime
  - SageMaker Runtime
  - Secrets Manager
  - Systems Manager (SSM)
  - CloudWatch Logs
  - ECR (Docker images)
```

## Identity & Access Management

### IAM Roles Architecture

#### Lambda Execution Role

```yaml
RoleName: LambdaTicketProcessorRole
AssumeRolePolicy:
  Service: lambda.amazonaws.com
Policies:
  - AWSLambdaVPCAccessExecutionRole (managed)
  - Custom Inline Policy:
      DynamoDB:
        - GetItem, PutItem, UpdateItem, Query
        - Tables: tickets, idempotency, feedback
      S3:
        - GetObject, PutObject
        - Buckets: raw-data-*, processed-data-*
      Bedrock:
        - InvokeModel
        - Models: claude-3-*, llama-2-*
      OpenSearch:
        - ESHttpGet, ESHttpPost, ESHttpPut
      Secrets:
        - GetSecretValue
        - Secrets: prod/opensearch-creds
      KMS:
        - Decrypt
        - Keys: alias/app-data-key
```

#### Step Functions Role

```yaml
RoleName: StepFunctionsOrchestrationRole
AssumeRolePolicy:
  Service: states.amazonaws.com
Policies:
  Lambda:
    - InvokeFunction on all app lambdas
  SageMaker:
    - CreateTrainingJob, CreateEndpoint
    - DescribeTrainingJob, DescribeEndpoint
  Bedrock:
    - InvokeModel
  DynamoDB:
    - UpdateItem on tickets table
  SNS:
    - Publish to notification topics
  Events:
    - PutEvents on default event bus
```

#### SageMaker Role

```yaml
RoleName: SageMakerMLOpsRole
AssumeRolePolicy:
  Service: sagemaker.amazonaws.com
Policies:
  S3:
    - Full access on ml-data-* buckets
    - GetObject on raw-data-* (read-only)
  ECR:
    - GetAuthorizationToken, BatchGetImage
  CloudWatch:
    - PutMetricData, CreateLogStream, PutLogEvents
  DynamoDB:
    - GetItem, Scan on training-labels table
```

#### API Gateway Execution Role

```yaml
RoleName: APIGatewayInvokeRole
AssumeRolePolicy:
  Service: apigateway.amazonaws.com
Policies:
  Lambda:
    - InvokeFunction on API handlers
  StepFunctions:
    - StartExecution on ticket-processing workflow
  SQS:
    - SendMessage on ticket-queue
  DynamoDB:
    - GetItem, PutItem (for direct integrations)
```

### Least Privilege Principle

**Best Practices applicate**:
- Ogni servizio ha il proprio role dedicato
- Permessi granulari su risorse specifiche (no wildcards)
- Condition keys per ulteriori restrizioni:
  ```json
  "Condition": {
    "StringEquals": {
      "s3:x-amz-server-side-encryption": "aws:kms"
    },
    "IpAddress": {
      "aws:SourceIp": "10.0.0.0/16"
    }
  }
  ```

### Service Control Policies (SCP)

A livello di AWS Organization:

```yaml
DenyUnapprovedRegions:
  Effect: Deny
  Action: "*"
  Resource: "*"
  Condition:
    StringNotEquals:
      aws:RequestedRegion:
        - eu-south-1  # Milano
        - eu-west-1   # Dublin (DR)

EnforceEncryption:
  Effect: Deny
  Action:
    - s3:PutObject
  Resource: "*"
  Condition:
    StringNotEquals:
      s3:x-amz-server-side-encryption: aws:kms

RequireMFA:
  Effect: Deny
  Action: "*"
  Resource: "*"
  Condition:
    BoolIfExists:
      aws:MultiFactorAuthPresent: false
    StringNotEquals:
      aws:PrincipalType: AssumedRole  # Exclude service roles
```

## Data Encryption

### Encryption at Rest

| Service | Encryption Method | Key Management |
|---------|-------------------|----------------|
| **S3** | SSE-KMS | Customer managed key |
| **DynamoDB** | AWS owned key | AWS managed |
| **RDS** | AES-256 | Customer managed KMS key |
| **OpenSearch** | Node-to-node encryption | AWS managed |
| **EBS** | AES-256 (Lambda) | AWS managed |
| **Secrets Manager** | KMS | Customer managed key |

### Encryption in Transit

**TLS/SSL Everywhere**:
- API Gateway: TLS 1.2+ only
- ALB: TLS 1.2+ with strong ciphers
- RDS: Force SSL connections
- OpenSearch: HTTPS only
- Lambda → services: boto3 with SSL verification

**Certificate Management**:
- ACM (AWS Certificate Manager) per ALB e CloudFront
- Auto-renewal 60 giorni prima della scadenza
- SNS notification se renewal fallisce

### KMS Key Architecture

```
Customer Master Keys (CMK):

1. app-data-key
   - Usage: S3, RDS, Secrets Manager
   - Rotation: Automatic annual
   - Policy: Lambda role can Decrypt only

2. logging-key
   - Usage: CloudWatch Logs encryption
   - Rotation: Automatic annual
   - Policy: CloudWatch service role

3. backup-key
   - Usage: Snapshot encryption
   - Rotation: Disabled (for restore compatibility)
```

**Key Policies**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Allow Lambda to decrypt",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT:role/LambdaTicketProcessorRole"
      },
      "Action": ["kms:Decrypt", "kms:DescribeKey"],
      "Resource": "*"
    }
  ]
}
```

## Application Security

### Authentication & Authorization

**Cognito User Pools**:
```
Pool Settings:
  - MFA: Optional (TOTP via app)
  - Password Policy:
      Min length: 12 characters
      Require: uppercase, lowercase, numbers, symbols
  - Account recovery: Email only
  - User verification: Email verification required

OAuth2 Flows:
  - Authorization Code Grant (Web UI)
  - Client Credentials (Service-to-Service)

Token Configuration:
  - Access token expiry: 1 hour
  - Refresh token expiry: 30 days
  - ID token expiry: 1 hour
```

**API Gateway Authorizers**:
```yaml
Type: Cognito User Pool Authorizer
IdentitySource: $request.header.Authorization
TokenValidation:
  - Signature verification
  - Expiry check
  - Issuer validation (Cognito User Pool)
  - Audience validation (App Client ID)
```

**RBAC (Role-Based Access Control)**:
```
Cognito Groups → IAM Roles mapping:

admin-group:
  - Full API access
  - Write KB, deploy models

operator-group:
  - Read/Write tickets
  - Read KB
  - Submit feedback

readonly-group:
  - Read tickets
  - Read KB
```

### Input Validation

**API Gateway Level**:
- JSON Schema validation su tutti i payload
- Max payload size: 10MB
- Request/response mapping templates

**Lambda Level**:
```python
def validate_ticket_input(event):
    schema = {
        "type": "object",
        "required": ["customer", "asset", "symptom_text"],
        "properties": {
            "customer": {
                "type": "object",
                "required": ["id", "name"]
            },
            "symptom_text": {
                "type": "string",
                "minLength": 10,
                "maxLength": 5000
            },
            "error_code": {
                "type": "string",
                "pattern": "^[A-Z][0-9]{3,4}$"
            }
        }
    }
    validate(instance=event, schema=schema)
```

### Output Security (Guardrails)

**Bedrock Guardrails Configuration**:
```yaml
GuardrailName: TechSupportGuardrail
DeniedTopics:
  - Harmful content (violence, hate speech)
  - PII exposure (credit cards, SSN, emails)
  - Unverified medical advice
  - Competitor mentions

ContentFilters:
  - Level: HIGH for sexual content
  - Level: HIGH for violence
  - Level: MEDIUM for hate speech
  - Level: HIGH for insults

PIIRedaction:
  - EMAIL: REDACT
  - PHONE: REDACT
  - CREDIT_CARD: BLOCK request
  - SSN: BLOCK request

ContextualGrounding:
  - Threshold: 0.75 (must be 75%+ grounded in context)
  - Action: BLOCK if below threshold
```

**Custom Post-processing**:
```python
def sanitize_response(llm_output, sources):
    # 1. Verify all claims are cited
    citations = extract_citations(llm_output)
    if len(citations) == 0:
        raise UncitedResponseError()

    # 2. PII detection (backup layer)
    pii_detected = detect_pii(llm_output)
    if pii_detected:
        llm_output = redact_pii(llm_output, pii_detected)

    # 3. Harmful content check
    toxicity_score = check_toxicity(llm_output)
    if toxicity_score > 0.5:
        raise ToxicContentError()

    return llm_output
```

## Monitoring & Detection

### GuardDuty

Threat detection su:
- Anomalous API calls
- Compromised EC2/Lambda instances
- Reconnaissance activity
- Cryptocurrency mining

**Automated Response**:
```yaml
GuardDuty Finding → EventBridge → Lambda → Actions:
  - High severity: Isolate security group, SNS alert
  - Medium severity: CloudWatch alarm, log to SIEM
  - Low severity: Log only
```

### CloudTrail

**Configuration**:
- Enabled su tutte le regioni
- Log file validation: Enabled
- S3 bucket encryption: KMS
- CloudWatch Logs integration per real-time alerts
- Object-level logging per S3 buckets critici

**Allarmi Configurati**:
```
- Root account usage
- IAM policy changes
- Security group changes
- Failed console login attempts > 5
- KMS key deletion
- S3 bucket policy changes
```

### VPC Flow Logs

Logging di tutto il traffico di rete:
- Livello: Subnet (per data layer)
- Destination: CloudWatch Logs
- Retention: 30 giorni
- Filter: Rejected traffic only (per ottimizzare costi)

### AWS Config

Compliance monitoring:
```
Rules attive:
  - encrypted-volumes (tutti gli EBS)
  - rds-encryption-enabled
  - s3-bucket-public-read-prohibited
  - iam-password-policy
  - cloudtrail-enabled
  - vpc-flow-logs-enabled
  - dynamodb-pitr-enabled

Remediation automatica:
  - s3-bucket-public-read → Remove public ACL
  - security-group-unrestricted-ssh → Revoke rule
```

## Secrets Management

### AWS Secrets Manager

**Struttura Secrets**:
```
prod/rds/credentials
  - username: app_user
  - password: <auto-generated>
  - rotation: Every 30 days

prod/opensearch/credentials
  - username: app_user
  - password: <auto-generated>
  - rotation: Every 30 days

prod/external-api/tokens
  - itsm_api_key: <manual>
  - rotation: Manual (90 days reminder)
```

**Rotation Lambda**:
```python
def rotate_secret(event):
    secret_arn = event['SecretId']
    token = event['ClientRequestToken']
    step = event['Step']

    if step == "createSecret":
        # Generate new password
        new_password = generate_strong_password()
        # Store in AWSPENDING
        store_pending_secret(secret_arn, new_password)

    elif step == "setSecret":
        # Update RDS/OpenSearch with new password
        update_database_password(new_password)

    elif step == "testSecret":
        # Verify new credentials work
        test_connection(new_password)

    elif step == "finishSecret":
        # Promote AWSPENDING to AWSCURRENT
        finalize_secret(secret_arn)
```

### Parameter Store

Per configurazioni non sensibili:
```
/app/config/max_tokens
/app/config/temperature
/app/config/top_k_results
/app/feature_flags/enable_streaming
```

## Compliance & Audit

### GDPR Compliance

**Right to Erasure**:
```python
def delete_user_data(customer_id):
    # 1. Delete from DynamoDB
    delete_dynamodb_records(customer_id)

    # 2. Delete from S3
    delete_s3_objects(customer_id)

    # 3. Delete from OpenSearch
    delete_opensearch_documents(customer_id)

    # 4. Log deletion event
    log_gdpr_event("erasure", customer_id)
```

**Data Portability**:
- API `/customers/{id}/export` ritorna tutti i dati in JSON
- Formato machine-readable
- Include: tickets, feedback, KB contributions

**Consent Management**:
- DynamoDB table `customer_consents`
- Track: purpose, timestamp, version
- Validate prima di ogni processing

### Audit Logging

**Application Logs**:
```json
{
  "timestamp": "2025-11-18T10:30:00Z",
  "level": "INFO",
  "service": "ticket-processor",
  "action": "generate_solution",
  "user_id": "usr_12345",
  "ticket_id": "tkt_67890",
  "model": "claude-3-sonnet",
  "tokens": 2500,
  "latency_ms": 2140,
  "grounding_score": 0.89
}
```

**Security Events**:
```json
{
  "timestamp": "2025-11-18T10:31:00Z",
  "level": "WARN",
  "event_type": "authentication_failure",
  "source_ip": "203.0.113.42",
  "user": "john.doe@example.com",
  "reason": "invalid_password",
  "attempts": 3
}
```

### Penetration Testing

**Allowed Testing**:
- AWS pre-approva testing su: EC2, RDS, Aurora, CloudFront, API Gateway, Lambda, Lightsail, Elastic Beanstalk
- No approval needed se rispetti AWS Customer Support Policy

**Scope**:
- API endpoints (rate limiting, auth bypass)
- Input validation (injection attacks)
- Authorization (privilege escalation)
- Output encoding (XSS, template injection)

**Forbidden**:
- DDoS attacks
- DNS zone walking
- Port flooding

## Incident Response

### Runbook

**Security Incident Detected**:
1. **Identify**: GuardDuty / CloudTrail alert
2. **Contain**: Isolate compromised resources (SG modification)
3. **Investigate**: Analyze CloudTrail logs, VPC Flow Logs
4. **Eradicate**: Terminate compromised instances, rotate credentials
5. **Recover**: Restore from clean snapshots
6. **Lessons Learned**: Update runbook, improve detection

**Automated Containment**:
```python
def isolate_instance(instance_id):
    # Create quarantine security group (no inbound/outbound)
    quarantine_sg = create_quarantine_sg()

    # Modify instance SG
    ec2.modify_instance_attribute(
        InstanceId=instance_id,
        Groups=[quarantine_sg]
    )

    # Snapshot for forensics
    create_snapshot(instance_id)

    # Notify security team
    sns.publish(Topic="security-incidents", Message=f"Instance {instance_id} quarantined")
```

## Riferimenti

- [Architettura Overview](./overview.md)
- [Deployment Topology](./deployment.md)
- [IAM Policies](../07-integration/iam-policies.md)
- [Compliance](../10-security-compliance.md)
