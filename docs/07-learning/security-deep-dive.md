# Security Deep Dive - IAM, Encryption, Secrets

## Overview

Questa guida fornisce una comprensione approfondita delle best practices di sicurezza AWS per il progetto AI Technical Support System, con focus su Identity & Access Management, crittografia dei dati, gestione dei segreti e protezione della rete.

### Perché è importante nel nostro contesto

Il sistema gestisce:
- **Dati sensibili dei clienti**: Ticket di supporto con informazioni tecniche e operative
- **Knowledge Base proprietaria**: Documentazione tecnica riservata
- **Credenziali e chiavi API**: Accesso a servizi esterni (ITSM, CRM)
- **Modelli AI**: Prompt e output che potrebbero contenere PII

**Threat Model**:
- Accesso non autorizzato ai dati dei clienti (violazione GDPR)
- Exfiltration della knowledge base (IP theft)
- Credential compromise (lateral movement)
- Data tampering (integrity attacks)
- DDoS su API pubbliche

### Quando usare questa guida

- Configurazione iniziale dell'infrastruttura AWS
- Design di nuovi servizi o funzionalità
- Security review prima del deployment in produzione
- Risposta a incident di sicurezza
- Audit di compliance (ISO 27001, SOC 2, GDPR)

### Architettura High-Level

```
┌─────────────────────────────────────────────────────────┐
│                    SECURITY LAYERS                       │
├─────────────────────────────────────────────────────────┤
│ Layer 7: Application │ Guardrails, Input Validation     │
│ Layer 6: Identity    │ Cognito MFA, IAM Least Privilege │
│ Layer 5: Encryption  │ KMS, TLS 1.3, Secrets Manager   │
│ Layer 4: Network     │ Security Groups, NACLs, VPC     │
│ Layer 3: Edge        │ WAF, Shield, CloudFront         │
│ Layer 2: Detection   │ GuardDuty, CloudTrail, Config   │
│ Layer 1: Governance  │ SCPs, Automated Remediation     │
└─────────────────────────────────────────────────────────┘
```

---

## Concetti Fondamentali

### Defense in Depth

**Principio**: Implementare controlli di sicurezza ridondanti a più livelli. Se un livello fallisce, gli altri forniscono protezione.

**Esempio pratico**:
- API Gateway valida il payload JSON (Layer 7)
- Lambda verifica ulteriormente i caratteri pericolosi (Layer 7)
- IAM policy limita le azioni della Lambda (Layer 6)
- Security Group limita il traffico di rete (Layer 4)
- VPC Flow Logs monitora il traffico anomalo (Layer 2)

### Least Privilege Principle

**Principio**: Concedere solo i permessi minimi necessari per svolgere un compito.

**Anti-pattern**: `"Action": "*"`, `"Resource": "*"`

**Best practice**:
```json
{
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::kb-documents-prod/public/*"
}
```

### Zero Trust Model

**Principio**: Non fidarsi mai, verificare sempre. Anche il traffico interno alla VPC deve essere autenticato e autorizzato.

**Implementazione**:
- Mutual TLS tra microservizi
- Token JWT con scadenza breve
- VPC Endpoints per traffico AWS (no internet)
- Encryption in transit sempre abilitata

### Terminologia Chiave

| Termine | Definizione | Esempio |
|---------|-------------|---------|
| **Principal** | Entità che può fare richieste ad AWS | IAM User, IAM Role, Service |
| **Resource-based policy** | Policy attaccata a una risorsa | S3 Bucket Policy, KMS Key Policy |
| **Identity-based policy** | Policy attaccata a un'identità | IAM User Policy, Role Policy |
| **SCP** | Service Control Policy (Organizations) | Deny tutti tranne eu-south-1 |
| **Envelope Encryption** | Crittografia a due livelli (data key + master key) | KMS GenerateDataKey |
| **Grant** | Permesso temporaneo per usare una KMS key | Cross-account access |
| **ABAC** | Attribute-Based Access Control | Tag-based permissions |

### Pattern Comuni

#### 1. Assume Role Chain
```
User → AssumeRole → Role A → AssumeRole → Role B → Access Resource
```
Utile per: Cross-account access, elevation of privilege

#### 2. Service-Linked Role
Ruolo creato e gestito automaticamente da un servizio AWS (es. AWSServiceRoleForAPIGateway).

#### 3. Permission Boundary
Limite massimo di permessi che un'identità può avere, anche se le policy concedono di più.

#### 4. Condition Keys
Restrizioni aggiuntive basate su contesto della richiesta (IP, MFA, tempo, tag).

---

## Implementazione Pratica

### 1. IAM Policy Fine-Grained per Lambda

#### Setup Step-by-Step

**Obiettivo**: Creare una policy IAM che segue il principio di least privilege per una Lambda che processa ticket.

**Requisiti funzionali**:
- Leggere/scrivere su DynamoDB table `tickets`
- Leggere oggetti S3 da `kb-documents-prod`
- Invocare Bedrock modello Claude
- Scrivere log su CloudWatch
- Decrittare con KMS key `app-data-key`

**Policy completa**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DynamoDBTicketsAccess",
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:Query"
      ],
      "Resource": [
        "arn:aws:dynamodb:eu-south-1:123456789012:table/tickets",
        "arn:aws:dynamodb:eu-south-1:123456789012:table/tickets/index/*"
      ],
      "Condition": {
        "ForAllValues:StringEquals": {
          "dynamodb:Attributes": [
            "ticket_id",
            "customer_id",
            "status",
            "symptom_text",
            "solution",
            "created_at",
            "updated_at"
          ]
        }
      }
    },
    {
      "Sid": "S3KnowledgeBaseReadOnly",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion"
      ],
      "Resource": "arn:aws:s3:::kb-documents-prod/public/*",
      "Condition": {
        "StringEquals": {
          "s3:ExistingObjectTag/classification": "public"
        }
      }
    },
    {
      "Sid": "BedrockInvokeClaudeOnly",
      "Effect": "Allow",
      "Action": "bedrock:InvokeModel",
      "Resource": [
        "arn:aws:bedrock:eu-south-1::foundation-model/anthropic.claude-3-sonnet-*",
        "arn:aws:bedrock:eu-south-1::foundation-model/anthropic.claude-3-haiku-*"
      ]
    },
    {
      "Sid": "CloudWatchLogsWrite",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:eu-south-1:123456789012:log-group:/aws/lambda/ticket-processor:*"
    },
    {
      "Sid": "KMSDecryptOnly",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "arn:aws:kms:eu-south-1:123456789012:key/app-data-key-id",
      "Condition": {
        "StringEquals": {
          "kms:EncryptionContext:service": "ticket-processor"
        }
      }
    },
    {
      "Sid": "VPCNetworkInterfaces",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateNetworkInterface",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DeleteNetworkInterface"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ec2:Subnet": [
            "arn:aws:ec2:eu-south-1:123456789012:subnet/subnet-private-a",
            "arn:aws:ec2:eu-south-1:123456789012:subnet/subnet-private-b"
          ]
        }
      }
    }
  ]
}
```

**Best Practices applicate**:
1. ✅ **Explicit Sid**: Ogni statement ha un ID descrittivo
2. ✅ **Granular Actions**: Solo le azioni necessarie (no `dynamodb:*`)
3. ✅ **Resource ARN specifici**: No wildcard `*` dove possibile
4. ✅ **Condition Keys**: Restrizioni aggiuntive (tag, encryption context)
5. ✅ **Separate Statements**: Logica separata per facilità di manutenzione

**Codice Python per validare la policy**:

```python
import json
import boto3
from typing import Dict, List

class IAMPolicyValidator:
    """
    Valida IAM policies contro best practices di sicurezza.
    """

    def __init__(self):
        self.iam = boto3.client('iam')
        self.access_analyzer = boto3.client('accessanalyzer')

    def validate_policy(self, policy_document: Dict) -> List[Dict]:
        """
        Valida policy contro best practices.

        Returns:
            Lista di finding (warning/error)
        """
        findings = []

        for statement in policy_document.get('Statement', []):
            # Check 1: No wildcard actions
            if isinstance(statement.get('Action'), str):
                actions = [statement['Action']]
            else:
                actions = statement.get('Action', [])

            for action in actions:
                if action == '*' or action.endswith(':*'):
                    findings.append({
                        'severity': 'HIGH',
                        'type': 'WILDCARD_ACTION',
                        'message': f'Wildcard action detected: {action}',
                        'recommendation': 'Use specific actions instead'
                    })

            # Check 2: No wildcard resources
            resources = statement.get('Resource', [])
            if isinstance(resources, str):
                resources = [resources]

            if '*' in resources:
                findings.append({
                    'severity': 'MEDIUM',
                    'type': 'WILDCARD_RESOURCE',
                    'message': 'Wildcard resource detected',
                    'recommendation': 'Specify ARNs explicitly'
                })

            # Check 3: Condition keys usage
            if not statement.get('Condition'):
                findings.append({
                    'severity': 'LOW',
                    'type': 'MISSING_CONDITION',
                    'message': f'No condition keys in statement {statement.get("Sid")}',
                    'recommendation': 'Consider adding IP, MFA, or tag conditions'
                })

        return findings

    def simulate_policy(
        self,
        policy_document: Dict,
        action: str,
        resource_arn: str
    ) -> Dict:
        """
        Simula se una policy consentirebbe una specifica azione.

        Args:
            policy_document: Policy JSON
            action: AWS action (es. 's3:GetObject')
            resource_arn: ARN della risorsa

        Returns:
            Risultato della simulazione
        """
        response = self.iam.simulate_custom_policy(
            PolicyInputList=[json.dumps(policy_document)],
            ActionNames=[action],
            ResourceArns=[resource_arn]
        )

        result = response['EvaluationResults'][0]
        return {
            'action': action,
            'resource': resource_arn,
            'decision': result['EvalDecision'],
            'matched_statements': result.get('MatchedStatements', [])
        }

    def analyze_external_access(self, policy_arn: str) -> List[Dict]:
        """
        Analizza se la policy garantisce accesso esterno (cross-account).

        Usa AWS IAM Access Analyzer.
        """
        # Create analyzer if not exists
        analyzer_name = 'security-review-analyzer'

        try:
            self.access_analyzer.create_analyzer(
                analyzerName=analyzer_name,
                type='ACCOUNT'
            )
        except self.access_analyzer.exceptions.ConflictException:
            pass  # Analyzer already exists

        # Get findings for the policy
        findings = self.access_analyzer.list_findings(
            analyzerArn=f'arn:aws:access-analyzer:eu-south-1:123456789012:analyzer/{analyzer_name}',
            filter={
                'resource': {'eq': [policy_arn]}
            }
        )

        return findings['findings']

# Esempio d'uso
if __name__ == '__main__':
    validator = IAMPolicyValidator()

    # Leggi policy da file
    with open('lambda-ticket-processor-policy.json') as f:
        policy = json.load(f)

    # Valida
    findings = validator.validate_policy(policy)

    for finding in findings:
        print(f"[{finding['severity']}] {finding['type']}: {finding['message']}")
        print(f"  → {finding['recommendation']}\n")

    # Simula azione
    result = validator.simulate_policy(
        policy_document=policy,
        action='s3:GetObject',
        resource_arn='arn:aws:s3:::kb-documents-prod/public/manual-001.pdf'
    )

    print(f"Simulation: {result['action']} on {result['resource']}")
    print(f"Decision: {result['decision']}")
```

**Testing della policy**:

```bash
# 1. Validate syntax
aws iam get-policy-version --policy-arn arn:aws:iam::123456789012:policy/LambdaTicketProcessorPolicy --version-id v1

