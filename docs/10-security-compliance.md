# Security & Compliance

## Overview

Il sistema implementa misure di sicurezza multi-livello e garantisce compliance con normative GDPR e standard di settore.

## Security Principles

### Defense in Depth

```
┌─────────────────────────────────────────────┐
│ Layer 7: Application Security              │
│ - Input validation                          │
│ - Output sanitization (Guardrails)          │
│ - PII redaction                             │
├─────────────────────────────────────────────┤
│ Layer 6: Identity & Access                 │
│ - Cognito MFA                               │
│ - IAM least privilege                       │
│ - Token expiration                          │
├─────────────────────────────────────────────┤
│ Layer 5: Data Encryption                   │
│ - KMS encryption at rest                    │
│ - TLS 1.2+ in transit                       │
│ - Secrets Manager rotation                  │
├─────────────────────────────────────────────┤
│ Layer 4: Network Security                  │
│ - Security Groups (stateful)                │
│ - NACLs (stateless)                         │
│ - VPC isolation                             │
├─────────────────────────────────────────────┤
│ Layer 3: Edge Protection                   │
│ - AWS WAF (OWASP Top 10)                    │
│ - AWS Shield (DDoS)                         │
│ - CloudFront geo-blocking                   │
├─────────────────────────────────────────────┤
│ Layer 2: Monitoring & Detection            │
│ - GuardDuty threats                         │
│ - CloudTrail audit                          │
│ - Config compliance                         │
├─────────────────────────────────────────────┤
│ Layer 1: Governance                        │
│ - AWS Organizations SCPs                    │
│ - Automated remediation                     │
│ - Regular audits                            │
└─────────────────────────────────────────────┘
```

## Encryption

### At Rest
- **S3**: SSE-KMS con customer-managed keys
- **DynamoDB**: AWS-owned encryption keys (default)
- **RDS**: KMS encryption per storage e backup
- **OpenSearch**: Node-to-node encryption enabled
- **EBS**: Automatic encryption per Lambda
- **Secrets Manager**: KMS encryption

### In Transit
- **TLS 1.2+** obbligatorio su tutte le connessioni
- **Certificate pinning** su client critici
- **Perfect Forward Secrecy** (PFS) enabled
- **HSTS headers** su tutte le API responses

### Key Management

```
KMS Keys:
  - app-data-key: S3, RDS, Secrets (rotation: annual)
  - logging-key: CloudWatch Logs (rotation: annual)
  - backup-key: Snapshots (no rotation, for restore)

Key Policies:
  - Least privilege per service
  - CloudTrail logging enabled
  - Key deletion prevention (7-day wait)
```

## Authentication & Authorization

### Cognito Configuration

```yaml
UserPool:
  Name: ai-support-users
  MFA: OPTIONAL (TOTP)
  PasswordPolicy:
    MinLength: 12
    RequireUppercase: true
    RequireLowercase: true
    RequireNumbers: true
    RequireSymbols: true
    TemporaryPasswordValidity: 1 day
  AccountRecovery: EMAIL_ONLY
  EmailVerification: REQUIRED
  DeviceTracking: ALWAYS

OAuthFlows:
  - AuthorizationCodeGrant (Web UI)
  - ClientCredentials (Service-to-Service)

Tokens:
  AccessToken: 1 hour
  IdToken: 1 hour
  RefreshToken: 30 days
```

### RBAC Model

```yaml
Groups:
  admin:
    Permissions:
      - Full API access
      - Deploy models
      - Manage users
      - Write KB
    IAM Role: AdminRole

  operator:
    Permissions:
      - Create/Read/Update tickets
      - Read KB
      - Submit feedback
    IAM Role: OperatorRole

  readonly:
    Permissions:
      - Read tickets
      - Read KB
      - View dashboards
    IAM Role: ReadOnlyRole

  api-client:
    Permissions:
      - Create tickets (API only)
      - Read own tickets
    IAM Role: APIClientRole
```

## Input Validation

### API Gateway Level

```json
{
  "type": "object",
  "required": ["customer", "asset", "symptom_text"],
  "properties": {
    "customer": {
      "type": "object",
      "required": ["id", "name"],
      "properties": {
        "id": {
          "type": "string",
          "minLength": 1,
          "maxLength": 100
        }
      }
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
  },
  "additionalProperties": false
}
```

### Application Level

```python
ALLOWED_CHARACTERS = re.compile(r'^[a-zA-Z0-9\s.,;:!?()\-_/\'\"àèéìòùÀÈÉÌÒÙ]+$')
MAX_PAYLOAD_SIZE = 10 * 1024 * 1024  # 10MB

def validate_input(data):
    # 1. Type validation
    if not isinstance(data, dict):
        raise ValidationError('Invalid payload type')

    # 2. Size validation
    if sys.getsizeof(json.dumps(data)) > MAX_PAYLOAD_SIZE:
        raise ValidationError('Payload too large')

    # 3. Character whitelist
    if 'symptom_text' in data:
        if not ALLOWED_CHARACTERS.match(data['symptom_text']):
            raise ValidationError('Invalid characters in symptom_text')

    # 4. SQL injection prevention (paranoid)
    dangerous_patterns = ['--', ';--', '/*', '*/', 'xp_', 'sp_']
    for pattern in dangerous_patterns:
        if pattern in str(data).lower():
            raise ValidationError('Potentially dangerous input detected')

    return True
```

## Output Security (Guardrails)

### Bedrock Guardrails

```yaml
GuardrailName: TechSupportGuardrail
Version: 1.0

ContentFilters:
  - Type: SEXUAL
    Strength: HIGH
  - Type: VIOLENCE
    Strength: HIGH
  - Type: HATE
    Strength: HIGH
  - Type: INSULTS
    Strength: MEDIUM

DeniedTopics:
  - Name: Medical Advice
    Definition: "Unverified medical procedures or diagnoses"
    Examples:
      - "This could be a heart condition"
      - "You should take this medication"

  - Name: Legal Advice
    Definition: "Legal interpretations or recommendations"

  - Name: Competitor Mentions
    Definition: "References to competitor products"

PIIFilters:
  - EMAIL: REDACT
  - PHONE: REDACT
  - CREDIT_CARD: BLOCK
  - SSN: BLOCK
  - DRIVER_LICENSE: REDACT
  - PASSPORT: REDACT

ContextualGrounding:
  Enabled: true
  Threshold: 0.75  # 75% grounded in provided context
  Action: BLOCK
```

### Custom Post-Processing

```python
def sanitize_output(llm_response, context_chunks):
    # 1. PII detection (backup layer)
    pii_patterns = {
        'email': r'\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b',
        'phone': r'\b\d{3}[-.]?\d{3}[-.]?\d{4}\b',
        'credit_card': r'\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b'
    }

    for pii_type, pattern in pii_patterns.items():
        if re.search(pattern, llm_response):
            logger.warning(f'PII detected: {pii_type}')
            llm_response = re.sub(pattern, '[REDACTED]', llm_response)

    # 2. Citation verification
    citations = extract_citations(llm_response)
    if len(citations) == 0:
        raise UncitedResponseError('No citations found')

    # Verify citations are from provided context
    valid_sources = {chunk['source_uri'] for chunk in context_chunks}
    for citation in citations:
        if citation['source_uri'] not in valid_sources:
            raise InvalidCitationError(f'Invalid source: {citation["source_uri"]}')

    # 3. Toxicity check
    toxicity_score = check_toxicity(llm_response)
    if toxicity_score > 0.5:
        raise ToxicContentError(f'Toxicity score: {toxicity_score}')

    # 4. Groundedness verification
    groundedness = calculate_groundedness(llm_response, context_chunks)
    if groundedness < 0.75:
        logger.warning(f'Low groundedness: {groundedness}')
        # Flag for review but don't block
        flag_for_review(llm_response, groundedness)

    return llm_response
```

## GDPR Compliance

### Data Subject Rights