# 2. Simulate actions
aws iam simulate-custom-policy \
  --policy-input-list file://policy.json \
  --action-names s3:GetObject bedrock:InvokeModel dynamodb:PutItem \
  --resource-arns \
    "arn:aws:s3:::kb-documents-prod/public/*" \
    "arn:aws:bedrock:eu-south-1::foundation-model/anthropic.claude-3-sonnet-*" \
    "arn:aws:dynamodb:eu-south-1:123456789012:table/tickets"

# 3. Check with Access Analyzer
aws accessanalyzer create-analyzer --analyzer-name security-review --type ACCOUNT
aws accessanalyzer start-policy-generation --policy-generation-details file://generation-config.json
```

---

### 2. Condition Keys Avanzate

#### IP Restrictions

Limita l'accesso alle API solo da IP aziendali:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowFromCorporateNetworkOnly",
      "Effect": "Allow",
      "Action": "execute-api:Invoke",
      "Resource": "arn:aws:execute-api:eu-south-1:123456789012:api-id/*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": [
            "203.0.113.0/24",
            "198.51.100.0/24"
          ]
        }
      }
    },
    {
      "Sid": "DenyAccessFromBlockedCountries",
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": [
            "us-east-1",
            "us-west-2"
          ]
        },
        "StringNotEquals": {
          "aws:PrincipalAccount": "123456789012"
        }
      }
    }
  ]
}
```

#### MFA Enforcement