#### Right to Access
```python
@app.route('/api/v1/customers/<customer_id>/export', methods=['GET'])
@require_auth
def export_customer_data(customer_id):
    # Verify requester owns the data
    verify_customer_ownership(customer_id, request.user)

    # Collect all data
    data = {
        'tickets': get_customer_tickets(customer_id),
        'feedback': get_customer_feedback(customer_id),
        'kb_contributions': get_customer_kb_contributions(customer_id),
        'usage_metrics': get_customer_metrics(customer_id)
    }

    # Format as JSON
    export_file = json.dumps(data, indent=2, ensure_ascii=False)

    # Log export event
    log_gdpr_event('data_export', customer_id, request.user)

    return export_file, 200, {'Content-Type': 'application/json'}
```

#### Right to Erasure
```python
def delete_customer_data(customer_id, reason='user_request'):
    # 1. DynamoDB
    delete_dynamodb_records('tickets', customer_id)
    delete_dynamodb_records('feedback', customer_id)

    # 2. S3
    delete_s3_prefix(f'raw-data/customers/{customer_id}/')
    delete_s3_prefix(f'processed-data/customers/{customer_id}/')

    # 3. OpenSearch
    delete_opensearch_documents(customer_id)

    # 4. RDS (if applicable)
    execute_sql(f"DELETE FROM customers WHERE id = '{customer_id}'")

    # 5. Audit log
    log_gdpr_event('erasure', customer_id, reason)

    # 6. Notify dependent systems
    publish_event('customer.deleted', customer_id)
```

#### Right to Rectification
```python
@app.route('/api/v1/customers/<customer_id>', methods=['PATCH'])
@require_auth
def update_customer_data(customer_id):
    verify_customer_ownership(customer_id, request.user)

    updates = request.json

    # Update DynamoDB
    dynamodb.update_item(
        TableName='customers',
        Key={'customer_id': customer_id},
        UpdateExpression='SET ' + ', '.join(f'#{k} = :{k}' for k in updates.keys()),
        ExpressionAttributeNames={f'#{k}': k for k in updates.keys()},
        ExpressionAttributeValues={f':{k}': v for k, v in updates.items()}
    )

    # Log change
    log_gdpr_event('rectification', customer_id, updates.keys())

    return {'status': 'updated'}, 200
```

### Data Retention

```yaml
Policies:
  Tickets:
    Retention: 7 years (compliance requirement)
    Location: DynamoDB + S3 cold storage after 1 year

  Logs:
    Application: 30 days (CloudWatch)
    Audit: 10 years (S3 Glacier)
    Access: 90 days (CloudWatch)

  Training Data:
    Retention: Indefinite (with consent)
    Anonymization: After 90 days

  Temporary Data:
    Session tokens: 1 hour (DynamoDB TTL)
    Idempotency keys: 24 hours (DynamoDB TTL)
    Cache: 1 hour (ElastiCache)
```

### Consent Management

```typescript
interface Consent {
  customer_id: string;
  purpose: 'training' | 'analytics' | 'marketing';
  granted: boolean;
  granted_at?: string;
  withdrawn_at?: string;
  version: string;  // Terms version
  ip_address: string;
  user_agent: string;
}

// Verify consent before processing
function checkConsent(customer_id: string, purpose: string): boolean {
  const consent = getConsent(customer_id, purpose);
  return consent && consent.granted && !consent.withdrawn_at;
}
```

## Audit Logging

### CloudTrail Configuration

```yaml
Trail:
  Name: ai-support-audit
  S3Bucket: audit-logs-bucket
  IncludeGlobalEvents: true
  IsMultiRegion: true
  LogFileValidation: true
  EventSelectors:
    - ReadWriteType: All
      IncludeManagementEvents: true
      DataResources:
        - Type: AWS::S3::Object
          Values: ['arn:aws:s3:::prod-*/']
        - Type: AWS::DynamoDB::Table
          Values: ['arn:aws:dynamodb:*:*:table/prod-*']

InsightSelectors:
  - InsightType: ApiCallRateInsight
```

### Application Audit Events

```python
def log_security_event(event_type, details):
    event = {
        'timestamp': datetime.utcnow().isoformat(),
        'event_type': event_type,
        'severity': get_severity(event_type),
        'user': get_current_user(),
        'ip_address': get_client_ip(),
        'details': details,
        'request_id': get_request_id()
    }

    # CloudWatch Logs (structured)
    logger.info('SECURITY_EVENT', extra=event)

    # Send to SIEM if high severity
    if event['severity'] in ['HIGH', 'CRITICAL']:
        send_to_siem(event)

    # Trigger alert if needed
    if event['event_type'] in ALERT_EVENTS:
        sns.publish(
            TopicArn=SECURITY_ALERTS_TOPIC,
            Message=json.dumps(event)
        )
```

### Logged Events

```yaml
Authentication:
  - Login success/failure
  - MFA challenge
  - Token refresh
  - Password reset
  - Logout

Authorization:
  - Access denied (403)
  - Privilege escalation attempt
  - Role assumption

Data Access:
  - PII data access
  - Bulk data export
  - Customer data deletion
  - KB document upload

System:
  - Configuration changes
  - IAM policy modifications
  - Security group changes
  - KMS key usage
  - Secrets access
```

## Incident Response

### Automated Response

```python
@eventbridge_handler('guardduty.finding')
def handle_security_finding(event):
    finding = event['detail']
    severity = finding['severity']

    if severity >= 7:  # High/Critical
        # 1. Isolate compromised resource
        resource_id = finding['resource']['instanceId']
        isolate_instance(resource_id)

        # 2. Snapshot for forensics
        create_forensic_snapshot(resource_id)

        # 3. Rotate credentials
        rotate_all_secrets()

        # 4. Alert security team
        page_security_team(finding)

        # 5. Create incident ticket
        create_incident_ticket(finding)

    elif severity >= 4:  # Medium
        # Log and alert
        log_security_event('guardduty_finding', finding)
        send_slack_alert(finding)
```

### Runbook

1. **Detection**: GuardDuty / CloudTrail alert
2. **Triage**: Severity assessment (< 5min)
3. **Containment**: Isolate affected resources (< 15min)
4. **Investigation**: Analyze logs, snapshots (< 2h)
5. **Eradication**: Remove threat, patch vulnerability (< 4h)
6. **Recovery**: Restore from clean state (< 8h)
7. **Post-Incident**: Report, lessons learned (< 48h)

## Compliance Frameworks

### GDPR
- ✅ Data encryption at rest and in transit
- ✅ Right to access, rectification, erasure
- ✅ Data portability
- ✅ Consent management
- ✅ Breach notification (< 72h)
- ✅ DPO designated
- ✅ Privacy by design

### ISO 27001 (in progress)
- ✅ Access control (A.9)
- ✅ Cryptography (A.10)
- ✅ Operations security (A.12)
- ✅ Incident management (A.16)
- 🔄 ISMS documentation
- 🔄 Risk assessment
- 🔄 External audit

### SOC 2 Type II (planned)
- Security
- Availability
- Confidentiality
- Processing Integrity
- Privacy

## Security Testing

### Automated Scanning

```yaml
Tools:
  - Snyk: Dependency vulnerability scanning (CI/CD)
  - TruffleHog: Secrets detection (pre-commit)
  - OWASP ZAP: API security testing (nightly)
  - AWS Inspector: Instance vulnerability scan (weekly)

Schedule:
  - Pre-commit: Secrets scan
  - PR: Dependency check
  - Daily: SAST (static analysis)
  - Weekly: DAST (dynamic analysis)
  - Monthly: Penetration test
  - Quarterly: External audit
```

### Penetration Testing

Scope:
- API endpoints (auth, input validation)
- AWS infrastructure (IAM, SG misconfig)
- Application logic (injection, XSS)
- Data exposure (S3 buckets, logs)

Frequency: Quarterly + after major changes

## Riferimenti

- [Architecture Security](./02-architecture/security.md)
- [IAM Roles](./03-aws-services/README.md#iam-roles)
- [Monitoring](./09-operations/monitoring.md)