Richiedi MFA per operazioni sensibili:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAllActionsWithMFA",
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    },
    {
      "Sid": "DenyDestructiveActionsWithoutMFA",
      "Effect": "Deny",
      "Action": [
        "s3:DeleteBucket",
        "dynamodb:DeleteTable",
        "rds:DeleteDBInstance",
        "kms:ScheduleKeyDeletion"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

#### Tag-Based Access Control (ABAC)

Concedi accesso solo a risorse con specifici tag:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAccessToOwnedResources",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::customer-data-prod/*",
      "Condition": {
        "StringEquals": {
          "s3:ExistingObjectTag/owner": "${aws:userid}",
          "s3:ExistingObjectTag/classification": "internal"
        }
      }
    },
    {
      "Sid": "RequireTagsOnCreate",
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::customer-data-prod/*",
      "Condition": {
        "StringEquals": {
          "s3:RequestObjectTag/classification": ["public", "internal", "confidential"],
          "s3:RequestObjectTag/owner": "${aws:userid}"
        },
        "ForAllValues:StringEquals": {
          "s3:RequestObjectTagKeys": ["classification", "owner", "project"]
        }
      }
    }
  ]
}
```

#### Time-Based Access

Consenti accesso solo in orari lavorativi:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowDuringBusinessHours",
      "Effect": "Allow",
      "Action": "dynamodb:*",
      "Resource": "arn:aws:dynamodb:*:*:table/tickets",
      "Condition": {
        "DateGreaterThan": {
          "aws:CurrentTime": "2024-01-01T08:00:00Z"
        },
        "DateLessThan": {
          "aws:CurrentTime": "2024-12-31T18:00:00Z"
        },
        "StringEquals": {
          "aws:RequestedRegion": "eu-south-1"
        }
      }
    }
  ]
}
```

#### Service-Specific Conditions

**S3 - Enforce encryption**:
```json
{
  "Sid": "DenyUnencryptedObjectUploads",
  "Effect": "Deny",
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::kb-documents-prod/*",
  "Condition": {
    "StringNotEquals": {
      "s3:x-amz-server-side-encryption": "aws:kms"
    }
  }
}
```

**Lambda - Restrict function URLs**:
```json
{
  "Sid": "AllowLambdaInvokeFromVPCOnly",
  "Effect": "Allow",
  "Action": "lambda:InvokeFunction",
  "Resource": "arn:aws:lambda:eu-south-1:123456789012:function:ticket-processor",
  "Condition": {
    "StringEquals": {
      "lambda:FunctionUrlAuthType": "AWS_IAM"
    }
  }
}
```

**Codice Python per testare Condition Keys**:

```python
import boto3
import json
from datetime import datetime, timezone
from typing import Dict, Any

class ConditionKeyTester:
    """
    Testa condition keys in IAM policies.
    """

    def __init__(self):
        self.sts = boto3.client('sts')
        self.iam = boto3.client('iam')

    def check_mfa_status(self) -> Dict[str, Any]:
        """
        Verifica se la sessione corrente ha MFA abilitata.
        """
        caller_identity = self.sts.get_caller_identity()

        # Check session token for MFA
        session_context = self.sts.get_session_token()

        return {
            'account': caller_identity['Account'],
            'user_id': caller_identity['UserId'],
            'arn': caller_identity['Arn'],
            'mfa_authenticated': 'mfa' in caller_identity.get('Arn', '').lower()
        }

    def simulate_condition_evaluation(
        self,
        policy: Dict,
        context: Dict[str, str]
    ) -> bool:
        """
        Simula la valutazione di condition keys.

        Args:
            policy: Policy statement
            context: Context variables (aws:SourceIp, aws:CurrentTime, etc.)

        Returns:
            True se le condizioni sono soddisfatte
        """
        conditions = policy.get('Condition', {})

        for operator, condition_block in conditions.items():
            if operator == 'IpAddress':
                # Valuta aws:SourceIp
                source_ip = context.get('aws:SourceIp')
                allowed_cidrs = condition_block.get('aws:SourceIp', [])
                if not self._ip_in_cidrs(source_ip, allowed_cidrs):
                    return False

            elif operator == 'BoolIfExists':
                # Valuta aws:MultiFactorAuthPresent
                for key, expected in condition_block.items():
                    if key == 'aws:MultiFactorAuthPresent':
                        mfa_present = context.get('aws:MultiFactorAuthPresent', 'false')
                        if mfa_present != expected:
                            return False

            elif operator == 'StringEquals':
                # Valuta string equality
                for key, expected_values in condition_block.items():
                    actual_value = context.get(key)
                    if isinstance(expected_values, list):
                        if actual_value not in expected_values:
                            return False
                    else:
                        if actual_value != expected_values:
                            return False

            elif operator == 'DateGreaterThan' or operator == 'DateLessThan':
                # Valuta time-based conditions
                current_time = datetime.fromisoformat(
                    context.get('aws:CurrentTime', datetime.now(timezone.utc).isoformat())
                )
                for key, threshold_str in condition_block.items():
                    threshold = datetime.fromisoformat(threshold_str.replace('Z', '+00:00'))
                    if operator == 'DateGreaterThan' and current_time <= threshold:
                        return False
                    elif operator == 'DateLessThan' and current_time >= threshold:
                        return False

        return True

    def _ip_in_cidrs(self, ip: str, cidrs: list) -> bool:
        """Helper: verifica se IP è in lista di CIDR."""
        import ipaddress
        ip_obj = ipaddress.ip_address(ip)
        for cidr in cidrs:
            if ip_obj in ipaddress.ip_network(cidr):
                return True
        return False

    def test_policy_with_contexts(
        self,
        policy: Dict,
        test_cases: list
    ) -> list:
        """
        Testa policy con diversi contesti.

        Args:
            policy: Policy document
            test_cases: Lista di scenari con context e expected result

        Returns:
            Risultati dei test
        """
        results = []

        for test_case in test_cases:
            context = test_case['context']
            expected = test_case['expected']
            description = test_case['description']

            actual = self.simulate_condition_evaluation(
                policy['Statement'][0],
                context
            )

            results.append({
                'description': description,
                'context': context,
                'expected': expected,
                'actual': actual,
                'passed': actual == expected
            })

        return results

# Esempio d'uso
if __name__ == '__main__':
    tester = ConditionKeyTester()

    # Policy con MFA e IP restrictions
    policy = {
        "Version": "2012-10-17",
        "Statement": [{
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": "*",
            "Condition": {
                "IpAddress": {
                    "aws:SourceIp": ["203.0.113.0/24"]
                },
                "BoolIfExists": {
                    "aws:MultiFactorAuthPresent": "true"
                }
            }
        }]
    }

    # Test cases
    test_cases = [
        {
            'description': 'Valid IP + MFA',
            'context': {
                'aws:SourceIp': '203.0.113.50',
                'aws:MultiFactorAuthPresent': 'true'
            },
            'expected': True
        },
        {
            'description': 'Valid IP but no MFA',
            'context': {
                'aws:SourceIp': '203.0.113.50',
                'aws:MultiFactorAuthPresent': 'false'
            },
            'expected': False
        },
        {
            'description': 'Invalid IP + MFA',
            'context': {
                'aws:SourceIp': '198.51.100.50',
                'aws:MultiFactorAuthPresent': 'true'
            },
            'expected': False
        }
    ]

    results = tester.test_policy_with_contexts(policy, test_cases)

    for result in results:
        status = '✅ PASS' if result['passed'] else '❌ FAIL'
        print(f"{status} - {result['description']}")
        print(f"  Expected: {result['expected']}, Got: {result['actual']}\n")
```

---

### 3. KMS Key Management

#### KMS Key Policy per Cross-Account Access

**Scenario**: Account A (123456789012) deve consentire ad Account B (987654321098) di usare una KMS key per decrittare dati S3.

**Key Policy completa**:

```json
{
  "Version": "2012-10-17",
  "Id": "cross-account-key-policy",
  "Statement": [
    {
      "Sid": "Enable IAM User Permissions",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "kms:*",
      "Resource": "*"
    },
    {
      "Sid": "Allow use of the key from Account B",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::987654321098:role/DataProcessorRole"
      },
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey",
        "kms:GenerateDataKey"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": [
            "s3.eu-south-1.amazonaws.com",
            "dynamodb.eu-south-1.amazonaws.com"
          ],
          "kms:EncryptionContext:project": "ai-support-system"
        }
      }
    },
    {
      "Sid": "Allow attachment of persistent resources",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::987654321098:role/DataProcessorRole"
      },
      "Action": [
        "kms:CreateGrant",
        "kms:ListGrants",
        "kms:RevokeGrant"
      ],
      "Resource": "*",
      "Condition": {
        "Bool": {
          "kms:GrantIsForAWSResource": "true"
        }
      }
    },
    {
      "Sid": "Deny key deletion",
      "Effect": "Deny",
      "Principal": "*",
      "Action": [
        "kms:ScheduleKeyDeletion",
        "kms:DeleteAlias"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Enable CloudTrail logging",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudtrail.amazonaws.com"
      },
      "Action": [
        "kms:GenerateDataKey",
        "kms:Decrypt"
      ],
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "kms:EncryptionContext:aws:cloudtrail:arn": "arn:aws:cloudtrail:*:123456789012:trail/*"
        }
      }
    }
  ]
}
```

**CloudFormation per KMS Key + Alias**:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: KMS key for cross-account data encryption

Parameters:
  TrustedAccountId:
    Type: String
    Description: AWS Account ID to grant access
    Default: '987654321098'

  TrustedRoleName:
    Type: String
    Description: IAM Role name in trusted account
    Default: 'DataProcessorRole'

Resources:
  AppDataKey:
    Type: AWS::KMS::Key
    Properties:
      Description: Customer-managed key for application data encryption
      KeyPolicy:
        Version: '2012-10-17'
        Statement:
          - Sid: Enable IAM User Permissions
            Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:root'
            Action: 'kms:*'
            Resource: '*'

          - Sid: Allow use from trusted account
            Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${TrustedAccountId}:role/${TrustedRoleName}'
            Action:
              - 'kms:Decrypt'
              - 'kms:DescribeKey'
              - 'kms:GenerateDataKey'
            Resource: '*'
            Condition:
              StringEquals:
                kms:ViaService:
                  - !Sub 's3.${AWS::Region}.amazonaws.com'
                  - !Sub 'dynamodb.${AWS::Region}.amazonaws.com'

          - Sid: Allow grant creation
            Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${TrustedAccountId}:role/${TrustedRoleName}'
            Action:
              - 'kms:CreateGrant'
              - 'kms:ListGrants'
              - 'kms:RevokeGrant'
            Resource: '*'
            Condition:
              Bool:
                kms:GrantIsForAWSResource: true

      KeySpec: SYMMETRIC_DEFAULT
      KeyUsage: ENCRYPT_DECRYPT
      Origin: AWS_KMS
      MultiRegion: false
      EnableKeyRotation: true
      Tags:
        - Key: Name
          Value: app-data-key
        - Key: Environment
          Value: production
        - Key: ManagedBy
          Value: CloudFormation

  AppDataKeyAlias:
    Type: AWS::KMS::Alias
    Properties:
      AliasName: alias/app-data-key
      TargetKeyId: !Ref AppDataKey

  AppDataKeyArnParameter:
    Type: AWS::SSM::Parameter
    Properties:
      Name: /kms/app-data-key/arn
      Type: String
      Value: !GetAtt AppDataKey.Arn
      Description: ARN of app-data-key KMS key
      Tags:
        Environment: production

Outputs:
  KeyId:
    Description: KMS Key ID
    Value: !Ref AppDataKey
    Export:
      Name: !Sub '${AWS::StackName}-KeyId'

  KeyArn:
    Description: KMS Key ARN
    Value: !GetAtt AppDataKey.Arn
    Export:
      Name: !Sub '${AWS::StackName}-KeyArn'

  KeyAlias:
    Description: KMS Key Alias
    Value: !Ref AppDataKeyAlias
    Export:
      Name: !Sub '${AWS::StackName}-KeyAlias'
```

**Codice Python per gestire KMS Keys**:

```python
import boto3
import json
from typing import Dict, Optional, List

class KMSManager:
    """
    Gestisce operazioni KMS: creazione key, grants, encryption/decryption.
    """

    def __init__(self, region: str = 'eu-south-1'):
        self.kms = boto3.client('kms', region_name=region)
        self.region = region

    def create_key_with_policy(
        self,
        description: str,
        key_policy: Dict,
        enable_rotation: bool = True
    ) -> Dict:
        """
        Crea una KMS key con policy personalizzata.

        Args:
            description: Descrizione della key
            key_policy: Policy JSON
            enable_rotation: Abilita auto-rotation annuale

        Returns:
            Key metadata
        """
        response = self.kms.create_key(
            Description=description,
            KeyUsage='ENCRYPT_DECRYPT',
            Origin='AWS_KMS',
            MultiRegion=False,
            Policy=json.dumps(key_policy),
            Tags=[
                {'TagKey': 'Environment', 'TagValue': 'production'},
                {'TagKey': 'ManagedBy', 'TagValue': 'automation'}
            ]
        )

        key_id = response['KeyMetadata']['KeyId']

        # Enable automatic rotation
        if enable_rotation:
            self.kms.enable_key_rotation(KeyId=key_id)

        return response['KeyMetadata']

    def create_grant(
        self,
        key_id: str,
        grantee_principal: str,
        operations: List[str],
        encryption_context: Optional[Dict[str, str]] = None
    ) -> str:
        """
        Crea un grant per consentire accesso temporaneo alla key.

        Args:
            key_id: KMS Key ID o ARN
            grantee_principal: ARN del principal (role, user)
            operations: Lista di operazioni (Decrypt, Encrypt, etc.)
            encryption_context: Encryption context constraints

        Returns:
            Grant ID
        """
        grant_params = {
            'KeyId': key_id,
            'GranteePrincipal': grantee_principal,
            'Operations': operations
        }

        if encryption_context:
            grant_params['Constraints'] = {
                'EncryptionContextSubset': encryption_context
            }

        response = self.kms.create_grant(**grant_params)
        return response['GrantId']

    def encrypt_data(
        self,
        key_id: str,
        plaintext: bytes,
        encryption_context: Optional[Dict[str, str]] = None
    ) -> Dict:
        """
        Cripta dati con KMS key.

        Args:
            key_id: KMS Key ID, ARN, o alias
            plaintext: Dati da crittare (max 4KB)
            encryption_context: Context per validazione decrypt

        Returns:
            Ciphertext + metadata
        """
        encrypt_params = {
            'KeyId': key_id,
            'Plaintext': plaintext
        }

        if encryption_context:
            encrypt_params['EncryptionContext'] = encryption_context

        response = self.kms.encrypt(**encrypt_params)

        return {
            'ciphertext': response['CiphertextBlob'],
            'key_id': response['KeyId'],
            'encryption_algorithm': response.get('EncryptionAlgorithm', 'SYMMETRIC_DEFAULT')
        }

    def decrypt_data(
        self,
        ciphertext: bytes,
        encryption_context: Optional[Dict[str, str]] = None
    ) -> bytes:
        """
        Decripta dati. KMS identifica automaticamente la key dal ciphertext.

        Args:
            ciphertext: Dati crittati
            encryption_context: Deve matchare quello usato in encrypt

        Returns:
            Plaintext
        """
        decrypt_params = {'CiphertextBlob': ciphertext}

        if encryption_context:
            decrypt_params['EncryptionContext'] = encryption_context

        response = self.kms.decrypt(**decrypt_params)
        return response['Plaintext']

    def generate_data_key(
        self,
        key_id: str,
        key_spec: str = 'AES_256',
        encryption_context: Optional[Dict[str, str]] = None
    ) -> Dict:
        """
        Genera data key per envelope encryption.

        Envelope encryption pattern:
        1. Genera data key con questa funzione
        2. Usa plaintext data key per crittare i dati (localmente)
        3. Salva encrypted data key insieme ai dati
        4. Distruggi plaintext data key dalla memoria
        5. Per decrypt: decripta data key con KMS, poi decripta dati

        Args:
            key_id: KMS Key ID
            key_spec: AES_256 o AES_128
            encryption_context: Context

        Returns:
            Plaintext data key + encrypted data key
        """
        generate_params = {
            'KeyId': key_id,
            'KeySpec': key_spec
        }

        if encryption_context:
            generate_params['EncryptionContext'] = encryption_context

        response = self.kms.generate_data_key(**generate_params)

        return {
            'plaintext_key': response['Plaintext'],  # Use and destroy immediately
            'encrypted_key': response['CiphertextBlob'],  # Store with data
            'key_id': response['KeyId']
        }

    def rotate_key(self, key_id: str):
        """
        Abilita automatic annual key rotation.
        """
        self.kms.enable_key_rotation(KeyId=key_id)

    def schedule_key_deletion(
        self,
        key_id: str,
        pending_days: int = 30
    ):
        """
        Schedula eliminazione key (7-30 giorni).

        ATTENZIONE: Operazione irreversibile. I dati crittati con questa key
        diventeranno irrecuperabili.
        """
        self.kms.schedule_key_deletion(
            KeyId=key_id,
            PendingWindowInDays=pending_days
        )

    def list_grants(self, key_id: str) -> List[Dict]:
        """
        Lista tutti i grants per una key.
        """
        response = self.kms.list_grants(KeyId=key_id)
        return response['Grants']

    def revoke_grant(self, key_id: str, grant_id: str):
        """
        Revoca un grant.
        """
        self.kms.revoke_grant(KeyId=key_id, GrantId=grant_id)

# Esempio d'uso
if __name__ == '__main__':
    manager = KMSManager(region='eu-south-1')

    # 1. Create key
    key_policy = {
        "Version": "2012-10-17",
        "Statement": [{
            "Sid": "Enable IAM User Permissions",
            "Effect": "Allow",
            "Principal": {"AWS": "arn:aws:iam::123456789012:root"},
            "Action": "kms:*",
            "Resource": "*"
        }]
    }

    key_metadata = manager.create_key_with_policy(
        description="Data encryption key for ticket processor",
        key_policy=key_policy,
        enable_rotation=True
    )

    key_id = key_metadata['KeyId']
    print(f"Created key: {key_id}")

    # 2. Create grant for cross-account access
    grant_id = manager.create_grant(
        key_id=key_id,
        grantee_principal="arn:aws:iam::987654321098:role/DataProcessorRole",
        operations=['Decrypt', 'DescribeKey'],
        encryption_context={'project': 'ai-support'}
    )

    print(f"Created grant: {grant_id}")

    # 3. Encrypt data
    plaintext = b"Sensitive customer data"
    encryption_context = {
        'service': 'ticket-processor',
        'customer_id': 'cust-12345'
    }

    encrypted = manager.encrypt_data(
        key_id=key_id,
        plaintext=plaintext,
        encryption_context=encryption_context
    )

    print(f"Encrypted {len(plaintext)} bytes")

    # 4. Decrypt data
    decrypted = manager.decrypt_data(
        ciphertext=encrypted['ciphertext'],
        encryption_context=encryption_context
    )

    assert decrypted == plaintext
    print("Decrypt successful")

    # 5. Generate data key for envelope encryption
    data_key = manager.generate_data_key(
        key_id=key_id,
        encryption_context=encryption_context
    )

    # Use data_key['plaintext_key'] to encrypt large data locally
    # Store data_key['encrypted_key'] with the encrypted data

    print("Data key generated for envelope encryption")
```

---

### 4. Secrets Manager Rotation

#### Lambda Function per Secrets Rotation

**Scenario**: Rotazione automatica della password RDS ogni 30 giorni.

**Codice Lambda completo**:

```python
import boto3
import json
import logging
import os
import random
import string
from typing import Dict

logger = logging.getLogger()
logger.setLevel(logging.INFO)

# AWS clients
secretsmanager = boto3.client('secretsmanager')
rds = boto3.client('rds')

def lambda_handler(event, context):
    """
    AWS Secrets Manager rotation lambda handler.

    Event structure:
    {
        "Step": "createSecret" | "setSecret" | "testSecret" | "finishSecret",
        "SecretId": "arn:aws:secretsmanager:...",
        "ClientRequestToken": "uuid",
        "Token": "uuid"
    }
    """
    secret_arn = event['SecretId']
    token = event['ClientRequestToken']
    step = event['Step']

    logger.info(f"Executing rotation step: {step} for secret {secret_arn}")

    # Get secret metadata
    metadata = secretsmanager.describe_secret(SecretId=secret_arn)

    if step == "createSecret":
        create_secret(secretsmanager, secret_arn, token)

    elif step == "setSecret":
        set_secret(secretsmanager, secret_arn, token)

    elif step == "testSecret":
        test_secret(secretsmanager, secret_arn, token)

    elif step == "finishSecret":
        finish_secret(secretsmanager, secret_arn, token)

    else:
        raise ValueError(f"Invalid step: {step}")

    logger.info(f"Successfully completed step: {step}")

def create_secret(client, arn: str, token: str):
    """
    Step 1: Genera nuovo password e lo salva in AWSPENDING.
    """
    # Check if AWSPENDING version already exists
    try:
        client.get_secret_value(SecretId=arn, VersionStage="AWSPENDING", VersionId=token)
        logger.info("AWSPENDING version already exists, skipping creation")
        return
    except client.exceptions.ResourceNotFoundException:
        pass

    # Get current secret
    current_secret = client.get_secret_value(SecretId=arn, VersionStage="AWSCURRENT")
    current_dict = json.loads(current_secret['SecretString'])

    # Generate new password
    new_password = generate_secure_password()

    # Create new secret version
    new_dict = current_dict.copy()
    new_dict['password'] = new_password

    client.put_secret_value(
        SecretId=arn,
        ClientRequestToken=token,
        SecretString=json.dumps(new_dict),
        VersionStages=['AWSPENDING']
    )

    logger.info("Created new secret version with AWSPENDING stage")

def set_secret(client, arn: str, token: str):
    """
    Step 2: Aggiorna la password nel database RDS.
    """
    # Get pending secret
    pending_secret = client.get_secret_value(SecretId=arn, VersionStage="AWSPENDING", VersionId=token)
    pending_dict = json.loads(pending_secret['SecretString'])

    # Get current secret for connection
    current_secret = client.get_secret_value(SecretId=arn, VersionStage="AWSCURRENT")
    current_dict = json.loads(current_secret['SecretString'])

    # Update RDS password
    update_rds_password(
        host=current_dict['host'],
        port=current_dict['port'],
        username=current_dict['username'],
        current_password=current_dict['password'],
        new_password=pending_dict['password']
    )

    logger.info("Successfully updated database password")

def test_secret(client, arn: str, token: str):
    """
    Step 3: Testa la nuova password.
    """
    # Get pending secret
    pending_secret = client.get_secret_value(SecretId=arn, VersionStage="AWSPENDING", VersionId=token)
    pending_dict = json.loads(pending_secret['SecretString'])

    # Test connection with new password
    test_database_connection(
        host=pending_dict['host'],
        port=pending_dict['port'],
        username=pending_dict['username'],
        password=pending_dict['password'],
        database=pending_dict.get('dbname', 'postgres')
    )

    logger.info("Successfully tested new secret")

def finish_secret(client, arn: str, token: str):
    """
    Step 4: Promuove AWSPENDING a AWSCURRENT.
    """
    # Get current version
    metadata = client.describe_secret(SecretId=arn)
    current_version = None

    for version, stages in metadata['VersionIdsToStages'].items():
        if 'AWSCURRENT' in stages:
            current_version = version
            break

    # Move AWSCURRENT stage to AWSPENDING version
    client.update_secret_version_stage(
        SecretId=arn,
        VersionStage='AWSCURRENT',
        MoveToVersionId=token,
        RemoveFromVersionId=current_version
    )

    logger.info("Successfully finished secret rotation")

def generate_secure_password(length: int = 32) -> str:
    """
    Genera password sicura.

    Requisiti:
    - Almeno 1 maiuscola
    - Almeno 1 minuscola
    - Almeno 1 numero
    - Almeno 1 carattere speciale
    - Lunghezza >= 32
    """
    lowercase = string.ascii_lowercase
    uppercase = string.ascii_uppercase
    digits = string.digits
    special = '!@#$%^&*()_+-=[]{}|'

    # Garantisce requisiti minimi
    password = [
        random.choice(lowercase),
        random.choice(uppercase),
        random.choice(digits),
        random.choice(special)
    ]

    # Riempie il resto
    all_chars = lowercase + uppercase + digits + special
    password += [random.choice(all_chars) for _ in range(length - 4)]

    # Shuffle
    random.shuffle(password)

    return ''.join(password)

def update_rds_password(
    host: str,
    port: int,
    username: str,
    current_password: str,
    new_password: str
):
    """
    Aggiorna password RDS usando connessione diretta.
    """
    import psycopg2  # For PostgreSQL

    try:
        # Connect with current credentials
        conn = psycopg2.connect(
            host=host,
            port=port,
            user=username,
            password=current_password,
            database='postgres',
            connect_timeout=5
        )

        conn.autocommit = True
        cursor = conn.cursor()

        # Change password
        cursor.execute(
            f"ALTER USER {username} WITH PASSWORD %s",
            (new_password,)
        )

        cursor.close()
        conn.close()

        logger.info(f"Password updated for user {username}")

    except Exception as e:
        logger.error(f"Failed to update password: {str(e)}")
        raise

def test_database_connection(
    host: str,
    port: int,
    username: str,
    password: str,
    database: str
):
    """
    Testa connessione al database con nuove credenziali.
    """
    import psycopg2

    try:
        conn = psycopg2.connect(
            host=host,
            port=port,
            user=username,
            password=password,
            database=database,
            connect_timeout=5
        )

        cursor = conn.cursor()
        cursor.execute("SELECT 1")
        cursor.fetchone()

        cursor.close()
        conn.close()

        logger.info("Database connection test successful")

    except Exception as e:
        logger.error(f"Database connection test failed: {str(e)}")
        raise
```

**CloudFormation per Secret + Rotation**:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: RDS credentials with automatic rotation

Parameters:
  DatabaseEndpoint:
    Type: String
    Description: RDS endpoint hostname

  DatabasePort:
    Type: Number
    Default: 5432

  DatabaseName:
    Type: String
    Default: app_db

  RotationLambdaArn:
    Type: String
    Description: ARN of rotation Lambda function

Resources:
  RDSCredentialsSecret:
    Type: AWS::SecretsManager::Secret
    Properties:
      Name: prod/rds/credentials
      Description: RDS database credentials with auto-rotation
      SecretString: !Sub |
        {
          "username": "app_user",
          "password": "${GeneratedPassword}",
          "host": "${DatabaseEndpoint}",
          "port": ${DatabasePort},
          "dbname": "${DatabaseName}"
        }
      KmsKeyId: !ImportValue AppDataKeyId
      Tags:
        - Key: Environment
          Value: production
        - Key: Service
          Value: ticket-processor

  GeneratedPassword:
    Type: AWS::SecretsManager::Secret
    Properties:
      GenerateSecretString:
        SecretStringTemplate: '{}'
        GenerateStringKey: "password"
        PasswordLength: 32
        ExcludeCharacters: '"@\\'
        RequireEachIncludedType: true

  SecretRotationSchedule:
    Type: AWS::SecretsManager::RotationSchedule
    Properties:
      SecretId: !Ref RDSCredentialsSecret
      RotationLambdaARN: !Ref RotationLambdaArn
      RotationRules:
        AutomaticallyAfterDays: 30
        Duration: 2h
        ScheduleExpression: 'rate(30 days)'

  RotationLambdaPermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref RotationLambdaArn
      Action: lambda:InvokeFunction
      Principal: secretsmanager.amazonaws.com

Outputs:
  SecretArn:
    Description: ARN of the RDS credentials secret
    Value: !Ref RDSCredentialsSecret
    Export:
      Name: !Sub '${AWS::StackName}-SecretArn'
```

**CloudFormation per Rotation Lambda**:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: Lambda function for Secrets Manager rotation

Resources:
  RotationLambdaRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole
      Policies:
        - PolicyName: SecretsManagerRotation
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - secretsmanager:DescribeSecret
                  - secretsmanager:GetSecretValue
                  - secretsmanager:PutSecretValue
                  - secretsmanager:UpdateSecretVersionStage
                Resource: !Sub 'arn:aws:secretsmanager:${AWS::Region}:${AWS::AccountId}:secret:prod/rds/*'

              - Effect: Allow
                Action:
                  - secretsmanager:GetRandomPassword
                Resource: '*'

              - Effect: Allow
                Action:
                  - kms:Decrypt
                  - kms:GenerateDataKey
                Resource: !ImportValue AppDataKeyArn

  RotationLambdaFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: rds-secrets-rotation
      Runtime: python3.11
      Handler: index.lambda_handler
      Role: !GetAtt RotationLambdaRole.Arn
      Timeout: 30
      VpcConfig:
        SecurityGroupIds:
          - !ImportValue PrivateSecurityGroupId
        SubnetIds:
          - !ImportValue PrivateSubnetA
          - !ImportValue PrivateSubnetB
      Code:
        ZipFile: |
          # Paste the rotation Lambda code here
          # (The code from above)
      Environment:
        Variables:
          EXCLUDE_CHARACTERS: '"@\'

Outputs:
  FunctionArn:
    Description: ARN of rotation Lambda
    Value: !GetAtt RotationLambdaFunction.Arn
    Export:
      Name: RotationLambdaArn
```

---

### 5. VPC Endpoints (PrivateLink)

#### Setup VPC Endpoints per servizi AWS

**Scenario**: Configurare VPC Endpoints per evitare traffico internet e migliorare sicurezza.

**CloudFormation Template completo**:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: VPC Endpoints for AWS services (PrivateLink)

Parameters:
  VpcId:
    Type: AWS::EC2::VPC::Id
    Description: VPC ID where endpoints will be created

  PrivateSubnetIds:
    Type: List<AWS::EC2::Subnet::Id>
    Description: Private subnet IDs for interface endpoints

  PrivateRouteTableIds:
    Type: CommaDelimitedList
    Description: Route table IDs for gateway endpoints

Resources:
  # Security Group for VPC Endpoints
  VPCEndpointSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: vpc-endpoints-sg
      GroupDescription: Security group for VPC endpoints
      VpcId: !Ref VpcId
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          SourceSecurityGroupId: !ImportValue LambdaSecurityGroupId
          Description: HTTPS from Lambda functions

        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 10.0.0.0/16
          Description: HTTPS from VPC CIDR

      SecurityGroupEgress:
        - IpProtocol: -1
          CidrIp: 0.0.0.0/0
          Description: Allow all outbound

      Tags:
        - Key: Name
          Value: vpc-endpoints-sg

  # Gateway Endpoints (no cost)
  S3GatewayEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VpcId
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.s3'
      VpcEndpointType: Gateway
      RouteTableIds: !Ref PrivateRouteTableIds
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal: '*'
            Action:
              - 's3:GetObject'
              - 's3:PutObject'
              - 's3:ListBucket'
            Resource:
              - 'arn:aws:s3:::kb-documents-prod/*'
              - 'arn:aws:s3:::raw-data-prod/*'
              - 'arn:aws:s3:::processed-data-prod/*'

  DynamoDBGatewayEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VpcId
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.dynamodb'
      VpcEndpointType: Gateway
      RouteTableIds: !Ref PrivateRouteTableIds
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal: '*'
            Action:
              - 'dynamodb:GetItem'
              - 'dynamodb:PutItem'
              - 'dynamodb:UpdateItem'
              - 'dynamodb:Query'
              - 'dynamodb:Scan'
            Resource:
              - !Sub 'arn:aws:dynamodb:${AWS::Region}:${AWS::AccountId}:table/tickets'
              - !Sub 'arn:aws:dynamodb:${AWS::Region}:${AWS::AccountId}:table/feedback'

  # Interface Endpoints ($0.01/hour each + data transfer)
  BedrockRuntimeEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VpcId
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.bedrock-runtime'
      VpcEndpointType: Interface
      SubnetIds: !Ref PrivateSubnetIds
      SecurityGroupIds:
        - !Ref VPCEndpointSecurityGroup
      PrivateDnsEnabled: true
      PolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal: '*'
            Action:
              - 'bedrock:InvokeModel'
              - 'bedrock:InvokeModelWithResponseStream'
            Resource:
              - !Sub 'arn:aws:bedrock:${AWS::Region}::foundation-model/anthropic.claude-*'

  SecretsManagerEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VpcId
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.secretsmanager'
      VpcEndpointType: Interface
      SubnetIds: !Ref PrivateSubnetIds
      SecurityGroupIds:
        - !Ref VPCEndpointSecurityGroup
      PrivateDnsEnabled: true

  CloudWatchLogsEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VpcId
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.logs'
      VpcEndpointType: Interface
      SubnetIds: !Ref PrivateSubnetIds
      SecurityGroupIds:
        - !Ref VPCEndpointSecurityGroup
      PrivateDnsEnabled: true

  SSMEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VpcId
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.ssm'
      VpcEndpointType: Interface
      SubnetIds: !Ref PrivateSubnetIds
      SecurityGroupIds:
        - !Ref VPCEndpointSecurityGroup
      PrivateDnsEnabled: true

  ECRAPIEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VpcId
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.ecr.api'
      VpcEndpointType: Interface
      SubnetIds: !Ref PrivateSubnetIds
      SecurityGroupIds:
        - !Ref VPCEndpointSecurityGroup
      PrivateDnsEnabled: true

  ECRDKREndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VpcId
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.ecr.dkr'
      VpcEndpointType: Interface
      SubnetIds: !Ref PrivateSubnetIds
      SecurityGroupIds:
        - !Ref VPCEndpointSecurityGroup
      PrivateDnsEnabled: true

  SageMakerRuntimeEndpoint:
    Type: AWS::EC2::VPCEndpoint
    Properties:
      VpcId: !Ref VpcId
      ServiceName: !Sub 'com.amazonaws.${AWS::Region}.sagemaker.runtime'
      VpcEndpointType: Interface
      SubnetIds: !Ref PrivateSubnetIds
      SecurityGroupIds:
        - !Ref VPCEndpointSecurityGroup
      PrivateDnsEnabled: true

Outputs:
  S3EndpointId:
    Description: S3 Gateway Endpoint ID
    Value: !Ref S3GatewayEndpoint
    Export:
      Name: S3GatewayEndpointId

  BedrockEndpointDNS:
    Description: Bedrock VPC Endpoint DNS names
    Value: !Select [0, !GetAtt BedrockRuntimeEndpoint.DnsEntries]
```

**Codice Python per testare VPC Endpoints**:

```python
import boto3
import socket
from typing import List, Dict

class VPCEndpointTester:
    """
    Testa connettività e funzionalità VPC endpoints.
    """

    def __init__(self, region: str = 'eu-south-1'):
        self.ec2 = boto3.client('ec2', region_name=region)
        self.region = region

    def list_endpoints(self, vpc_id: str) -> List[Dict]:
        """
        Lista tutti i VPC endpoints in una VPC.
        """
        response = self.ec2.describe_vpc_endpoints(
            Filters=[
                {'Name': 'vpc-id', 'Values': [vpc_id]}
            ]
        )

        endpoints = []
        for endpoint in response['VpcEndpoints']:
            endpoints.append({
                'id': endpoint['VpcEndpointId'],
                'service': endpoint['ServiceName'],
                'type': endpoint['VpcEndpointType'],
                'state': endpoint['State'],
                'dns_entries': endpoint.get('DnsEntries', [])
            })

        return endpoints

    def test_endpoint_connectivity(self, endpoint_dns: str, port: int = 443) -> bool:
        """
        Testa connettività TCP a un endpoint.
        """
        try:
            sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
            sock.settimeout(5)
            result = sock.connect_ex((endpoint_dns, port))
            sock.close()
            return result == 0
        except Exception as e:
            print(f"Connection test failed: {str(e)}")
            return False

    def test_s3_via_endpoint(self, bucket_name: str):
        """
        Testa accesso S3 tramite Gateway Endpoint.
        """
        s3 = boto3.client('s3', region_name=self.region)

        try:
            # List objects (should route via Gateway Endpoint)
            response = s3.list_objects_v2(
                Bucket=bucket_name,
                MaxKeys=1
            )
            print(f"✅ S3 access via endpoint successful")
            return True
        except Exception as e:
            print(f"❌ S3 access failed: {str(e)}")
            return False

    def test_bedrock_via_endpoint(self, model_id: str):
        """
        Testa invocazione Bedrock tramite Interface Endpoint.
        """
        bedrock = boto3.client('bedrock-runtime', region_name=self.region)

        try:
            response = bedrock.invoke_model(
                modelId=model_id,
                body='{"prompt": "test", "max_tokens": 10}'
            )
            print(f"✅ Bedrock access via endpoint successful")
            return True
        except Exception as e:
            print(f"❌ Bedrock access failed: {str(e)}")
            return False

    def get_endpoint_cost_estimate(
        self,
        interface_endpoint_count: int,
        data_processed_gb: float
    ) -> Dict:
        """
        Stima costi VPC endpoints.

        Pricing (us-east-1):
        - Interface endpoint: $0.01/hour ($7.30/month)
        - Data processed: $0.01/GB
        - Gateway endpoint: FREE
        """
        hours_per_month = 730

        interface_cost = interface_endpoint_count * 0.01 * hours_per_month
        data_cost = data_processed_gb * 0.01

        return {
            'interface_endpoints_monthly': interface_cost,
            'data_transfer_cost': data_cost,
            'total_monthly': interface_cost + data_cost
        }

# Esempio d'uso
if __name__ == '__main__':
    tester = VPCEndpointTester(region='eu-south-1')

    vpc_id = 'vpc-12345678'

    # Lista endpoints
    endpoints = tester.list_endpoints(vpc_id)
    print(f"Found {len(endpoints)} VPC endpoints:")
    for ep in endpoints:
        print(f"  - {ep['service']} ({ep['type']}): {ep['state']}")

    # Test connectivity
    for ep in endpoints:
        if ep['type'] == 'Interface' and ep['dns_entries']:
            dns_name = ep['dns_entries'][0]['DnsName']
            connected = tester.test_endpoint_connectivity(dns_name)
            status = '✅' if connected else '❌'
            print(f"{status} {ep['service']}: {dns_name}")

    # Test S3 Gateway Endpoint
    tester.test_s3_via_endpoint('kb-documents-prod')

    # Test Bedrock Interface Endpoint
    tester.test_bedrock_via_endpoint('anthropic.claude-3-sonnet-20240229-v1:0')

    # Cost estimate
    cost = tester.get_endpoint_cost_estimate(
        interface_endpoint_count=7,  # Bedrock, Secrets, Logs, SSM, ECR API, ECR DKR, SageMaker
        data_processed_gb=1000  # 1TB/month
    )

    print(f"\nMonthly cost estimate:")
    print(f"  Interface endpoints: ${cost['interface_endpoints_monthly']:.2f}")
    print(f"  Data transfer: ${cost['data_transfer_cost']:.2f}")
    print(f"  Total: ${cost['total_monthly']:.2f}")
```

---

### 6. AWS WAF Rules (OWASP Top 10 Protection)

#### Setup Step-by-Step

**Obiettivo**: Proteggere le API pubbliche da attacchi comuni (SQL injection, XSS, etc.) usando AWS WAF.

**CloudFormation per WAF WebACL**:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: AWS WAF WebACL with OWASP Top 10 protection

Parameters:
  APIGatewayArn:
    Type: String
    Description: ARN of API Gateway to protect

Resources:
  # IP Rate Limiting (anti-DDoS)
  RateLimitRule:
    Type: AWS::WAFv2::WebACL
    Properties:
      Name: api-protection-waf
      Scope: REGIONAL
      Description: WAF rules for API Gateway protection
      DefaultAction:
        Allow: {}

      Rules:
        # Rule 1: Rate limiting (max 2000 requests per 5 minutes per IP)
        - Name: RateLimitRule
          Priority: 1
          Statement:
            RateBasedStatement:
              Limit: 2000
              AggregateKeyType: IP
          Action:
            Block:
              CustomResponse:
                ResponseCode: 429
                CustomResponseBodyKey: TooManyRequests
          VisibilityConfig:
            SampledRequestsEnabled: true
            CloudWatchMetricsEnabled: true
            MetricName: RateLimitRule

        # Rule 2: AWS Managed Rules - Core Rule Set (OWASP Top 10)
        - Name: AWSManagedRulesCommonRuleSet
          Priority: 2
          OverrideAction:
            None: {}
          Statement:
            ManagedRuleGroupStatement:
              VendorName: AWS
              Name: AWSManagedRulesCommonRuleSet
              ExcludedRules:
                - Name: SizeRestrictions_BODY  # Allow larger payloads
                - Name: GenericRFI_BODY       # Too many false positives
          VisibilityConfig:
            SampledRequestsEnabled: true
            CloudWatchMetricsEnabled: true
            MetricName: AWSManagedRulesCommonMetric

        # Rule 3: AWS Managed Rules - Known Bad Inputs
        - Name: AWSManagedRulesKnownBadInputsRuleSet
          Priority: 3
          OverrideAction:
            None: {}
          Statement:
            ManagedRuleGroupStatement:
              VendorName: AWS
              Name: AWSManagedRulesKnownBadInputsRuleSet
          VisibilityConfig:
            SampledRequestsEnabled: true
            CloudWatchMetricsEnabled: true
            MetricName: KnownBadInputsMetric

        # Rule 4: SQL Injection protection
        - Name: SQLInjectionProtection
          Priority: 4
          Statement:
            SqliMatchStatement:
              FieldToMatch:
                Body:
                  OversizeHandling: CONTINUE
              TextTransformations:
                - Priority: 0
                  Type: URL_DECODE
                - Priority: 1
                  Type: HTML_ENTITY_DECODE
          Action:
            Block:
              CustomResponse:
                ResponseCode: 403
                CustomResponseBodyKey: ForbiddenRequest
          VisibilityConfig:
            SampledRequestsEnabled: true
            CloudWatchMetricsEnabled: true
            MetricName: SQLInjectionMetric

        # Rule 5: XSS protection
        - Name: XSSProtection
          Priority: 5
          Statement:
            XssMatchStatement:
              FieldToMatch:
                Body:
                  OversizeHandling: CONTINUE
              TextTransformations:
                - Priority: 0
                  Type: URL_DECODE
                - Priority: 1
                  Type: HTML_ENTITY_DECODE
          Action:
            Block: {}
          VisibilityConfig:
            SampledRequestsEnabled: true
            CloudWatchMetricsEnabled: true
            MetricName: XSSMetric

        # Rule 6: Geo-blocking (block specific countries)
        - Name: GeoBlockRule
          Priority: 6
          Statement:
            GeoMatchStatement:
              CountryCodes:
                - CN  # China
                - RU  # Russia
                - KP  # North Korea
          Action:
            Block: {}
          VisibilityConfig:
            SampledRequestsEnabled: true
            CloudWatchMetricsEnabled: true
            MetricName: GeoBlockMetric

        # Rule 7: Block requests without User-Agent
        - Name: BlockMissingUserAgent
          Priority: 7
          Statement:
            NotStatement:
              Statement:
                ByteMatchStatement:
                  FieldToMatch:
                    SingleHeader:
                      Name: user-agent
                  PositionalConstraint: CONTAINS
                  SearchString: Mozilla
                  TextTransformations:
                    - Priority: 0
                      Type: NONE
          Action:
            Block: {}
          VisibilityConfig:
            SampledRequestsEnabled: true
            CloudWatchMetricsEnabled: true
            MetricName: MissingUserAgentMetric

        # Rule 8: Custom rule - Block suspicious patterns
        - Name: BlockSuspiciousPatterns
          Priority: 8
          Statement:
            OrStatement:
              Statements:
                # Block common attack tools
                - ByteMatchStatement:
                    FieldToMatch:
                      SingleHeader:
                        Name: user-agent
                    PositionalConstraint: CONTAINS
                    SearchString: sqlmap
                    TextTransformations:
                      - Priority: 0
                        Type: LOWERCASE

                - ByteMatchStatement:
                    FieldToMatch:
                      SingleHeader:
                        Name: user-agent
                    PositionalConstraint: CONTAINS
                    SearchString: nikto
                    TextTransformations:
                      - Priority: 0
                        Type: LOWERCASE

                # Block requests with suspicious URI
                - ByteMatchStatement:
                    FieldToMatch:
                      UriPath: {}
                    PositionalConstraint: CONTAINS
                    SearchString: ..
                    TextTransformations:
                      - Priority: 0
                        Type: URL_DECODE
          Action:
            Block: {}
          VisibilityConfig:
            SampledRequestsEnabled: true
            CloudWatchMetricsEnabled: true
            MetricName: SuspiciousPatternsMetric

      CustomResponseBodies:
        TooManyRequests:
          ContentType: APPLICATION_JSON
          Content: '{"error": "Rate limit exceeded. Please try again later."}'

        ForbiddenRequest:
          ContentType: APPLICATION_JSON
          Content: '{"error": "Request blocked by security policy."}'

      VisibilityConfig:
        SampledRequestsEnabled: true
        CloudWatchMetricsEnabled: true
        MetricName: APIProtectionWAF

      Tags:
        - Key: Environment
          Value: production
        - Key: Service
          Value: api-gateway

  # Associate WebACL with API Gateway
  WAFAssociation:
    Type: AWS::WAFv2::WebACLAssociation
    Properties:
      ResourceArn: !Ref APIGatewayArn
      WebACLArn: !GetAtt RateLimitRule.Arn

  # CloudWatch Alarm for blocked requests
  BlockedRequestsAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: waf-high-blocked-requests
      AlarmDescription: Alert when WAF blocks many requests
      MetricName: BlockedRequests
      Namespace: AWS/WAFV2
      Statistic: Sum
      Period: 300
      EvaluationPeriods: 2
      Threshold: 100
      ComparisonOperator: GreaterThanThreshold
      TreatMissingData: notBreaching
      Dimensions:
        - Name: Rule
          Value: ALL
        - Name: WebACL
          Value: api-protection-waf
      AlarmActions:
        - !ImportValue SecurityAlertsTopicArn

Outputs:
  WebACLId:
    Description: WAF WebACL ID
    Value: !GetAtt RateLimitRule.Id
    Export:
      Name: WAFWebACLId

  WebACLArn:
    Description: WAF WebACL ARN
    Value: !GetAtt RateLimitRule.Arn
    Export:
      Name: WAFWebACLArn
```

**Codice Python per gestire WAF**:

```python
import boto3
import json
from typing import Dict, List

class WAFManager:
    """
    Gestisce AWS WAF rules e analizza log.
    """

    def __init__(self, region: str = 'eu-south-1'):
        self.wafv2 = boto3.client('wafv2', region_name=region)
        self.cloudwatch = boto3.client('cloudwatch', region_name=region)

    def get_blocked_requests_count(
        self,
        web_acl_name: str,
        hours: int = 24
    ) -> int:
        """
        Conta le richieste bloccate nelle ultime N ore.
        """
        from datetime import datetime, timedelta

        end_time = datetime.utcnow()
        start_time = end_time - timedelta(hours=hours)

        response = self.cloudwatch.get_metric_statistics(
            Namespace='AWS/WAFV2',
            MetricName='BlockedRequests',
            Dimensions=[
                {'Name': 'WebACL', 'Value': web_acl_name},
                {'Name': 'Rule', 'Value': 'ALL'}
            ],
            StartTime=start_time,
            EndTime=end_time,
            Period=3600,
            Statistics=['Sum']
        )

        total = sum(point['Sum'] for point in response['Datapoints'])
        return int(total)

    def get_top_blocked_ips(
        self,
        web_acl_id: str,
        scope: str = 'REGIONAL',
        limit: int = 10
    ) -> List[Dict]:
        """
        Ottiene i top IP bloccati da WAF.

        Richiede WAF logging abilitato su Kinesis Firehose o S3.
        """
        # This would parse WAF logs from S3/Kinesis
        # Simplified example:
        from collections import Counter

        # Mock data - in production, parse from S3 logs
        blocked_ips = [
            '203.0.113.1',
            '203.0.113.2',
            '203.0.113.1',
            '198.51.100.5',
            '203.0.113.1'
        ]

        ip_counts = Counter(blocked_ips)

        return [
            {'ip': ip, 'count': count}
            for ip, count in ip_counts.most_common(limit)
        ]

    def create_ip_set_block_list(
        self,
        name: str,
        ip_addresses: List[str],
        scope: str = 'REGIONAL'
    ) -> str:
        """
        Crea IP Set per bloccare specifici IP.

        Args:
            name: Nome dell'IP set
            ip_addresses: Lista di IP/CIDR da bloccare
            scope: REGIONAL o CLOUDFRONT

        Returns:
            IP Set ID
        """
        response = self.wafv2.create_ip_set(
            Name=name,
            Scope=scope,
            Description=f'Block list created at {datetime.utcnow().isoformat()}',
            IPAddressVersion='IPV4',
            Addresses=ip_addresses,
            Tags=[
                {'Key': 'ManagedBy', 'Value': 'automation'}
            ]
        )

        return response['Summary']['Id']

    def update_ip_set(
        self,
        ip_set_id: str,
        ip_addresses: List[str],
        scope: str = 'REGIONAL'
    ):
        """
        Aggiorna IP Set con nuovi IP da bloccare.
        """
        # Get current IP set details
        ip_set = self.wafv2.get_ip_set(
            Id=ip_set_id,
            Scope=scope
        )

        # Update with new addresses
        self.wafv2.update_ip_set(
            Id=ip_set_id,
            Scope=scope,
            Addresses=ip_addresses,
            LockToken=ip_set['LockToken']
        )

    def analyze_sampled_requests(
        self,
        web_acl_id: str,
        rule_name: str,
        scope: str = 'REGIONAL',
        max_items: int = 100
    ) -> List[Dict]:
        """
        Analizza richieste campionate per un rule specifico.
        """
        from datetime import datetime, timedelta

        end_time = datetime.utcnow()
        start_time = end_time - timedelta(hours=1)

        response = self.wafv2.get_sampled_requests(
            WebAclId=web_acl_id,
            RuleMetricName=rule_name,
            Scope=scope,
            TimeWindow={
                'StartTime': start_time,
                'EndTime': end_time
            },
            MaxItems=max_items
        )

        sampled = []
        for request in response['SampledRequests']:
            sampled.append({
                'client_ip': request['Request']['ClientIP'],
                'country': request['Request'].get('Country', 'Unknown'),
                'uri': request['Request']['URI'],
                'method': request['Request']['Method'],
                'action': request['Action'],
                'timestamp': request['Timestamp']
            })

        return sampled

# Esempio d'uso
if __name__ == '__main__':
    manager = WAFManager(region='eu-south-1')

    web_acl_name = 'api-protection-waf'
    web_acl_id = 'web-acl-id-here'

    # Conta richieste bloccate
    blocked_count = manager.get_blocked_requests_count(web_acl_name, hours=24)
    print(f"Blocked requests (24h): {blocked_count}")

    # Top IP bloccati
    top_ips = manager.get_top_blocked_ips(web_acl_id, limit=10)
    print(f"\nTop 10 blocked IPs:")
    for item in top_ips:
        print(f"  {item['ip']}: {item['count']} requests")

    # Crea IP Set per bloccare IP malevoli
    malicious_ips = ['203.0.113.1/32', '198.51.100.5/32']
    ip_set_id = manager.create_ip_set_block_list(
        name='malicious-ips-blocklist',
        ip_addresses=malicious_ips
    )
    print(f"\nCreated IP Set: {ip_set_id}")

    # Analizza richieste campionate
    samples = manager.analyze_sampled_requests(
        web_acl_id=web_acl_id,
        rule_name='SQLInjectionProtection',
        max_items=50
    )

    print(f"\nSQL Injection attempts:")
    for sample in samples[:5]:
        print(f"  {sample['client_ip']} ({sample['country']}): {sample['uri']}")
```

---

### 7. GuardDuty Automated Response

#### Setup Step-by-Step

**Obiettivo**: Configurare risposta automatica a GuardDuty findings critici.

**Lambda Function per automated response**:

```python
import boto3
import json
import logging
from typing import Dict, Any

logger = logging.getLogger()
logger.setLevel(logging.INFO)

# AWS clients
ec2 = boto3.client('ec2')
guardduty = boto3.client('guardduty')
sns = boto3.client('sns')
iam = boto3.client('iam')
secretsmanager = boto3.client('secretsmanager')

SECURITY_TEAM_TOPIC_ARN = 'arn:aws:sns:eu-south-1:123456789012:security-alerts'

def lambda_handler(event, context):
    """
    EventBridge trigger per GuardDuty findings.

    Event structure:
    {
        "detail": {
            "severity": 7.5,
            "type": "UnauthorizedAccess:EC2/SSHBruteForce",
            "resource": {...},
            "service": {...}
        }
    }
    """
    finding = event['detail']

    severity = finding['severity']
    finding_type = finding['type']
    resource = finding.get('resource', {})

    logger.info(f"Processing GuardDuty finding: {finding_type} (severity: {severity})")

    # Determine response action based on severity
    if severity >= 7.0:  # High/Critical
        handle_critical_finding(finding)
    elif severity >= 4.0:  # Medium
        handle_medium_finding(finding)
    else:  # Low
        handle_low_finding(finding)

    return {'statusCode': 200, 'body': 'Finding processed'}

def handle_critical_finding(finding: Dict[str, Any]):
    """
    Risposta automatica per finding critici.

    Actions:
    1. Isolate compromised resource
    2. Create forensic snapshot
    3. Rotate all credentials
    4. Page security team
    5. Create incident ticket
    """
    finding_type = finding['type']
    resource = finding.get('resource', {})

    logger.error(f"CRITICAL FINDING: {finding_type}")

    # 1. Isolate resource
    if 'instanceDetails' in resource:
        instance_id = resource['instanceDetails']['instanceId']
        isolate_ec2_instance(instance_id)
        create_forensic_snapshot(instance_id)

    # 2. Rotate credentials
    if 'accessKeyDetails' in finding.get('service', {}).get('action', {}):
        access_key_id = finding['service']['action']['accessKeyDetails']['accessKeyId']
        user_name = finding['service']['action']['accessKeyDetails']['userName']
        deactivate_access_key(user_name, access_key_id)

    # 3. Rotate all secrets (paranoid mode)
    rotate_all_secrets()

    # 4. Alert security team
    page_security_team(finding, priority='CRITICAL')

    # 5. Create incident ticket (integrate with PagerDuty/Jira)
    create_incident_ticket(finding)

def handle_medium_finding(finding: Dict[str, Any]):
    """
    Risposta per finding medium severity.

    Actions:
    1. Log to SIEM
    2. Send Slack alert
    3. Create CloudWatch alarm
    """
    logger.warning(f"MEDIUM FINDING: {finding['type']}")

    # Send Slack notification
    send_slack_alert(finding)

    # Log to CloudWatch for analysis
    log_to_cloudwatch(finding)

def handle_low_finding(finding: Dict[str, Any]):
    """
    Risposta per finding low severity.

    Actions:
    1. Log only
    """
    logger.info(f"LOW FINDING: {finding['type']}")
    log_to_cloudwatch(finding)

def isolate_ec2_instance(instance_id: str):
    """
    Isola istanza EC2 modificando il security group.

    Crea un SG "quarantine" che blocca tutto il traffico.
    """
    logger.info(f"Isolating EC2 instance: {instance_id}")

    # Get instance VPC
    response = ec2.describe_instances(InstanceIds=[instance_id])
    vpc_id = response['Reservations'][0]['Instances'][0]['VpcId']

    # Create or get quarantine security group
    try:
        sg_response = ec2.describe_security_groups(
            Filters=[
                {'Name': 'group-name', 'Values': ['quarantine-sg']},
                {'Name': 'vpc-id', 'Values': [vpc_id]}
            ]
        )

        if sg_response['SecurityGroups']:
            quarantine_sg_id = sg_response['SecurityGroups'][0]['GroupId']
        else:
            # Create quarantine SG
            create_response = ec2.create_security_group(
                GroupName='quarantine-sg',
                Description='Quarantine security group - blocks all traffic',
                VpcId=vpc_id
            )
            quarantine_sg_id = create_response['GroupId']

            # Remove all inbound/outbound rules (implicit deny)
            # No rules = no traffic

        # Modify instance security groups
        ec2.modify_instance_attribute(
            InstanceId=instance_id,
            Groups=[quarantine_sg_id]
        )

        # Tag instance
        ec2.create_tags(
            Resources=[instance_id],
            Tags=[
                {'Key': 'SecurityStatus', 'Value': 'Quarantined'},
                {'Key': 'QuarantineDate', 'Value': str(datetime.utcnow())}
            ]
        )

        logger.info(f"Instance {instance_id} successfully quarantined")

    except Exception as e:
        logger.error(f"Failed to isolate instance: {str(e)}")
        raise

def create_forensic_snapshot(instance_id: str):
    """
    Crea snapshot dei volumi per forensic analysis.
    """
    logger.info(f"Creating forensic snapshot for instance: {instance_id}")

    try:
        # Get volumes attached to instance
        response = ec2.describe_instances(InstanceIds=[instance_id])
        instance = response['Reservations'][0]['Instances'][0]

        for block_device in instance.get('BlockDeviceMappings', []):
            volume_id = block_device['Ebs']['VolumeId']

            # Create snapshot
            snapshot = ec2.create_snapshot(
                VolumeId=volume_id,
                Description=f'Forensic snapshot for {instance_id} - GuardDuty incident',
                TagSpecifications=[{
                    'ResourceType': 'snapshot',
                    'Tags': [
                        {'Key': 'Purpose', 'Value': 'Forensics'},
                        {'Key': 'InstanceId', 'Value': instance_id},
                        {'Key': 'CreatedBy', 'Value': 'GuardDutyResponse'}
                    ]
                }]
            )

            logger.info(f"Created snapshot {snapshot['SnapshotId']} for volume {volume_id}")

    except Exception as e:
        logger.error(f"Failed to create forensic snapshot: {str(e)}")

def deactivate_access_key(user_name: str, access_key_id: str):
    """
    Deattiva access key compromessa.
    """
    logger.warning(f"Deactivating access key {access_key_id} for user {user_name}")

    try:
        iam.update_access_key(
            UserName=user_name,
            AccessKeyId=access_key_id,
            Status='Inactive'
        )

        logger.info(f"Access key {access_key_id} deactivated")

    except Exception as e:
        logger.error(f"Failed to deactivate access key: {str(e)}")

def rotate_all_secrets():
    """
    Trigger rotation per tutti i secrets critici.
    """
    logger.info("Triggering rotation for all secrets")

    try:
        # List all secrets with auto-rotation enabled
        paginator = secretsmanager.get_paginator('list_secrets')

        for page in paginator.paginate():
            for secret in page['SecretList']:
                if secret.get('RotationEnabled'):
                    secret_id = secret['ARN']

                    try:
                        secretsmanager.rotate_secret(
                            SecretId=secret_id,
                            RotateImmediately=True
                        )
                        logger.info(f"Triggered rotation for {secret['Name']}")
                    except Exception as e:
                        logger.warning(f"Failed to rotate {secret['Name']}: {str(e)}")

    except Exception as e:
        logger.error(f"Failed to rotate secrets: {str(e)}")

def page_security_team(finding: Dict, priority: str = 'HIGH'):
    """
    Invia alert al security team via SNS.
    """
    subject = f"[{priority}] GuardDuty Finding: {finding['type']}"

    message = f"""
SECURITY ALERT

Severity: {finding['severity']}
Type: {finding['type']}
Description: {finding.get('description', 'N/A')}

Resource:
{json.dumps(finding.get('resource', {}), indent=2)}

Action Taken:
- Resource isolated
- Forensic snapshot created
- Credentials rotated

Please investigate immediately.

GuardDuty Console: https://console.aws.amazon.com/guardduty/
    """

    sns.publish(
        TopicArn=SECURITY_TEAM_TOPIC_ARN,
        Subject=subject,
        Message=message
    )

    logger.info("Security team alerted")

def create_incident_ticket(finding: Dict):
    """
    Crea incident ticket in sistema di ticketing (Jira/ServiceNow/PagerDuty).
    """
    # Integrate with your ticketing system
    logger.info("Creating incident ticket (not implemented)")

def send_slack_alert(finding: Dict):
    """
    Invia alert su Slack channel.
    """
    # Integrate with Slack webhook
    logger.info("Sending Slack alert (not implemented)")

def log_to_cloudwatch(finding: Dict):
    """
    Log strutturato su CloudWatch per analisi.
    """
    logger.info(json.dumps({
        'event_type': 'guardduty_finding',
        'severity': finding['severity'],
        'finding_type': finding['type'],
        'account_id': finding['accountId'],
        'region': finding['region'],
        'resource': finding.get('resource', {}),
        'timestamp': finding['updatedAt']
    }))
```

**EventBridge Rule CloudFormation**:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: EventBridge rule for GuardDuty automated response

Resources:
  GuardDutyResponseLambdaRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: GuardDutyResponsePolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - ec2:DescribeInstances
                  - ec2:DescribeSecurityGroups
                  - ec2:CreateSecurityGroup
                  - ec2:ModifyInstanceAttribute
                  - ec2:CreateTags
                  - ec2:CreateSnapshot
                  - ec2:DescribeSnapshots
                Resource: '*'

              - Effect: Allow
                Action:
                  - iam:UpdateAccessKey
                  - iam:ListAccessKeys
                Resource: '*'

              - Effect: Allow
                Action:
                  - secretsmanager:RotateSecret
                  - secretsmanager:ListSecrets
                Resource: '*'

              - Effect: Allow
                Action:
                  - sns:Publish
                Resource: !Ref SecurityAlertsTo
pic

  SecurityAlertsTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: security-alerts
      DisplayName: Security Team Alerts
      Subscription:
        - Protocol: email
          Endpoint: security@example.com
        - Protocol: sms
          Endpoint: +391234567890

  GuardDutyResponseFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: guardduty-automated-response
      Runtime: python3.11
      Handler: index.lambda_handler
      Role: !GetAtt GuardDutyResponseLambdaRole.Arn
      Timeout: 300
      Code:
        ZipFile: |
          # Paste the Lambda code here
      Environment:
        Variables:
          SECURITY_TEAM_TOPIC_ARN: !Ref SecurityAlertsTopic

  GuardDutyEventRule:
    Type: AWS::Events::Rule
    Properties:
      Name: guardduty-findings-rule
      Description: Trigger automated response for GuardDuty findings
      EventPattern:
        source:
          - aws.guardduty
        detail-type:
          - GuardDuty Finding
        detail:
          severity:
            - numeric:
                - ">="
                - 4.0
      State: ENABLED
      Targets:
        - Arn: !GetAtt GuardDutyResponseFunction.Arn
          Id: GuardDutyResponseTarget

  GuardDutyLambdaPermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref GuardDutyResponseFunction
      Action: lambda:InvokeFunction
      Principal: events.amazonaws.com
      SourceArn: !GetAtt GuardDutyEventRule.Arn

Outputs:
  LambdaArn:
    Description: GuardDuty Response Lambda ARN
    Value: !GetAtt GuardDutyResponseFunction.Arn
```

---

### 8. Encryption Examples

#### S3 Server-Side Encryption with KMS

```python
import boto3
import json

class S3EncryptionManager:
    """
    Gestisce encryption per S3 con KMS.
    """

    def __init__(self, kms_key_id: str):
        self.s3 = boto3.client('s3')
        self.kms_key_id = kms_key_id

    def upload_encrypted_object(
        self,
        bucket: str,
        key: str,
        data: bytes,
        encryption_context: dict = None
    ):
        """
        Upload oggetto con SSE-KMS encryption.
        """
        put_params = {
            'Bucket': bucket,
            'Key': key,
            'Body': data,
            'ServerSideEncryption': 'aws:kms',
            'SSEKMSKeyId': self.kms_key_id
        }

        if encryption_context:
            put_params['SSEKMSEncryptionContext'] = json.dumps(encryption_context)

        self.s3.put_object(**put_params)

    def enforce_encryption_policy(self, bucket: str):
        """
        Applica bucket policy che richiede encryption.
        """
        policy = {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Sid": "DenyUnencryptedObjectUploads",
                    "Effect": "Deny",
                    "Principal": "*",
                    "Action": "s3:PutObject",
                    "Resource": f"arn:aws:s3:::{bucket}/*",
                    "Condition": {
                        "StringNotEquals": {
                            "s3:x-amz-server-side-encryption": "aws:kms"
                        }
                    }
                }
            ]
        }

        self.s3.put_bucket_policy(
            Bucket=bucket,
            Policy=json.dumps(policy)
        )

# Esempio d'uso
manager = S3EncryptionManager(kms_key_id='alias/app-data-key')
manager.upload_encrypted_object(
    bucket='kb-documents-prod',
    key='manual-001.pdf',
    data=b'...',
    encryption_context={'document_type': 'manual', 'classification': 'internal'}
)
```

#### DynamoDB Encryption at Rest

```python
import boto3

def create_encrypted_table():
    """
    Crea DynamoDB table con encryption.
    """
    dynamodb = boto3.client('dynamodb')

    response = dynamodb.create_table(
        TableName='tickets',
        KeySchema=[
            {'AttributeName': 'ticket_id', 'KeyType': 'HASH'}
        ],
        AttributeDefinitions=[
            {'AttributeName': 'ticket_id', 'AttributeType': 'S'}
        ],
        BillingMode='PAY_PER_REQUEST',
        SSESpecification={
            'Enabled': True,
            'SSEType': 'KMS',
            'KMSMasterKeyId': 'alias/app-data-key'
        },
        Tags=[
            {'Key': 'Environment', 'Value': 'production'}
        ]
    )

    return response
```

#### RDS Encryption

```yaml
# CloudFormation for encrypted RDS
RDSInstance:
  Type: AWS::RDS::DBInstance
  Properties:
    DBInstanceIdentifier: app-db-prod
    Engine: postgres
    EngineVersion: '15.3'
    DBInstanceClass: db.t3.medium
    AllocatedStorage: 100
    StorageType: gp3
    StorageEncrypted: true
    KmsKeyId: !Ref AppDataKey
    MasterUsername: !Sub '{{resolve:secretsmanager:${DBCredentialsSecret}:SecretString:username}}'
    MasterUserPassword: !Sub '{{resolve:secretsmanager:${DBCredentialsSecret}:SecretString:password}}'
    VPCSecurityGroups:
      - !Ref DatabaseSecurityGroup
    DBSubnetGroupName: !Ref DBSubnetGroup
    BackupRetentionPeriod: 30
    PreferredBackupWindow: '03:00-04:00'
    EnableCloudwatchLogsExports:
      - postgresql
    DeletionProtection: true
```

---

## Best Practices

### Do's ✅

1. **Always use least privilege**
   - Grant minimum permissions necessary
   - Use specific ARNs instead of wildcards
   - Add condition keys for extra restrictions

2. **Enable encryption everywhere**
   - S3: SSE-KMS
   - DynamoDB: KMS encryption
   - RDS: Storage encryption + SSL connections
   - Secrets Manager: KMS encryption
   - EBS volumes: Auto-encryption for Lambda

3. **Rotate credentials regularly**
   - Secrets Manager: Auto-rotation every 30 days
   - Access keys: Rotate every 90 days
   - KMS keys: Enable automatic annual rotation

4. **Use VPC Endpoints**
   - Avoid internet traffic for AWS services
   - Reduce data transfer costs
   - Improve security posture

5. **Enable comprehensive logging**
   - CloudTrail: All API calls
   - VPC Flow Logs: Network traffic
   - GuardDuty: Threat detection
   - AWS Config: Compliance monitoring

6. **Implement defense in depth**
   - Multiple layers of security controls
   - Fail securely if one layer is compromised
   - Validate at every layer (API Gateway + Lambda + Database)

7. **Use Service Control Policies**
   - Enforce encryption at organization level
   - Restrict approved regions
   - Require MFA for sensitive operations

8. **Automate security response**
   - EventBridge rules for GuardDuty findings
   - Lambda for automated remediation
   - SNS for security team alerts

9. **Tag everything**
   - Resource ownership
   - Cost allocation
   - Security classification
   - Backup policies

10. **Test disaster recovery**
    - Regular DR drills
    - Backup restoration tests
    - Incident response simulations

### Don'ts ❌

1. **Never use wildcards in production policies**
   - ❌ `"Action": "*"`
   - ❌ `"Resource": "*"`
   - ✅ Use specific actions and ARNs

2. **Don't store secrets in code**
   - ❌ Hardcoded passwords in Lambda
   - ❌ API keys in environment variables
   - ✅ Use Secrets Manager or Parameter Store

3. **Don't skip input validation**
   - ❌ Trust user input
   - ✅ Validate at API Gateway level
   - ✅ Re-validate in Lambda
   - ✅ Use parameterized queries

4. **Don't use long-lived credentials**
   - ❌ IAM user access keys
   - ✅ IAM roles with temporary credentials
   - ✅ Cognito tokens with short expiry

5. **Don't ignore security findings**
   - ❌ Suppress GuardDuty alerts
   - ❌ Ignore Config non-compliant resources
   - ✅ Investigate and remediate

6. **Don't use default security groups**
   - ❌ Default SG allows all outbound
   - ✅ Create custom SGs with explicit rules

7. **Don't forget about data in transit**
   - ❌ Unencrypted HTTP
   - ✅ HTTPS/TLS everywhere
   - ✅ Certificate pinning for critical connections

8. **Don't mix environments**
   - ❌ Prod and dev in same VPC
   - ✅ Separate AWS accounts per environment
   - ✅ Use AWS Organizations

---

## Troubleshooting

### Problem 1: AccessDenied errors

**Symptoms**:
```
botocore.exceptions.ClientError: An error occurred (AccessDenied) when calling the GetObject operation
```

**Root causes**:
1. IAM policy missing required permission
2. S3 bucket policy denying access
3. KMS key policy not allowing decrypt
4. Wrong IAM role assumed

**Solution**:
```bash
# 1. Check IAM policy simulator
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:role/LambdaRole \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::bucket/key

# 2. Check S3 bucket policy
aws s3api get-bucket-policy --bucket bucket-name

# 3. Check KMS key policy
aws kms get-key-policy --key-id alias/app-data-key --policy-name default

# 4. Check assumed role
aws sts get-caller-identity
```

### Problem 2: KMS Key Usage Limits Exceeded

**Symptoms**:
```
ThrottlingException: Rate exceeded for key alias/app-data-key
```

**Root cause**: KMS has quota limits (5500 req/sec per key for symmetric encryption)

**Solutions**:
1. **Use Data Key Caching**:
```python
from aws_encryption_sdk import EncryptionSDKClient, CachingCryptoMaterialsManager
from aws_encryption_sdk.key_providers.kms import KMSMasterKeyProvider

kms_key_provider = KMSMasterKeyProvider(key_ids=['alias/app-data-key'])
cache = LocalCryptoMaterialsCache(capacity=100)
crypto_cache = CachingCryptoMaterialsManager(
    master_key_provider=kms_key_provider,
    cache=cache,
    max_age=300  # 5 minutes
)
```

2. **Use envelope encryption**: Generate data key once, reuse for multiple operations

3. **Request quota increase**: AWS Support ticket

### Problem 3: Secrets Manager rotation fails

**Symptoms**:
```
ResourceNotFoundException: Secrets Manager can't find the specified secret version with VersionStage: AWSPENDING
```

**Root cause**: Rotation Lambda failed during one of the steps

**Solution**:
```bash
# 1. Check Lambda logs
aws logs tail /aws/lambda/rds-secrets-rotation --follow

# 2. Check secret versions
aws secretsmanager describe-secret --secret-id prod/rds/credentials

# 3. Manually trigger rotation
aws secretsmanager rotate-secret \
  --secret-id prod/rds/credentials \
  --rotation-lambda-arn arn:aws:lambda:eu-south-1:123456789012:function:rotation

# 4. If stuck, cancel rotation
aws secretsmanager cancel-rotate-secret --secret-id prod/rds/credentials
```

### Problem 4: VPC Endpoint DNS resolution fails

**Symptoms**:
Lambda can't connect to S3/DynamoDB via VPC endpoint

**Root cause**: DNS resolution not enabled

**Solution**:
```bash
# 1. Check endpoint DNS settings
aws ec2 describe-vpc-endpoints --vpc-endpoint-ids vpce-12345

# 2. Enable private DNS (for Interface Endpoints)
aws ec2 modify-vpc-endpoint \
  --vpc-endpoint-id vpce-12345 \
  --private-dns-enabled

# 3. Check VPC DNS settings
aws ec2 describe-vpc-attribute \
  --vpc-id vpc-12345 \
  --attribute enableDnsHostnames

# 4. Enable if disabled
aws ec2 modify-vpc-attribute \
  --vpc-id vpc-12345 \
  --enable-dns-hostnames
```

### Problem 5: GuardDuty false positives

**Symptoms**:
GuardDuty generates many low-severity findings for legitimate traffic

**Solution**:
```python
# Create suppression rules
guardduty = boto3.client('guardduty')

response = guardduty.create_filter(
    DetectorId='detector-id',
    Name='suppress-known-scanner',
    Description='Suppress findings from internal security scanner',
    FindingCriteria={
        'Criterion': {
            'type': {
                'Eq': ['Recon:EC2/PortProbeUnprotectedPort']
            },
            'service.action.networkConnectionAction.remoteIpDetails.ipAddressV4': {
                'Eq': ['10.0.0.50']  # Internal scanner IP
            }
        }
    },
    Action='ARCHIVE'
)
```

---

## Esempi Reali dal Progetto

### Esempio 1: Lambda accessing encrypted S3 and DynamoDB

**Scenario**: Lambda function che legge documenti KB da S3 (encrypted with KMS) e scrive metadata su DynamoDB.

**IAM Role Policy**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::kb-documents-prod/*",
      "Condition": {
        "StringEquals": {
          "s3:ExistingObjectTag/classification": "public"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": ["dynamodb:PutItem"],
      "Resource": "arn:aws:dynamodb:eu-south-1:123456789012:table/kb-metadata"
    },
    {
      "Effect": "Allow",
      "Action": ["kms:Decrypt"],
      "Resource": "arn:aws:kms:eu-south-1:123456789012:key/app-data-key-id",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "s3.eu-south-1.amazonaws.com"
        }
      }
    }
  ]
}
```

Riferimento: `docs/04-data-flows/ticket-processing.md:125`

### Esempio 2: Cross-account KB sharing

**Scenario**: Account A (prod) condivide Knowledge Base S3 bucket con Account B (dev) per testing.

**Account A - S3 Bucket Policy**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::987654321098:role/DevTestRole"
      },
      "Action": ["s3:GetObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::kb-documents-prod",
        "arn:aws:s3:::kb-documents-prod/*"
      ],
      "Condition": {
        "StringEquals": {
          "s3:ExistingObjectTag/shareable": "true"
        }
      }
    }
  ]
}
```

**Account A - KMS Key Policy**:
```json
{
  "Sid": "Allow dev account to decrypt",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::987654321098:role/DevTestRole"
  },
  "Action": ["kms:Decrypt", "kms:DescribeKey"],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "kms:ViaService": "s3.eu-south-1.amazonaws.com"
    }
  }
}
```

Riferimento: `docs/02-architecture/security.md:340`

### Esempio 3: Secrets rotation for RDS

**Scenario**: Automatic rotation ogni 30 giorni della password RDS usata da Lambda.

**Setup completo** in sezione 4 sopra.

**Lambda Code snippet**:
```python
import boto3
import json

secretsmanager = boto3.client('secretsmanager')

def get_db_credentials():
    """
    Retrieve DB credentials from Secrets Manager.
    """
    response = secretsmanager.get_secret_value(
        SecretId='prod/rds/credentials'
    )

    secret = json.loads(response['SecretString'])

    return {
        'host': secret['host'],
        'port': secret['port'],
        'username': secret['username'],
        'password': secret['password'],
        'dbname': secret['dbname']
    }

# Usage
creds = get_db_credentials()
conn = psycopg2.connect(**creds)
```

Riferimento: `docs/03-aws-services/README.md:45`

---

## Riferimenti

### Documentazione Interna
- [Security & Compliance](../10-security-compliance.md) - Overview generale sicurezza
- [Architecture Security](../02-architecture/security.md) - Dettagli architettura
- [AWS Services README](../03-aws-services/README.md) - IAM roles e policies
- [Deployment](../02-architecture/deployment.md) - VPC e network security

### AWS Documentation
- [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [KMS Developer Guide](https://docs.aws.amazon.com/kms/latest/developerguide/)
- [Secrets Manager User Guide](https://docs.aws.amazon.com/secretsmanager/latest/userguide/)
- [VPC Endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html)
- [AWS WAF Developer Guide](https://docs.aws.amazon.com/waf/latest/developerguide/)
- [GuardDuty User Guide](https://docs.aws.amazon.com/guardduty/latest/ug/)

### Blog Posts e Risorse
- [AWS Security Blog](https://aws.amazon.com/blogs/security/)
- [IAM Policy Simulator](https://policysim.aws.amazon.com/)
- [AWS Security Best Practices Whitepaper](https://docs.aws.amazon.com/whitepapers/latest/aws-security-best-practices/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [CIS AWS Foundations Benchmark](https://www.cisecurity.org/benchmark/amazon_web_services)

### Tools
- [AWS CLI](https://aws.amazon.com/cli/)
- [IAM Access Analyzer](https://aws.amazon.com/iam/features/analyze-access/)
- [AWS Security Hub](https://aws.amazon.com/security-hub/)
- [CloudFormation Guard](https://github.com/aws-cloudformation/cloudformation-guard)
- [Prowler](https://github.com/prowler-cloud/prowler) - AWS security assessment tool

---

**Version**: 1.0
**Last Updated**: 2025-11-18
**Author**: Security Team
**Reviewed by**: Tech Lead

**Changelog**:
- 2025-11-18: Initial version with all 8 examples and comprehensive coverage
