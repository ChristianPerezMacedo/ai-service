# API Gateway + Lambda Production Patterns

## Overview

### Cos'è e perché è importante nel nostro contesto

L'integrazione **API Gateway + Lambda** rappresenta il cuore dell'architettura serverless del nostro AI Technical Support System. Questo pattern consente di esporre le funzionalità AI/ML attraverso API REST scalabili, senza gestire server o infrastruttura.

**Vantaggi principali**:
- **Zero Server Management**: Nessun provisioning, patching, o scaling manuale
- **Pay-per-Use**: Costi solo per richieste effettive (no idle capacity)
- **Auto-Scaling**: Da 0 a migliaia di richieste/sec automaticamente
- **High Availability**: Multi-AZ by design
- **Security Built-in**: WAF, throttling, authentication integrati

Nel nostro sistema, API Gateway + Lambda gestiscono:
- Creazione e tracking ticket (`POST /tickets`, `GET /tickets/{id}`)
- Streaming soluzioni AI in tempo reale (`GET /tickets/{id}/solution/stream`)
- Ricerca knowledge base (`POST /kb/search`)
- Feedback operatori (`POST /tickets/{id}/feedback`)

### Quando usarlo

**Usa API Gateway + Lambda quando**:
- ✅ Traffico variabile o imprevedibile
- ✅ Microservizi con responsabilità ben definite
- ✅ Necessità di scaling rapido (0 → 1000 req/sec)
- ✅ Budget limitato e pay-per-use preferibile
- ✅ Eventi asincroni (webhook, trigger)

**Evita quando**:
- ❌ Latenza critica < 10ms (cold start overhead)
- ❌ Processi long-running > 15 minuti (limite Lambda)
- ❌ Applicazioni monolitiche esistenti senza refactor
- ❌ Workload costanti e prevedibili (EC2 può essere più economico)

### Architettura High-Level

```
┌──────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                               │
│         Web UI │ Mobile App │ ITSM Integration                   │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │   CloudFront    │ (CDN, DDoS protection)
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   AWS WAF       │ (Security rules)
                    └────────┬────────┘
                             │
        ┌────────────────────▼─────────────────────┐
        │          API GATEWAY (REST API)          │
        │  • Authentication (Cognito Authorizer)   │
        │  • Rate Limiting (Throttling)            │
        │  • Request Validation (JSON Schema)      │
        │  • Response Transformation               │
        │  • CORS Configuration                    │
        └────┬─────────────────┬────────────────┬──┘
             │                 │                │
    ┌────────▼────────┐  ┌────▼─────┐  ┌──────▼──────┐
    │ Lambda Function │  │ Lambda   │  │ Lambda      │
    │ CreateTicket    │  │ GetStatus│  │ SearchKB    │
    └────┬────────────┘  └────┬─────┘  └──────┬──────┘
         │                    │                │
         │    ┌───────────────┴────────────────┘
         │    │
    ┌────▼────▼──────────────────────────────┐
    │        DATA & SERVICE LAYER             │
    │  DynamoDB │ OpenSearch │ Step Functions │
    └─────────────────────────────────────────┘
```

## Concetti Fondamentali

### 1. API Gateway Overview

API Gateway è un servizio managed che funge da "front door" per le API:

**Tipi di API**:
- **REST API**: Full-featured, supporto completo request/response transformation
- **HTTP API**: Semplificato, latenza ridotta, costo 70% inferiore
- **WebSocket API**: Connessioni bidirezionali persistenti

Nel nostro sistema usiamo **REST API** per:
- Maggiore controllo su validation e transformation
- Supporto Lambda Authorizer custom
- Request/response mapping avanzato
- Caching integrato

**Componenti chiave**:
```
API Gateway
├── Resources (/tickets, /kb/search)
├── Methods (GET, POST, PUT, DELETE)
├── Stages (dev, staging, prod)
├── Authorizers (Cognito, Lambda, IAM)
├── Models (JSON Schema)
├── Gateway Responses (errori custom)
└── Usage Plans (rate limiting per client)
```

### 2. Lambda Integration Types

API Gateway può integrare Lambda in 2 modi:

#### Lambda Proxy Integration (Raccomandato)

**Caratteristiche**:
- Request completa passata a Lambda (headers, body, query params, path params)
- Lambda restituisce formato specifico con statusCode, headers, body
- Nessuna transformation in API Gateway (logica in Lambda)

**Quando usare**:
- ✅ Massima flessibilità nel Lambda handler
- ✅ Accesso completo a headers e context
- ✅ Rapid development (meno configurazione API Gateway)

**Formato richiesta Lambda**:
```json
{
  "resource": "/tickets",
  "path": "/tickets",
  "httpMethod": "POST",
  "headers": {
    "Authorization": "Bearer eyJhbG...",
    "Content-Type": "application/json"
  },
  "queryStringParameters": {"limit": "10"},
  "pathParameters": null,
  "body": "{\"customer\":{\"id\":\"C123\"}}",
  "isBase64Encoded": false,
  "requestContext": {
    "requestId": "abc123",
    "authorizer": {"claims": {...}}
  }
}
```

**Formato risposta Lambda**:
```json
{
  "statusCode": 202,
  "headers": {
    "Content-Type": "application/json",
    "Location": "/v1/tickets/tkt_67890"
  },
  "body": "{\"ticket_id\":\"tkt_67890\",\"status\":\"PROCESSING\"}"
}
```

#### Lambda Custom Integration

**Caratteristiche**:
- Request/response transformation in API Gateway (VTL templates)
- Lambda riceve solo parametri necessari (mappati)
- Maggiore controllo su formato, ma più complessità

**Quando usare**:
- ✅ Backend legacy che richiede formato specifico
- ✅ Trasformazioni complesse senza modificare Lambda
- ✅ Integrazione con servizi AWS diretti (non solo Lambda)

### 3. Lambda Handler Pattern

**Thin Handler, Fat Service** pattern:

```python
# ❌ BAD: Logica nel handler
def lambda_handler(event, context):
    body = json.loads(event['body'])
    # 100 righe di business logic...
    return {'statusCode': 200, 'body': json.dumps(result)}

# ✅ GOOD: Handler sottile, service layer robusto
def lambda_handler(event, context):
    """Entry point - parsing e response formatting only"""
    try:
        request = parse_request(event)
        result = TicketService().create_ticket(request)
        return success_response(result, status_code=202)
    except ValidationError as e:
        return error_response(e, status_code=400)
    except Exception as e:
        logger.exception("Unexpected error")
        return error_response("Internal error", status_code=500)
```

### 4. Cold Start e Warm Start

**Cold Start**: Prima invocazione o dopo inattività
- Lambda provision container (~100-300ms)
- Load code e dependencies (~200-500ms)
- Initialize connections (~100-500ms)
- **Total cold start**: 400-1300ms (Python), 2-5s (Java)

**Warm Start**: Container riutilizzato
- Bypass provision e load
- **Latency**: 5-50ms tipicamente

**Ottimizzazioni cold start**:
1. **Provisioned Concurrency**: Container pre-warmed sempre disponibili
2. **Minimize package size**: < 50MB (use Layers per dependencies)
3. **Optimize initialization**: Lazy load libraries, connection pooling globale
4. **Choose runtime wisely**: Python/Node.js migliori di Java/C# per cold start

### 5. Concurrency Management

**Concurrency = numero di invocazioni simultanee**

**Tipi di concurrency**:
- **Unreserved**: Pool condiviso (default 1000 account-wide)
- **Reserved**: Garantita per funzione specifica (es. 100 sempre disponibili)
- **Provisioned**: Pre-initialized, zero cold start

**Scenari**:

```
Scenario 1: Traffic Spike
- Unreserved: 900 disponibili
- Request burst: 1500 req/sec
- Result: 900 eseguite, 600 throttled (429 error)

Scenario 2: Reserved Concurrency
- CreateTicket Lambda: 100 reserved
- Altre Lambda: 900 unreserved
- Request burst: 1500 req/sec su CreateTicket
- Result: 100 eseguite, 1400 throttled (ma altre Lambda non impattate)

Scenario 3: Provisioned Concurrency
- GetSolution Lambda: 50 provisioned
- Request: 45 req/sec
- Result: 0 cold starts, latenza costante < 100ms
```

### 6. VPC Networking

Lambda può accedere a risorse in VPC (RDS, OpenSearch, ElastiCache):

**Come funziona**:
1. Lambda crea **ENI (Elastic Network Interface)** in subnet VPC
2. ENI creato una volta, riutilizzato da tutti i container
3. Lambda può accedere a risorse private (no internet senza NAT)

**Impatto**:
- ❌ Cold start +10-30s per ENI creation (solo prima invocazione post-deploy)
- ✅ Dopo ENI ready, no overhead aggiuntivo
- ✅ Migliorato da AWS (2019+): ENI pooling riduce delay

**Best practices VPC**:
- Use **Hyperplane ENI** (enabled by default 2019+)
- Connection pooling nelle variabili globali
- Minimize subnets (1-2 per AZ sufficienti)
- Use VPC Endpoints per AWS services (no NAT Gateway)

## Implementazione Pratica

### Esempio 1: Request Validation con JSON Schema

**Obiettivo**: Validare payload prima di invocare Lambda, risparmiando costi e tempo.

**API Gateway Model** (JSON Schema):

```json
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "title": "CreateTicketRequest",
  "type": "object",
  "required": ["customer", "asset", "symptom_text"],
  "properties": {
    "customer": {
      "type": "object",
      "required": ["id", "name"],
      "properties": {
        "id": {"type": "string", "pattern": "^C[0-9]+$"},
        "name": {"type": "string", "minLength": 1, "maxLength": 100},
        "contact_email": {"type": "string", "format": "email"}
      }
    },
    "asset": {
      "type": "object",
      "required": ["product_type", "model", "serial"],
      "properties": {
        "product_type": {
          "type": "string",
          "enum": ["EV_CHARGER", "INVERTER", "BATTERY"]
        },
        "model": {"type": "string", "maxLength": 50},
        "serial": {"type": "string", "pattern": "^SN[A-Z0-9]+$"},
        "firmware_version": {"type": "string"}
      }
    },
    "symptom_text": {
      "type": "string",
      "minLength": 10,
      "maxLength": 5000
    },
    "error_code": {"type": "string", "pattern": "^E[0-9]{3}$"},
    "priority": {
      "type": "string",
      "enum": ["P1", "P2", "P3", "P4"],
      "default": "P3"
    },
    "lang": {
      "type": "string",
      "enum": ["it-IT", "en-US", "de-DE"],
      "default": "it-IT"
    },
    "policies": {
      "type": "object",
      "properties": {
        "safe_instructions": {"type": "boolean", "default": true},
        "grounded_only": {"type": "boolean", "default": true}
      }
    }
  }
}
```

**CloudFormation - API Gateway Method con Validation**:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Resources:
  # Model definition
  CreateTicketModel:
    Type: AWS::ApiGateway::Model
    Properties:
      RestApiId: !Ref ApiGateway
      Name: CreateTicketRequest
      ContentType: application/json
      Schema: !Sub |
        {
          "$schema": "http://json-schema.org/draft-04/schema#",
          "title": "CreateTicketRequest",
          "type": "object",
          "required": ["customer", "asset", "symptom_text"],
          "properties": {
            "customer": {
              "type": "object",
              "required": ["id", "name"],
              "properties": {
                "id": {"type": "string", "pattern": "^C[0-9]+$"},
                "name": {"type": "string", "minLength": 1, "maxLength": 100}
              }
            },
            "symptom_text": {"type": "string", "minLength": 10}
          }
        }

  # Request Validator
  RequestValidator:
    Type: AWS::ApiGateway::RequestValidator
    Properties:
      RestApiId: !Ref ApiGateway
      Name: ValidateBodyAndParams
      ValidateRequestBody: true
      ValidateRequestParameters: true

  # POST /tickets method
  CreateTicketMethod:
    Type: AWS::ApiGateway::Method
    Properties:
      RestApiId: !Ref ApiGateway
      ResourceId: !Ref TicketsResource
      HttpMethod: POST
      AuthorizationType: COGNITO_USER_POOLS
      AuthorizerId: !Ref CognitoAuthorizer
      RequestValidatorId: !Ref RequestValidator
      RequestModels:
        application/json: !Ref CreateTicketModel
      Integration:
        Type: AWS_PROXY
        IntegrationHttpMethod: POST
        Uri: !Sub arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${CreateTicketLambda.Arn}/invocations
      MethodResponses:
        - StatusCode: 202
          ResponseModels:
            application/json: Empty
        - StatusCode: 400
          ResponseModels:
            application/json: Error
```

**Response su validation failure**:

```json
{
  "message": "Invalid request body",
  "errors": [
    {
      "keyword": "pattern",
      "dataPath": "/customer/id",
      "message": "does not match pattern \"^C[0-9]+$\""
    },
    {
      "keyword": "minLength",
      "dataPath": "/symptom_text",
      "message": "should NOT be shorter than 10 characters"
    }
  ]
}
```

**Benefici**:
- ⚡ Validation in ~5ms (vs ~100ms Lambda invocation)
- 💰 No Lambda invocation cost per richieste invalide
- 🔒 Security: Blocca payload malformed prima del backend

---

### Esempio 2: Response Mapping - Transform Lambda Output

**Obiettivo**: Trasformare output Lambda per compatibilità client o nascondere dati interni.

**Scenario**: Lambda restituisce dati interni dettagliati, vogliamo esporre solo subset.

**Lambda output (interno)**:

```json
{
  "ticket_id": "tkt_67890",
  "status": "PROCESSING",
  "internal_tracking_id": "TRK-INTERNAL-123",
  "ml_model_version": "v2.3.1-beta",
  "processing_cost_usd": 0.042,
  "operator_assigned": null,
  "sla_deadline": "2025-11-18T12:00:00Z"
}
```

**API Gateway Response desiderato (pubblico)**:

```json
{
  "ticket_id": "tkt_67890",
  "status": "PROCESSING",
  "sla_deadline": "2025-11-18T12:00:00Z"
}
```

**VTL Template (API Gateway Integration Response)**:

```velocity
#set($inputRoot = $input.path('$'))
{
  "ticket_id": "$inputRoot.ticket_id",
  "status": "$inputRoot.status",
  "sla_deadline": "$inputRoot.sla_deadline"
}
```

**CloudFormation**:

```yaml
CreateTicketMethod:
  Type: AWS::ApiGateway::Method
  Properties:
    # ... (auth, etc.)
    Integration:
      Type: AWS  # Custom integration (not PROXY)
      IntegrationHttpMethod: POST
      Uri: !Sub arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${CreateTicketLambda.Arn}/invocations
      IntegrationResponses:
        - StatusCode: 202
          ResponseTemplates:
            application/json: |
              #set($inputRoot = $input.path('$'))
              {
                "ticket_id": "$inputRoot.ticket_id",
                "status": "$inputRoot.status",
                "sla_deadline": "$inputRoot.sla_deadline"
              }
    MethodResponses:
      - StatusCode: 202
```

**Nota**: Con **Lambda Proxy**, transformation va fatta in Lambda:

```python
def lambda_handler(event, context):
    # Business logic
    internal_data = process_ticket()

    # Filter sensitive data
    public_data = {
        'ticket_id': internal_data['ticket_id'],
        'status': internal_data['status'],
        'sla_deadline': internal_data['sla_deadline']
    }

    return {
        'statusCode': 202,
        'body': json.dumps(public_data)
    }
```

---

### Esempio 3: Lambda Authorizer - Custom Auth Logic

**Obiettivo**: Implementare autenticazione custom (API key, JWT custom, OAuth2 non-Cognito).

**Use case**: Sistema ITSM esterno usa API key + signature HMAC per autenticazione.

**Lambda Authorizer (Python)**:

```python
import os
import hmac
import hashlib
import json
from datetime import datetime, timedelta

def lambda_handler(event, context):
    """
    Token-based Lambda Authorizer
    Valida API key + HMAC signature
    """

    # Extract authorization header
    token = event.get('authorizationToken', '')
    method_arn = event['methodArn']

    # Parse token format: "Bearer <api_key>:<signature>:<timestamp>"
    try:
        _, auth_data = token.split(' ')
        api_key, signature, timestamp = auth_data.split(':')
    except ValueError:
        raise Exception('Unauthorized')  # 401

    # Validate timestamp (prevent replay attacks)
    try:
        request_time = datetime.fromisoformat(timestamp)
        if abs((datetime.utcnow() - request_time).total_seconds()) > 300:  # 5 min window
            raise Exception('Unauthorized')
    except ValueError:
        raise Exception('Unauthorized')

    # Lookup API key in DynamoDB
    secret = get_api_secret(api_key)
    if not secret:
        raise Exception('Unauthorized')

    # Verify HMAC signature
    expected_signature = compute_signature(api_key, timestamp, secret)
    if not hmac.compare_digest(signature, expected_signature):
        raise Exception('Unauthorized')

    # Authorization successful - generate IAM policy
    return generate_policy(
        principal_id=api_key,
        effect='Allow',
        resource=method_arn,
        context={
            'api_key': api_key,
            'customer_id': get_customer_id(api_key),
            'rate_limit_tier': 'standard'
        }
    )

def compute_signature(api_key: str, timestamp: str, secret: str) -> str:
    """Compute HMAC-SHA256 signature"""
    message = f"{api_key}:{timestamp}"
    return hmac.new(
        secret.encode('utf-8'),
        message.encode('utf-8'),
        hashlib.sha256
    ).hexdigest()

def get_api_secret(api_key: str) -> str:
    """Lookup API key secret from DynamoDB"""
    import boto3
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table(os.environ['API_KEYS_TABLE'])

    try:
        response = table.get_item(Key={'api_key': api_key})
        return response.get('Item', {}).get('secret')
    except Exception:
        return None

def get_customer_id(api_key: str) -> str:
    """Get customer ID associated with API key"""
    import boto3
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table(os.environ['API_KEYS_TABLE'])

    response = table.get_item(Key={'api_key': api_key})
    return response.get('Item', {}).get('customer_id', 'unknown')

def generate_policy(principal_id: str, effect: str, resource: str, context: dict = None):
    """Generate IAM policy for API Gateway"""
    policy = {
        'principalId': principal_id,
        'policyDocument': {
            'Version': '2012-10-17',
            'Statement': [
                {
                    'Action': 'execute-api:Invoke',
                    'Effect': effect,
                    'Resource': resource
                }
            ]
        }
    }

    # Context available to Lambda via event['requestContext']['authorizer']
    if context:
        policy['context'] = context

    return policy
```

**CloudFormation - Authorizer**:

```yaml
CustomAuthorizer:
  Type: AWS::ApiGateway::Authorizer
  Properties:
    Name: ApiKeyAuthorizer
    Type: TOKEN
    AuthorizerUri: !Sub arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${AuthorizerLambda.Arn}/invocations
    AuthorizerCredentials: !GetAtt ApiGatewayAuthorizerRole.Arn
    IdentitySource: method.request.header.Authorization
    AuthorizerResultTtlInSeconds: 300  # Cache policy 5 min

CreateTicketMethod:
  Type: AWS::ApiGateway::Method
  Properties:
    RestApiId: !Ref ApiGateway
    ResourceId: !Ref TicketsResource
    HttpMethod: POST
    AuthorizationType: CUSTOM
    AuthorizerId: !Ref CustomAuthorizer
    # ...
```

**Lambda handler - Access context**:

```python
def lambda_handler(event, context):
    """CreateTicket Lambda - access authorizer context"""

    # Extract authorizer context
    authorizer = event['requestContext']['authorizer']
    customer_id = authorizer['customer_id']
    rate_limit_tier = authorizer['rate_limit_tier']

    logger.info(f"Request from customer {customer_id}, tier {rate_limit_tier}")

    # Business logic with customer context
    # ...
```

**Client request**:

```bash
# Generate signature
API_KEY="ak_12345"
SECRET="sk_abcdef"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
MESSAGE="${API_KEY}:${TIMESTAMP}"
SIGNATURE=$(echo -n "$MESSAGE" | openssl dgst -sha256 -hmac "$SECRET" | cut -d' ' -f2)

# Call API
curl -X POST https://api.example.com/v1/tickets \
  -H "Authorization: Bearer ${API_KEY}:${SIGNATURE}:${TIMESTAMP}" \
  -H "Content-Type: application/json" \
  -d @ticket.json
```

**Benefici**:
- 🔐 Custom authentication logic (qualsiasi schema)
- ⚡ Policy caching (300s) = 1 auth call per 600 requests
- 💰 Economico: $3.50 per 1M auth calls
- 📊 Context propagation: Dati auth disponibili in tutti i Lambda

---

### Esempio 4: Provisioned Concurrency - Warm Starts

**Obiettivo**: Eliminare cold start per endpoint latency-critical.

**Scenario**: `GET /tickets/{id}/solution` deve rispondere in < 200ms sempre.

**Senza Provisioned Concurrency**:
- Cold start: 800ms
- Warm start: 50ms
- Variabilità alta, SLA non garantito

**Con Provisioned Concurrency**:
- Sempre warm: 50ms
- Variabilità bassa, SLA garantito

**CloudFormation**:

```yaml
GetSolutionLambda:
  Type: AWS::Lambda::Function
  Properties:
    FunctionName: GetSolution
    Runtime: python3.11
    Handler: index.lambda_handler
    Code:
      ZipFile: |
        import json
        def lambda_handler(event, context):
            # Business logic
            return {'statusCode': 200, 'body': json.dumps({'solution': '...'})}
    MemorySize: 1024
    Timeout: 30
    Environment:
      Variables:
        DYNAMODB_TABLE: !Ref TicketsTable

# Alias for versioning
GetSolutionAlias:
  Type: AWS::Lambda::Alias
  Properties:
    FunctionName: !Ref GetSolutionLambda
    FunctionVersion: !GetAtt GetSolutionVersion.Version
    Name: prod

GetSolutionVersion:
  Type: AWS::Lambda::Version
  Properties:
    FunctionName: !Ref GetSolutionLambda

# Provisioned Concurrency
GetSolutionProvisionedConcurrency:
  Type: AWS::Lambda::EventInvokeConfig
  Properties:
    FunctionName: !Ref GetSolutionLambda
    Qualifier: !Ref GetSolutionAlias

# Auto-scaling for provisioned concurrency
ProvisionedConcurrencyAutoScaling:
  Type: AWS::ApplicationAutoScaling::ScalableTarget
  Properties:
    ServiceNamespace: lambda
    ResourceId: !Sub function:${GetSolutionLambda}:${GetSolutionAlias}
    ScalableDimension: lambda:function:ProvisionedConcurrentExecutions
    MinCapacity: 10
    MaxCapacity: 100

# Scaling policy
ProvisionedConcurrencyScalingPolicy:
  Type: AWS::ApplicationAutoScaling::ScalingPolicy
  Properties:
    PolicyName: ProvisionedConcurrencyUtilization
    ServiceNamespace: lambda
    ResourceId: !Sub function:${GetSolutionLambda}:${GetSolutionAlias}
    ScalableDimension: lambda:function:ProvisionedConcurrentExecutions
    PolicyType: TargetTrackingScaling
    TargetTrackingScalingPolicyConfiguration:
      TargetValue: 0.70  # 70% utilization
      PredefinedMetricSpecification:
        PredefinedMetricType: LambdaProvisionedConcurrencyUtilization
```

**Alternative: CDK (TypeScript)**:

```typescript
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as apigw from 'aws-cdk-lib/aws-apigateway';

const getSolutionLambda = new lambda.Function(this, 'GetSolutionLambda', {
  runtime: lambda.Runtime.PYTHON_3_11,
  handler: 'index.lambda_handler',
  code: lambda.Code.fromAsset('lambda/get_solution'),
  memorySize: 1024,
  timeout: Duration.seconds(30),
});

// Create alias with provisioned concurrency
const alias = new lambda.Alias(this, 'ProdAlias', {
  aliasName: 'prod',
  version: getSolutionLambda.currentVersion,
  provisionedConcurrentExecutions: 10,  // Always 10 warm containers
});

// Auto-scaling
const autoScaling = alias.addAutoScaling({
  minCapacity: 10,
  maxCapacity: 100,
});

autoScaling.scaleOnUtilization({
  utilizationTarget: 0.70,  // Scale at 70% utilization
});

// API Gateway integration
const api = new apigw.RestApi(this, 'Api');
const tickets = api.root.addResource('tickets');
const ticket = tickets.addResource('{id}');
const solution = ticket.addResource('solution');

solution.addMethod('GET', new apigw.LambdaIntegration(alias));
```

**Costi**:
- Provisioned concurrency: $0.000004115 per GB-second
- On-demand: $0.0000166667 per GB-second
- **Esempio**: 10 containers @ 1GB, 24/7:
  - Provisioned base: $106/month
  - Execution: $0.20 per 1M requests
  - **Total**: ~$106/month per zero cold starts

**Monitoring**:

```python
import boto3
cloudwatch = boto3.client('cloudwatch')

# CloudWatch metrics
response = cloudwatch.get_metric_statistics(
    Namespace='AWS/Lambda',
    MetricName='ProvisionedConcurrencyUtilization',
    Dimensions=[
        {'Name': 'FunctionName', 'Value': 'GetSolution'},
        {'Name': 'Resource', 'Value': 'GetSolution:prod'}
    ],
    StartTime=datetime.utcnow() - timedelta(hours=1),
    EndTime=datetime.utcnow(),
    Period=300,
    Statistics=['Average', 'Maximum']
)
```

---

### Esempio 5: Lambda Layers - Shared Dependencies

**Obiettivo**: Condividere librerie comuni tra Lambda, ridurre package size, semplificare updates.

**Scenario**: 10 Lambda condividono stesso set di dependencies (boto3, requests, numpy, custom utilities).

**Struttura senza Layers** (❌):
```
lambda_function_1.zip (50MB)
  ├── handler.py
  ├── boto3/
  ├── requests/
  ├── numpy/
  └── utils/

lambda_function_2.zip (50MB)
  ├── handler.py
  ├── boto3/  # Duplicato!
  ├── requests/  # Duplicato!
  └── ...

Total: 10 * 50MB = 500MB storage
```

**Struttura con Layers** (✅):
```
layer_common_deps.zip (45MB)
  └── python/
      ├── boto3/
      ├── requests/
      └── numpy/

layer_custom_utils.zip (2MB)
  └── python/
      └── utils/

lambda_function_1.zip (3MB)
  └── handler.py

lambda_function_2.zip (3MB)
  └── handler.py

Total: 45MB + 2MB + (10 * 3MB) = 77MB storage
```

**Build Lambda Layer**:

```bash
#!/bin/bash
# build_layer.sh

# Create layer directory structure
mkdir -p layer/python

# Install dependencies
pip install -r requirements.txt -t layer/python/

# Create zip
cd layer
zip -r ../common-deps-layer.zip .
cd ..

# Publish layer
aws lambda publish-layer-version \
  --layer-name common-dependencies \
  --description "Boto3, requests, numpy" \
  --zip-file fileb://common-deps-layer.zip \
  --compatible-runtimes python3.11 python3.10
```

**requirements.txt**:
```
boto3==1.28.0
requests==2.31.0
numpy==1.24.3
pandas==2.0.3
```

**Custom utilities layer**:

```python
# layer/python/utils/__init__.py

import json
import logging
from typing import Dict, Any

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def parse_request(event: Dict[str, Any]) -> Dict[str, Any]:
    """Parse API Gateway proxy request"""
    body = event.get('body', '{}')
    return json.loads(body) if isinstance(body, str) else body

def success_response(data: Any, status_code: int = 200) -> Dict[str, Any]:
    """Format successful API Gateway response"""
    return {
        'statusCode': status_code,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*'
        },
        'body': json.dumps(data, default=str)
    }

def error_response(error: Any, status_code: int = 500) -> Dict[str, Any]:
    """Format error API Gateway response"""
    return {
        'statusCode': status_code,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps({
            'error': str(error),
            'type': type(error).__name__
        })
    }

class DynamoDBHelper:
    """Helper for common DynamoDB operations"""
    def __init__(self, table_name: str):
        import boto3
        self.dynamodb = boto3.resource('dynamodb')
        self.table = self.dynamodb.Table(table_name)

    def get_item(self, key: Dict[str, Any]) -> Dict[str, Any]:
        response = self.table.get_item(Key=key)
        return response.get('Item')

    def put_item(self, item: Dict[str, Any]):
        self.table.put_item(Item=item)
```

**CloudFormation - Layer + Lambda**:

```yaml
# Layer definition
CommonDepsLayer:
  Type: AWS::Lambda::LayerVersion
  Properties:
    LayerName: common-dependencies
    Description: Boto3, requests, numpy
    Content:
      S3Bucket: !Ref ArtifactsBucket
      S3Key: layers/common-deps-layer.zip
    CompatibleRuntimes:
      - python3.11
      - python3.10

CustomUtilsLayer:
  Type: AWS::Lambda::LayerVersion
  Properties:
    LayerName: custom-utils
    Description: Custom utility functions
    Content:
      S3Bucket: !Ref ArtifactsBucket
      S3Key: layers/custom-utils-layer.zip
    CompatibleRuntimes:
      - python3.11

# Lambda using layers
CreateTicketLambda:
  Type: AWS::Lambda::Function
  Properties:
    FunctionName: CreateTicket
    Runtime: python3.11
    Handler: handler.lambda_handler
    Code:
      S3Bucket: !Ref ArtifactsBucket
      S3Key: lambda/create_ticket.zip
    Layers:
      - !Ref CommonDepsLayer
      - !Ref CustomUtilsLayer
    MemorySize: 512
    Timeout: 30
```

**Lambda handler usando layer**:

```python
# handler.py (3KB solo business logic)

# Import from layer
from utils import parse_request, success_response, error_response, DynamoDBHelper
import os

# Global initialization (outside handler = reuse across invocations)
dynamodb_helper = DynamoDBHelper(os.environ['TICKETS_TABLE'])

def lambda_handler(event, context):
    """Create ticket handler - thin and clean"""
    try:
        # Parse request (from utils layer)
        request_data = parse_request(event)

        # Validate
        validate_ticket_request(request_data)

        # Create ticket
        ticket = create_ticket(request_data)

        # Save to DynamoDB (using layer helper)
        dynamodb_helper.put_item(ticket)

        # Return success (from utils layer)
        return success_response(ticket, status_code=202)

    except ValueError as e:
        return error_response(e, status_code=400)
    except Exception as e:
        logger.exception("Unexpected error")
        return error_response("Internal error", status_code=500)

def validate_ticket_request(data):
    """Validation logic"""
    if not data.get('symptom_text'):
        raise ValueError("symptom_text required")
    # ...

def create_ticket(data):
    """Business logic"""
    # ...
```

**Benefici**:
- 📦 **Package size ridotto**: 50MB → 3MB per Lambda (deploy più veloce)
- ♻️ **Riuso**: Layer condiviso tra tutte le Lambda
- 🔄 **Update facile**: Aggiorna layer, tutte le Lambda beneficiano
- 💰 **Costi storage ridotti**: 500MB → 77MB
- ⚡ **Cold start migliorato**: Meno code da caricare

**Limitazioni**:
- Max 5 layers per Lambda
- Max 250MB unzipped (tutti layers + function code)
- Layer immutabile (versioning obbligatorio)

---

### Esempio 6: VPC Configuration - Private Resource Access

**Obiettivo**: Lambda accede a risorse private (RDS, OpenSearch, ElastiCache) in VPC.

**Scenario**: `GetSolution` Lambda deve query OpenSearch cluster in VPC privata.

**Network architecture**:

```
┌─────────────────────────────────────────────────────┐
│                  VPC 10.0.0.0/16                     │
│                                                      │
│  ┌──────────────────────────────────────────────┐  │
│  │  Private Subnet 10.0.11.0/24 (AZ-1)          │  │
│  │  ┌────────────────┐   ┌──────────────────┐  │  │
│  │  │ Lambda (ENI)   │───│ OpenSearch Node  │  │  │
│  │  └────────────────┘   └──────────────────┘  │  │
│  └──────────────────────────────────────────────┘  │
│                                                      │
│  ┌──────────────────────────────────────────────┐  │
│  │  Private Subnet 10.0.12.0/24 (AZ-2)          │  │
│  │  ┌────────────────┐   ┌──────────────────┐  │  │
│  │  │ Lambda (ENI)   │───│ OpenSearch Node  │  │  │
│  │  └────────────────┘   └──────────────────┘  │  │
│  └──────────────────────────────────────────────┘  │
│                                                      │
│  ┌──────────────────────────────────────────────┐  │
│  │  Public Subnet 10.0.1.0/24                    │  │
│  │  ┌──────────────┐                             │  │
│  │  │ NAT Gateway  │ (for Lambda → Internet)     │  │
│  │  └──────────────┘                             │  │
│  └──────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

**CloudFormation - VPC Setup**:

```yaml
VPC:
  Type: AWS::EC2::VPC
  Properties:
    CidrBlock: 10.0.0.0/16
    EnableDnsHostnames: true
    EnableDnsSupport: true
    Tags:
      - Key: Name
        Value: AIServiceVPC

# Private subnets (Lambda + Data)
PrivateSubnet1:
  Type: AWS::EC2::Subnet
  Properties:
    VpcId: !Ref VPC
    CidrBlock: 10.0.11.0/24
    AvailabilityZone: !Select [0, !GetAZs '']
    Tags:
      - Key: Name
        Value: Private-AZ1

PrivateSubnet2:
  Type: AWS::EC2::Subnet
  Properties:
    VpcId: !Ref VPC
    CidrBlock: 10.0.12.0/24
    AvailabilityZone: !Select [1, !GetAZs '']
    Tags:
      - Key: Name
        Value: Private-AZ2

# Security Group per Lambda
LambdaSecurityGroup:
  Type: AWS::EC2::SecurityGroup
  Properties:
    GroupDescription: Security group for Lambda functions
    VpcId: !Ref VPC
    SecurityGroupEgress:
      # Allow HTTPS to OpenSearch
      - IpProtocol: tcp
        FromPort: 443
        ToPort: 443
        DestinationSecurityGroupId: !Ref OpenSearchSecurityGroup
      # Allow HTTPS to internet (via NAT)
      - IpProtocol: tcp
        FromPort: 443
        ToPort: 443
        CidrIp: 0.0.0.0/0

# Security Group per OpenSearch
OpenSearchSecurityGroup:
  Type: AWS::EC2::SecurityGroup
  Properties:
    GroupDescription: Security group for OpenSearch
    VpcId: !Ref VPC
    SecurityGroupIngress:
      # Allow HTTPS from Lambda only
      - IpProtocol: tcp
        FromPort: 443
        ToPort: 443
        SourceSecurityGroupId: !Ref LambdaSecurityGroup

# Lambda function with VPC config
GetSolutionLambda:
  Type: AWS::Lambda::Function
  Properties:
    FunctionName: GetSolution
    Runtime: python3.11
    Handler: handler.lambda_handler
    Code:
      S3Bucket: !Ref ArtifactsBucket
      S3Key: lambda/get_solution.zip
    VpcConfig:
      SecurityGroupIds:
        - !Ref LambdaSecurityGroup
      SubnetIds:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2
    Environment:
      Variables:
        OPENSEARCH_ENDPOINT: !GetAtt OpenSearchDomain.DomainEndpoint
    MemorySize: 1024
    Timeout: 30
    Role: !GetAtt LambdaExecutionRole.Arn

# IAM Role con VPC permissions
LambdaExecutionRole:
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
      - arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole  # Critical!
      - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
    Policies:
      - PolicyName: OpenSearchAccess
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Action:
                - es:ESHttpGet
                - es:ESHttpPost
              Resource: !Sub ${OpenSearchDomain.Arn}/*
```

**Lambda handler con connection pooling**:

```python
# handler.py

import os
import json
from opensearchpy import OpenSearch, RequestsHttpConnection
from requests_aws4auth import AWS4Auth
import boto3

# GLOBAL SCOPE = Connection reused across invocations (critical for VPC)
_opensearch_client = None

def get_opensearch_client():
    """Lazy initialization of OpenSearch client with connection pooling"""
    global _opensearch_client

    if _opensearch_client is None:
        # AWS Signature V4 auth
        credentials = boto3.Session().get_credentials()
        awsauth = AWS4Auth(
            credentials.access_key,
            credentials.secret_key,
            os.environ['AWS_REGION'],
            'es',
            session_token=credentials.token
        )

        # OpenSearch client with connection pooling
        _opensearch_client = OpenSearch(
            hosts=[{
                'host': os.environ['OPENSEARCH_ENDPOINT'],
                'port': 443
            }],
            http_auth=awsauth,
            use_ssl=True,
            verify_certs=True,
            connection_class=RequestsHttpConnection,
            pool_maxsize=10,  # Connection pool (reuse across invocations)
            timeout=30
        )

    return _opensearch_client

def lambda_handler(event, context):
    """Get solution from OpenSearch"""

    ticket_id = event['pathParameters']['id']

    # Use pooled connection
    opensearch = get_opensearch_client()

    # Query OpenSearch
    response = opensearch.search(
        index='tickets',
        body={
            'query': {
                'term': {'ticket_id': ticket_id}
            }
        }
    )

    if response['hits']['total']['value'] == 0:
        return {
            'statusCode': 404,
            'body': json.dumps({'error': 'Ticket not found'})
        }

    solution = response['hits']['hits'][0]['_source']

    return {
        'statusCode': 200,
        'body': json.dumps(solution)
    }
```

**VPC Endpoints (evitare NAT Gateway)**:

```yaml
# S3 Gateway Endpoint (free)
S3Endpoint:
  Type: AWS::EC2::VPCEndpoint
  Properties:
    VpcId: !Ref VPC
    ServiceName: !Sub com.amazonaws.${AWS::Region}.s3
    RouteTableIds:
      - !Ref PrivateRouteTable
    VpcEndpointType: Gateway

# DynamoDB Gateway Endpoint (free)
DynamoDBEndpoint:
  Type: AWS::EC2::VPCEndpoint
  Properties:
    VpcId: !Ref VPC
    ServiceName: !Sub com.amazonaws.${AWS::Region}.dynamodb
    RouteTableIds:
      - !Ref PrivateRouteTable
    VpcEndpointType: Gateway

# Bedrock Interface Endpoint ($0.01/hour)
BedrockEndpoint:
  Type: AWS::EC2::VPCEndpoint
  Properties:
    VpcId: !Ref VPC
    ServiceName: !Sub com.amazonaws.${AWS::Region}.bedrock-runtime
    VpcEndpointType: Interface
    PrivateDnsEnabled: true
    SubnetIds:
      - !Ref PrivateSubnet1
      - !Ref PrivateSubnet2
    SecurityGroupIds:
      - !Ref LambdaSecurityGroup
```

**Cost comparison**:

| Setup | Cost/month | Notes |
|-------|-----------|-------|
| NAT Gateway | ~$32 + data transfer | Simple, internet access |
| VPC Endpoints (all) | ~$22 | No internet, secure |
| Hybrid (NAT + S3/DDB endpoints) | ~$32 | Best of both |

**Troubleshooting VPC Lambda**:

```python
# Test connectivity from Lambda
def test_connectivity(event, context):
    """Diagnostic Lambda to test VPC connectivity"""
    import socket
    import urllib3

    results = {}

    # Test OpenSearch
    opensearch_host = os.environ['OPENSEARCH_ENDPOINT']
    try:
        socket.create_connection((opensearch_host, 443), timeout=5)
        results['opensearch'] = 'OK'
    except Exception as e:
        results['opensearch'] = f'FAILED: {e}'

    # Test Internet (via NAT or endpoint)
    try:
        http = urllib3.PoolManager()
        http.request('GET', 'https://aws.amazon.com', timeout=5)
        results['internet'] = 'OK'
    except Exception as e:
        results['internet'] = f'FAILED: {e}'

    # Test S3 (via endpoint)
    try:
        s3 = boto3.client('s3')
        s3.list_buckets()
        results['s3_endpoint'] = 'OK'
    except Exception as e:
        results['s3_endpoint'] = f'FAILED: {e}'

    return {'statusCode': 200, 'body': json.dumps(results)}
```

---

### Esempio 7: Idempotency - Duplicate Request Handling

**Obiettivo**: Garantire che richieste duplicate non creino risorse multiple (es. doppio addebito, doppio ticket).

**Scenario**: Client retry su timeout → rischio creazione ticket duplicati.

**Strategia: Idempotency Key**

Client invia header `Idempotency-Key: <uuid>`. Server:
1. Check se key già processata → return cached response
2. Se nuova → process + cache response
3. TTL 24h su cache

**DynamoDB Schema per idempotency**:

```python
# Table: idempotency_keys
# PK: idempotency_key (String)
# Attributes:
#   - status: IN_PROGRESS | COMPLETED | FAILED
#   - response: Cached response (JSON)
#   - created_at: Timestamp
#   - ttl: Expiration (24h)
```

**Lambda decorator per idempotency**:

```python
# idempotency.py (in Layer)

import os
import json
import hashlib
import time
from functools import wraps
from typing import Callable
import boto3
from boto3.dynamodb.conditions import Key

dynamodb = boto3.resource('dynamodb')
idempotency_table = dynamodb.Table(os.environ['IDEMPOTENCY_TABLE'])

class IdempotencyError(Exception):
    """Request already in progress"""
    pass

def idempotent(ttl_seconds: int = 86400):
    """
    Decorator to make Lambda handler idempotent

    Usage:
        @idempotent(ttl_seconds=86400)
        def lambda_handler(event, context):
            # Your logic
    """
    def decorator(handler: Callable):
        @wraps(handler)
        def wrapper(event, context):
            # Extract idempotency key from headers
            headers = event.get('headers', {})
            idempotency_key = headers.get('Idempotency-Key') or headers.get('idempotency-key')

            if not idempotency_key:
                # No idempotency key = proceed normally (not idempotent)
                return handler(event, context)

            # Check if already processed
            try:
                response = idempotency_table.get_item(
                    Key={'idempotency_key': idempotency_key}
                )

                if 'Item' in response:
                    item = response['Item']

                    if item['status'] == 'IN_PROGRESS':
                        # Request already in progress (concurrent request)
                        raise IdempotencyError("Request already in progress")

                    elif item['status'] == 'COMPLETED':
                        # Return cached response
                        return json.loads(item['response'])

                    elif item['status'] == 'FAILED':
                        # Previous request failed, allow retry
                        pass

            except IdempotencyError:
                raise
            except Exception as e:
                # DynamoDB error, log but proceed
                print(f"Idempotency check failed: {e}")

            # Mark as IN_PROGRESS (conditional write to prevent race condition)
            try:
                idempotency_table.put_item(
                    Item={
                        'idempotency_key': idempotency_key,
                        'status': 'IN_PROGRESS',
                        'created_at': int(time.time()),
                        'ttl': int(time.time()) + ttl_seconds
                    },
                    ConditionExpression='attribute_not_exists(idempotency_key)'
                )
            except dynamodb.meta.client.exceptions.ConditionalCheckFailedException:
                # Race condition: another request won
                raise IdempotencyError("Request already in progress")

            # Execute handler
            try:
                result = handler(event, context)

                # Cache successful response
                idempotency_table.update_item(
                    Key={'idempotency_key': idempotency_key},
                    UpdateExpression='SET #status = :status, #response = :response',
                    ExpressionAttributeNames={
                        '#status': 'status',
                        '#response': 'response'
                    },
                    ExpressionAttributeValues={
                        ':status': 'COMPLETED',
                        ':response': json.dumps(result)
                    }
                )

                return result

            except Exception as e:
                # Mark as FAILED
                idempotency_table.update_item(
                    Key={'idempotency_key': idempotency_key},
                    UpdateExpression='SET #status = :status, error_message = :error',
                    ExpressionAttributeNames={'#status': 'status'},
                    ExpressionAttributeValues={
                        ':status': 'FAILED',
                        ':error': str(e)
                    }
                )
                raise

        return wrapper
    return decorator
```

**CloudFormation - Idempotency Table**:

```yaml
IdempotencyTable:
  Type: AWS::DynamoDB::Table
  Properties:
    TableName: idempotency-keys
    BillingMode: PAY_PER_REQUEST
    AttributeDefinitions:
      - AttributeName: idempotency_key
        AttributeType: S
    KeySchema:
      - AttributeName: idempotency_key
        KeyType: HASH
    TimeToLiveSpecification:
      Enabled: true
      AttributeName: ttl
    StreamSpecification:
      StreamViewType: NEW_AND_OLD_IMAGES
```

**Lambda handler usando decorator**:

```python
# handler.py

from idempotency import idempotent
import json
import uuid

@idempotent(ttl_seconds=86400)  # 24h cache
def lambda_handler(event, context):
    """Create ticket - idempotent"""

    body = json.loads(event['body'])

    # Generate ticket ID
    ticket_id = f"tkt_{uuid.uuid4().hex[:12]}"

    # Save ticket to DynamoDB
    save_ticket(ticket_id, body)

    # Trigger Step Functions
    start_processing(ticket_id)

    # Response
    return {
        'statusCode': 202,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps({
            'ticket_id': ticket_id,
            'status': 'PROCESSING'
        })
    }
```

**Client usage**:

```python
# Client with retry + idempotency

import requests
import uuid
import time

def create_ticket_with_retry(ticket_data, max_retries=3):
    """Create ticket with automatic retry and idempotency"""

    # Generate idempotency key (same for all retries)
    idempotency_key = str(uuid.uuid4())

    for attempt in range(max_retries):
        try:
            response = requests.post(
                'https://api.example.com/v1/tickets',
                json=ticket_data,
                headers={
                    'Authorization': f'Bearer {get_token()}',
                    'Idempotency-Key': idempotency_key,
                    'Content-Type': 'application/json'
                },
                timeout=10
            )

            if response.status_code == 202:
                return response.json()
            elif response.status_code == 409:
                # Request in progress, wait and retry
                time.sleep(2 ** attempt)
                continue
            else:
                response.raise_for_status()

        except requests.exceptions.Timeout:
            # Timeout, retry with same idempotency key
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)
                continue
            raise

    raise Exception("Max retries exceeded")
```

**Benefits**:
- ✅ **Safe retries**: Client può retry senza rischio duplicati
- ✅ **Concurrent protection**: Race condition gestite
- ✅ **Automatic caching**: Response salvata 24h
- ✅ **TTL cleanup**: DynamoDB rimuove automaticamente old keys

---

### Esempio 8: Error Handling - Retry Policies e DLQ

**Obiettivo**: Gestire errori Lambda con retry automatico e Dead Letter Queue per failure investigation.

**Scenario**: Lambda processing ticket fallisce sporadicamente (timeout Bedrock, OpenSearch unreachable).

**Error types**:
1. **Transient errors**: Timeout, throttling → **RETRY**
2. **Permanent errors**: Validation, NotFound → **NO RETRY**
3. **Unknown errors**: Unexpected → **RETRY + DLQ**

**CloudFormation - Lambda with retry + DLQ**:

```yaml
# Dead Letter Queue
TicketProcessingDLQ:
  Type: AWS::SQS::Queue
  Properties:
    QueueName: ticket-processing-dlq
    MessageRetentionPeriod: 1209600  # 14 days
    VisibilityTimeout: 300

# SNS Topic for DLQ alerts
DLQAlertTopic:
  Type: AWS::SNS::Topic
  Properties:
    TopicName: dlq-alerts
    Subscription:
      - Endpoint: ops-team@example.com
        Protocol: email

# CloudWatch Alarm on DLQ messages
DLQAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: TicketProcessingDLQAlarm
    AlarmDescription: Alert when messages in DLQ
    MetricName: ApproximateNumberOfMessagesVisible
    Namespace: AWS/SQS
    Statistic: Average
    Period: 300
    EvaluationPeriods: 1
    Threshold: 1
    ComparisonOperator: GreaterThanOrEqualToThreshold
    Dimensions:
      - Name: QueueName
        Value: !GetAtt TicketProcessingDLQ.QueueName
    AlarmActions:
      - !Ref DLQAlertTopic

# Lambda with retry configuration
ProcessTicketLambda:
  Type: AWS::Lambda::Function
  Properties:
    FunctionName: ProcessTicket
    Runtime: python3.11
    Handler: handler.lambda_handler
    Code:
      S3Bucket: !Ref ArtifactsBucket
      S3Key: lambda/process_ticket.zip
    MemorySize: 1024
    Timeout: 60
    DeadLetterConfig:
      TargetArn: !GetAtt TicketProcessingDLQ.Arn
    Environment:
      Variables:
        OPENSEARCH_ENDPOINT: !GetAtt OpenSearchDomain.DomainEndpoint
        BEDROCK_MODEL_ID: anthropic.claude-3-sonnet-20240229-v1:0

# Event Source Mapping (SQS → Lambda) with retry
TicketQueueEventSource:
  Type: AWS::Lambda::EventSourceMapping
  Properties:
    FunctionName: !Ref ProcessTicketLambda
    EventSourceArn: !GetAtt TicketQueue.Arn
    BatchSize: 10
    MaximumBatchingWindowInSeconds: 5
    FunctionResponseTypes:
      - ReportBatchItemFailures  # Partial batch failure handling
    ScalingConfig:
      MaximumConcurrency: 100

# Asynchronous invocation configuration
ProcessTicketAsyncConfig:
  Type: AWS::Lambda::EventInvokeConfig
  Properties:
    FunctionName: !Ref ProcessTicketLambda
    Qualifier: $LATEST
    MaximumRetryAttempts: 2  # Retry 2 times
    MaximumEventAgeInSeconds: 3600  # Discard after 1h
    DestinationConfig:
      OnFailure:
        Destination: !GetAtt TicketProcessingDLQ.Arn
      OnSuccess:
        Destination: !GetAtt SuccessEventBus.Arn  # Optional success tracking
```

**Lambda handler con error classification**:

```python
# handler.py

import json
import logging
import traceback
from typing import Dict, Any, List
import boto3
from botocore.exceptions import ClientError

logger = logging.getLogger()
logger.setLevel(logging.INFO)

# Error classification
class RetryableError(Exception):
    """Errors that should trigger retry"""
    pass

class PermanentError(Exception):
    """Errors that should NOT retry (fail immediately)"""
    pass

def lambda_handler(event, context):
    """
    Process ticket with proper error handling

    For SQS batch: Return partial failures
    For async invoke: Raise for retry
    """

    # Check if SQS batch
    if 'Records' in event:
        return handle_sqs_batch(event, context)
    else:
        return handle_single_event(event, context)

def handle_sqs_batch(event: Dict[str, Any], context: Any) -> Dict[str, Any]:
    """Handle SQS batch with partial failure reporting"""

    batch_item_failures = []

    for record in event['Records']:
        try:
            message = json.loads(record['body'])
            process_ticket(message)

        except PermanentError as e:
            # Permanent error: Log and remove from queue (no retry)
            logger.error(f"Permanent error processing {record['messageId']}: {e}")
            send_to_dlq(record, str(e), permanent=True)

        except RetryableError as e:
            # Retryable error: Add to failed items (will retry)
            logger.warning(f"Retryable error processing {record['messageId']}: {e}")
            batch_item_failures.append({'itemIdentifier': record['messageId']})

        except Exception as e:
            # Unknown error: Treat as retryable
            logger.exception(f"Unknown error processing {record['messageId']}")
            batch_item_failures.append({'itemIdentifier': record['messageId']})

    # Return partial batch failures
    return {'batchItemFailures': batch_item_failures}

def handle_single_event(event: Dict[str, Any], context: Any) -> Dict[str, Any]:
    """Handle async invocation (API Gateway → Lambda)"""

    try:
        result = process_ticket(event)
        return {
            'statusCode': 200,
            'body': json.dumps(result)
        }

    except PermanentError as e:
        # Permanent error: Return 4xx (no retry)
        logger.error(f"Permanent error: {e}")
        return {
            'statusCode': 400,
            'body': json.dumps({'error': str(e)})
        }

    except RetryableError as e:
        # Retryable error: Raise exception (Lambda retries)
        logger.warning(f"Retryable error, will retry: {e}")
        raise

    except Exception as e:
        # Unknown error: Raise (Lambda retries)
        logger.exception("Unknown error, will retry")
        raise

def process_ticket(data: Dict[str, Any]) -> Dict[str, Any]:
    """Process ticket - classify errors appropriately"""

    ticket_id = data.get('ticket_id')

    # Validation errors = Permanent
    if not ticket_id:
        raise PermanentError("Missing ticket_id")

    try:
        # Retrieve from knowledge base
        context = retrieve_knowledge(data)

    except ClientError as e:
        error_code = e.response['Error']['Code']

        # Throttling = Retryable
        if error_code in ['ThrottlingException', 'TooManyRequestsException']:
            raise RetryableError(f"Throttled: {e}")

        # NotFound = Permanent
        elif error_code in ['ResourceNotFoundException', 'NoSuchKey']:
            raise PermanentError(f"Resource not found: {e}")

        # Other AWS errors = Retryable
        else:
            raise RetryableError(f"AWS error: {e}")

    try:
        # Generate solution with Bedrock
        solution = generate_solution(context, data)

    except ClientError as e:
        error_code = e.response['Error']['Code']

        # Bedrock throttling = Retryable
        if error_code == 'ThrottlingException':
            raise RetryableError(f"Bedrock throttled: {e}")

        # Model not found = Permanent
        elif error_code == 'ResourceNotFoundException':
            raise PermanentError(f"Model not found: {e}")

        # Validation error = Permanent
        elif error_code == 'ValidationException':
            raise PermanentError(f"Invalid request: {e}")

        else:
            raise RetryableError(f"Bedrock error: {e}")

    # Save solution
    try:
        save_solution(ticket_id, solution)
    except Exception as e:
        # DynamoDB errors typically retryable
        raise RetryableError(f"Failed to save: {e}")

    return {'ticket_id': ticket_id, 'status': 'completed'}

def send_to_dlq(record: Dict, error: str, permanent: bool = False):
    """Send failed message to DLQ with metadata"""

    sqs = boto3.client('sqs')

    dlq_message = {
        'original_message': record['body'],
        'error': error,
        'error_type': 'permanent' if permanent else 'retryable',
        'message_id': record['messageId'],
        'timestamp': record.get('timestamp'),
        'stack_trace': traceback.format_exc()
    }

    sqs.send_message(
        QueueUrl=os.environ['DLQ_URL'],
        MessageBody=json.dumps(dlq_message)
    )
```

**DLQ Processor Lambda (investigate failures)**:

```python
# dlq_processor.py

import json
import boto3

cloudwatch = boto3.client('cloudwatch')
sns = boto3.client('sns')

def lambda_handler(event, context):
    """
    Process messages in DLQ
    - Log for investigation
    - Send alert
    - Optionally replay
    """

    for record in event['Records']:
        dlq_message = json.loads(record['body'])

        # Log detailed error
        logger.error(
            f"DLQ Message",
            extra={
                'message_id': dlq_message['message_id'],
                'error_type': dlq_message['error_type'],
                'error': dlq_message['error'],
                'original_message': dlq_message['original_message']
            }
        )

        # Send CloudWatch metric
        cloudwatch.put_metric_data(
            Namespace='TicketProcessing',
            MetricData=[{
                'MetricName': 'DLQMessages',
                'Value': 1,
                'Unit': 'Count',
                'Dimensions': [
                    {'Name': 'ErrorType', 'Value': dlq_message['error_type']}
                ]
            }]
        )

        # Alert ops team
        if dlq_message['error_type'] == 'permanent':
            sns.publish(
                TopicArn=os.environ['ALERT_TOPIC_ARN'],
                Subject=f"Permanent failure: {dlq_message['message_id']}",
                Message=json.dumps(dlq_message, indent=2)
            )
```

**Monitoring dashboard**:

```python
# CloudWatch Insights query

# Failed invocations by error type
fields @timestamp, error_type, error
| filter @message like /DLQ Message/
| stats count() by error_type

# Average retry attempts before success
fields @timestamp, ticket_id
| filter @message like /Retryable error/
| stats count() as retry_count by ticket_id
| stats avg(retry_count) as avg_retries
```

**Benefits**:
- ✅ **Automatic retry**: Transient errors handled automatically
- ✅ **Cost efficient**: No retry per permanent errors
- ✅ **Observability**: DLQ provides failure audit trail
- ✅ **Alerting**: Ops team notified of critical failures
- ✅ **Replay capability**: Messages in DLQ can be replayed after fix

## Best Practices

### Handler Pattern: Thin Handler, Fat Service

**Do**:
```python
# ✅ Separazione concerns
def lambda_handler(event, context):
    request = parse_request(event)
    result = TicketService().create(request)
    return success_response(result)

class TicketService:
    def create(self, data):
        # Business logic qui
        pass
```

**Don't**:
```python
# ❌ Tutto nel handler
def lambda_handler(event, context):
    # 200 righe di business logic
    # Difficile testare, riusare, debug
    pass
```

### Dependency Injection

**Do**:
```python
# ✅ DI per testability
class TicketService:
    def __init__(self, dynamodb_client, bedrock_client):
        self.dynamodb = dynamodb_client
        self.bedrock = bedrock_client

    def create(self, data):
        self.dynamodb.put_item(...)

# In Lambda
def lambda_handler(event, context):
    service = TicketService(
        dynamodb_client=boto3.client('dynamodb'),
        bedrock_client=boto3.client('bedrock-runtime')
    )
    return service.create(parse_request(event))

# In tests
def test_create_ticket():
    mock_dynamodb = Mock()
    service = TicketService(mock_dynamodb, Mock())
    service.create({'ticket_id': 'test'})
    mock_dynamodb.put_item.assert_called_once()
```

### Logging Structured Data

**Do**:
```python
# ✅ JSON logging
import json
import logging

logger = logging.getLogger()

def lambda_handler(event, context):
    logger.info(json.dumps({
        'event': 'ticket_created',
        'ticket_id': ticket_id,
        'customer_id': customer_id,
        'latency_ms': latency,
        'request_id': context.request_id
    }))
```

**Don't**:
```python
# ❌ String logging (difficile query)
logger.info(f"Created ticket {ticket_id} for customer {customer_id}")
```

### Connection Pooling (VPC Lambda)

**Do**:
```python
# ✅ Global scope = reuse
import boto3

# Outside handler = singleton
opensearch_client = OpenSearch(...)

def lambda_handler(event, context):
    # Reuse connection
    opensearch_client.search(...)
```

**Don't**:
```python
# ❌ Inside handler = new connection ogni invocazione
def lambda_handler(event, context):
    opensearch_client = OpenSearch(...)  # Lento!
    opensearch_client.search(...)
```

### Environment Variables e Secrets

**Do**:
```python
# ✅ Secrets Manager per credentials
import os
import boto3
import json

secrets_client = boto3.client('secretsmanager')

def get_api_key():
    secret_arn = os.environ['API_KEY_SECRET_ARN']
    response = secrets_client.get_secret_value(SecretId=secret_arn)
    return json.loads(response['SecretString'])['api_key']
```

**Don't**:
```python
# ❌ Hardcoded o in environment variable
API_KEY = "sk_live_123456"  # Mai!
```

### Memory vs CPU Optimization

Lambda CPU è proporzionale a memoria:
- 128 MB = 0.083 vCPU
- 1024 MB = 0.67 vCPU
- 1769 MB = 1 full vCPU
- 10240 MB = 6 vCPU

**Strategia**:
1. Benchmark con diverse memory size
2. Calcola costo per invocazione = (duration * memory * price)
3. Sweet spot spesso 1024-2048 MB (duration ↓ compensa memory ↑)

**Esempio**:
```
Memory: 512MB  → Duration: 2000ms → Cost: $0.000001667
Memory: 1024MB → Duration: 1000ms → Cost: $0.000001667
Memory: 2048MB → Duration: 600ms  → Cost: $0.000002000
```
Scelta: **1024MB** (costo uguale, latenza -50%)

### Cold Start Mitigation Checklist

- [ ] Package size < 50MB (use Layers)
- [ ] Minimize dependencies (solo necessarie)
- [ ] Lazy load heavy libraries
- [ ] Use Provisioned Concurrency per latency-critical endpoints
- [ ] Consider runtime (Python > Java per cold start)
- [ ] Connection pooling in global scope
- [ ] VPC: Minimize subnet count

### Error Handling Strategy

**Classify errors**:
1. **4xx Permanent**: ValidationError, NotFound → No retry, log, return 4xx
2. **5xx Retryable**: Timeout, Throttling → Retry with backoff
3. **Unknown**: Default to retryable, investigate via DLQ

**Implement**:
- Custom exception classes (PermanentError, RetryableError)
- DLQ per all async invocations
- CloudWatch Alarms on DLQ depth
- Structured logging per error investigation

### Security Best Practices

- [ ] **Least privilege IAM**: Solo permissions necessarie
- [ ] **Secrets in Secrets Manager**: No environment variables per credentials
- [ ] **VPC per risorse private**: Lambda in VPC per accesso RDS/OpenSearch
- [ ] **Encryption at rest**: Environment variables encrypted (KMS)
- [ ] **Validation input**: JSON Schema in API Gateway
- [ ] **Rate limiting**: Usage Plans in API Gateway
- [ ] **CORS configurato**: Whitelist domains specifici
- [ ] **CloudTrail enabled**: Audit all API calls

### Cost Optimization

**Lambda**:
- Use Graviton2 (arm64): 20% cheaper, 19% faster
- Right-size memory (benchmark!)
- Reserved concurrency solo se necessario
- Use SQS buffering per smooth spikes (vs scaling Lambda)

**API Gateway**:
- Use HTTP API invece REST API (70% cheaper) se non serve full features
- Caching per GET requests (reduce Lambda invocations)
- Regional endpoint (no CloudFront se non serve global)

**VPC**:
- Use VPC Endpoints per AWS services (no NAT Gateway)
- Share NAT Gateway tra Lambda (no per-function NAT)

## Troubleshooting

### Problema 1: Cold Start Alto (> 2s)

**Sintomi**:
- Latency p95 > 2000ms
- Latency bimodale (50ms o 2000ms)
- CloudWatch: Init Duration > 1500ms

**Diagnosi**:

```python
# Lambda Insights metric
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name InitDuration \
  --dimensions Name=FunctionName,Value=CreateTicket \
  --start-time 2025-11-18T00:00:00Z \
  --end-time 2025-11-18T23:59:59Z \
  --period 3600 \
  --statistics Average,Maximum
```

**Soluzioni**:

1. **Reduce package size**:
```bash
# Check deployment package size
aws lambda get-function --function-name CreateTicket \
  | jq '.Code.CodeSize'

# If > 50MB, move dependencies to Layer
```

2. **Lazy load libraries**:
```python
# ❌ Import at top (loaded always)
import pandas as pd
import numpy as np

def lambda_handler(event, context):
    # Use pandas only 10% of time
    if event.get('advanced_analytics'):
        df = pd.DataFrame(...)

# ✅ Import when needed
def lambda_handler(event, context):
    if event.get('advanced_analytics'):
        import pandas as pd  # Loaded solo se necessario
        df = pd.DataFrame(...)
```

3. **Use Provisioned Concurrency**:
```bash
aws lambda put-provisioned-concurrency-config \
  --function-name CreateTicket \
  --qualifier prod \
  --provisioned-concurrent-executions 10
```

4. **Switch to Python/Node (from Java/C#)**:
   - Python 3.11 cold start: 200-500ms
   - Java 11 cold start: 2000-5000ms

---

### Problema 2: Lambda Throttling (429 Errors)

**Sintomi**:
- API Gateway ritorna 429 Too Many Requests
- CloudWatch: ThrottledInvocations > 0
- Spike traffico non gestito

**Diagnosi**:

```bash
# Check concurrent executions
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name ConcurrentExecutions \
  --dimensions Name=FunctionName,Value=CreateTicket \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Maximum

# Check account limit
aws lambda get-account-settings \
  | jq '.AccountLimit.ConcurrentExecutions'
```

**Soluzioni**:

1. **Request limit increase**:
```bash
# Via AWS Support, richiedi aumento da 1000 → 5000
```

2. **Reserved concurrency** (proteggi altre Lambda):
```bash
aws lambda put-function-concurrency \
  --function-name CreateTicket \
  --reserved-concurrent-executions 500
```

3. **SQS buffering** (smooth spikes):
```yaml
# API Gateway → SQS → Lambda (invece di diretto)
# Lambda processes da SQS a ritmo controllato
TicketQueue:
  Type: AWS::SQS::Queue
  Properties:
    VisibilityTimeout: 300

EventSourceMapping:
  Type: AWS::Lambda::EventSourceMapping
  Properties:
    FunctionName: !Ref CreateTicketLambda
    EventSourceArn: !GetAtt TicketQueue.Arn
    BatchSize: 10
    MaximumConcurrency: 100  # Limit concurrent Lambda
```

---

### Problema 3: VPC Lambda Timeout

**Sintomi**:
- Lambda timeout accessing RDS/OpenSearch
- CloudWatch: Task timed out after 30.00 seconds
- No errors, solo timeout

**Diagnosi**:

```python
# Test connectivity
def lambda_handler(event, context):
    import socket

    # Test OpenSearch
    host = os.environ['OPENSEARCH_ENDPOINT']
    try:
        s = socket.create_connection((host, 443), timeout=5)
        s.close()
        return "Connection OK"
    except Exception as e:
        return f"Connection FAILED: {e}"
```

**Soluzioni**:

1. **Check Security Groups**:
```bash
# Lambda SG deve avere outbound rule verso OpenSearch SG
# OpenSearch SG deve avere inbound rule da Lambda SG

aws ec2 describe-security-groups \
  --group-ids sg-lambda sg-opensearch
```

2. **Check Route Tables**:
```bash
# Private subnet deve avere route a NAT Gateway (per internet)
# oppure VPC Endpoints (per AWS services)

aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=subnet-private"
```

3. **Check NACLs**:
```bash
# Network ACL deve permettere traffico ephemeral ports

aws ec2 describe-network-acls \
  --filters "Name=association.subnet-id,Values=subnet-private"
```

4. **Use VPC Endpoints** (avoid NAT):
```yaml
# S3, DynamoDB, Bedrock endpoints
S3Endpoint:
  Type: AWS::EC2::VPCEndpoint
  Properties:
    VpcId: !Ref VPC
    ServiceName: !Sub com.amazonaws.${AWS::Region}.s3
    RouteTableIds: [!Ref PrivateRouteTable]
```

---

### Problema 4: API Gateway 502 Bad Gateway

**Sintomi**:
- API Gateway ritorna 502 Bad Gateway
- Lambda execution succeeded (200 OK in logs)
- Response format issue

**Diagnosi**:

```python
# Lambda logs show success, but API Gateway returns 502
# → Lambda response format incorrect
```

**Causa comune**: Lambda response non segue formato richiesto.

**Soluzioni**:

1. **Lambda Proxy format** (required):
```python
# ✅ Correct format
return {
    'statusCode': 200,
    'headers': {'Content-Type': 'application/json'},
    'body': json.dumps({'result': 'ok'})  # MUST be string!
}

# ❌ Wrong formats
return {'result': 'ok'}  # Missing statusCode
return {
    'statusCode': 200,
    'body': {'result': 'ok'}  # Body not string!
}
```

2. **Enable CloudWatch logs** per API Gateway:
```yaml
ApiGatewayAccount:
  Type: AWS::ApiGateway::Account
  Properties:
    CloudWatchRoleArn: !GetAtt ApiGatewayCloudWatchRole.Arn

Stage:
  Type: AWS::ApiGateway::Stage
  Properties:
    StageName: prod
    RestApiId: !Ref ApiGateway
    MethodSettings:
      - ResourcePath: /*
        HttpMethod: '*'
        LoggingLevel: INFO
        DataTraceEnabled: true
```

3. **Check timeout**:
```yaml
# API Gateway timeout max 29s
# Lambda timeout deve essere < 29s

CreateTicketMethod:
  Type: AWS::ApiGateway::Method
  Properties:
    Integration:
      TimeoutInMillis: 29000  # 29s max
```

---

### Problema 5: High DynamoDB Costs (Idempotency Table)

**Sintomi**:
- DynamoDB costs alto per idempotency table
- Molte write/read requests
- TTL non removing expired items

**Diagnosi**:

```bash
# Check item count
aws dynamodb describe-table \
  --table-name idempotency-keys \
  | jq '.Table.ItemCount'

# Check RCU/WCU consumed
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name ConsumedReadCapacityUnits \
  --dimensions Name=TableName,Value=idempotency-keys \
  --start-time $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 3600 \
  --statistics Sum
```

**Soluzioni**:

1. **Verify TTL enabled**:
```bash
aws dynamodb describe-time-to-live \
  --table-name idempotency-keys

# Enable if not
aws dynamodb update-time-to-live \
  --table-name idempotency-keys \
  --time-to-live-specification "Enabled=true, AttributeName=ttl"
```

2. **Reduce TTL** (se 24h troppo lungo):
```python
# Change from 86400s (24h) to 3600s (1h)
'ttl': int(time.time()) + 3600
```

3. **Use DynamoDB Streams** per cleanup immediato:
```python
# Lambda triggered by DynamoDB Stream
# Delete item subito dopo COMPLETED status (no wait TTL)
def lambda_handler(event, context):
    for record in event['Records']:
        if record['eventName'] == 'MODIFY':
            new_image = record['dynamodb']['NewImage']
            if new_image['status']['S'] == 'COMPLETED':
                # Delete immediately (no wait 24h TTL)
                dynamodb.delete_item(
                    TableName='idempotency-keys',
                    Key={'idempotency_key': new_image['idempotency_key']}
                )
```

## Esempi Reali dal Progetto

### Esempio 1: POST /tickets - Create Ticket Flow

**Architettura completa**:

```
Client
  ↓ POST /tickets + Idempotency-Key
API Gateway
  ↓ Cognito Authorizer (JWT validation)
  ↓ Request Validation (JSON Schema)
  ↓ Lambda Proxy Integration
Lambda: CreateTicket (512MB, 30s timeout)
  ↓ Check idempotency (DynamoDB)
  ↓ Generate ticket ID
  ↓ Save to DynamoDB
  ↓ Publish to EventBridge
  ↓ Start Step Functions
  ↓ Return 202 Accepted
```

**Lambda handler completo**:

```python
# create_ticket/handler.py

import os
import json
import uuid
import time
import boto3
from decimal import Decimal
from utils import parse_request, success_response, error_response, DynamoDBHelper
from idempotency import idempotent

# Global clients (reused across invocations)
dynamodb_helper = DynamoDBHelper(os.environ['TICKETS_TABLE'])
eventbridge = boto3.client('events')
stepfunctions = boto3.client('stepfunctions')

@idempotent(ttl_seconds=86400)
def lambda_handler(event, context):
    """
    Create ticket endpoint

    POST /tickets
    Request: {customer, asset, symptom_text, ...}
    Response: 202 {ticket_id, status}
    """

    # Parse and validate request
    try:
        request_data = parse_request(event)
        validate_request(request_data)
    except ValueError as e:
        return error_response(str(e), status_code=400)

    # Extract authorizer context
    authorizer = event['requestContext']['authorizer']
    customer_id = authorizer.get('claims', {}).get('custom:customer_id')

    # Generate ticket
    ticket = create_ticket_record(request_data, customer_id)

    # Save to DynamoDB
    try:
        dynamodb_helper.put_item(ticket)
    except Exception as e:
        logger.exception("Failed to save ticket")
        return error_response("Failed to create ticket", status_code=500)

    # Publish event to EventBridge
    try:
        publish_ticket_created_event(ticket)
    except Exception as e:
        logger.warning(f"Failed to publish event: {e}")
        # Non-critical, continue

    # Start Step Functions workflow
    try:
        execution_arn = start_processing_workflow(ticket)
        ticket['execution_arn'] = execution_arn
    except Exception as e:
        logger.exception("Failed to start workflow")
        # Mark ticket as failed
        dynamodb_helper.update_item(
            Key={'ticket_id': ticket['ticket_id']},
            UpdateExpression='SET #status = :failed',
            ExpressionAttributeNames={'#status': 'status'},
            ExpressionAttributeValues={':failed': 'FAILED'}
        )
        return error_response("Failed to start processing", status_code=500)

    # Return success
    return success_response({
        'ticket_id': ticket['ticket_id'],
        'status': ticket['status'],
        'created_at': ticket['created_at'],
        'estimated_completion': ticket['estimated_completion']
    }, status_code=202)

def validate_request(data):
    """Validate ticket request"""
    required = ['customer', 'asset', 'symptom_text']
    for field in required:
        if field not in data:
            raise ValueError(f"Missing required field: {field}")

    if len(data['symptom_text']) < 10:
        raise ValueError("symptom_text must be at least 10 characters")

def create_ticket_record(data, customer_id):
    """Create ticket DynamoDB record"""
    ticket_id = f"tkt_{uuid.uuid4().hex[:12]}"
    timestamp = int(time.time())

    return {
        'ticket_id': ticket_id,
        'customer_id': customer_id or data['customer']['id'],
        'status': 'PROCESSING',
        'customer': data['customer'],
        'asset': data['asset'],
        'symptom_text': data['symptom_text'],
        'error_code': data.get('error_code'),
        'priority': data.get('priority', 'P3'),
        'lang': data.get('lang', 'it-IT'),
        'policies': data.get('policies', {}),
        'created_at': timestamp,
        'updated_at': timestamp,
        'estimated_completion': timestamp + 120,  # 2 min
        'ttl': timestamp + 2592000  # 30 days
    }

def publish_ticket_created_event(ticket):
    """Publish to EventBridge"""
    eventbridge.put_events(
        Entries=[{
            'Source': 'ai-service.tickets',
            'DetailType': 'TicketCreated',
            'Detail': json.dumps(ticket, default=str),
            'EventBusName': os.environ['EVENT_BUS_NAME']
        }]
    )

def start_processing_workflow(ticket):
    """Start Step Functions execution"""
    response = stepfunctions.start_execution(
        stateMachineArn=os.environ['STATE_MACHINE_ARN'],
        name=f"ticket-{ticket['ticket_id']}-{int(time.time())}",
        input=json.dumps(ticket, default=str)
    )
    return response['executionArn']
```

**CloudFormation completo**:

```yaml
# API Gateway + Lambda + DynamoDB

ApiGateway:
  Type: AWS::ApiGateway::RestApi
  Properties:
    Name: AI-Service-API
    EndpointConfiguration:
      Types: [REGIONAL]

TicketsResource:
  Type: AWS::ApiGateway::Resource
  Properties:
    RestApiId: !Ref ApiGateway
    ParentId: !GetAtt ApiGateway.RootResourceId
    PathPart: tickets

CreateTicketMethod:
  Type: AWS::ApiGateway::Method
  Properties:
    RestApiId: !Ref ApiGateway
    ResourceId: !Ref TicketsResource
    HttpMethod: POST
    AuthorizationType: COGNITO_USER_POOLS
    AuthorizerId: !Ref CognitoAuthorizer
    RequestValidatorId: !Ref RequestValidator
    RequestModels:
      application/json: !Ref CreateTicketModel
    Integration:
      Type: AWS_PROXY
      IntegrationHttpMethod: POST
      Uri: !Sub arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${CreateTicketLambda.Arn}/invocations
    MethodResponses:
      - StatusCode: 202

CreateTicketLambda:
  Type: AWS::Lambda::Function
  Properties:
    FunctionName: CreateTicket
    Runtime: python3.11
    Handler: handler.lambda_handler
    Code:
      S3Bucket: !Ref ArtifactsBucket
      S3Key: lambda/create_ticket.zip
    Layers:
      - !Ref CommonDepsLayer
      - !Ref CustomUtilsLayer
    MemorySize: 512
    Timeout: 30
    Environment:
      Variables:
        TICKETS_TABLE: !Ref TicketsTable
        IDEMPOTENCY_TABLE: !Ref IdempotencyTable
        EVENT_BUS_NAME: !Ref EventBus
        STATE_MACHINE_ARN: !Ref TicketProcessingStateMachine
    Role: !GetAtt CreateTicketLambdaRole.Arn

TicketsTable:
  Type: AWS::DynamoDB::Table
  Properties:
    TableName: tickets
    BillingMode: PAY_PER_REQUEST
    AttributeDefinitions:
      - AttributeName: ticket_id
        AttributeType: S
      - AttributeName: customer_id
        AttributeType: S
      - AttributeName: created_at
        AttributeType: N
    KeySchema:
      - AttributeName: ticket_id
        KeyType: HASH
    GlobalSecondaryIndexes:
      - IndexName: customer-created-index
        KeySchema:
          - AttributeName: customer_id
            KeyType: HASH
          - AttributeName: created_at
            KeyType: RANGE
        Projection:
          ProjectionType: ALL
    StreamSpecification:
      StreamViewType: NEW_AND_OLD_IMAGES
    TimeToLiveSpecification:
      Enabled: true
      AttributeName: ttl
```

---

### Esempio 2: GET /tickets/{id}/solution/stream - SSE Streaming

**Obiettivo**: Streaming soluzione in tempo reale mentre Bedrock genera.

**Lambda handler con streaming**:

```python
# get_solution_stream/handler.py

import os
import json
import boto3
from utils import parse_request

bedrock_runtime = boto3.client('bedrock-runtime')

def lambda_handler(event, context):
    """
    Stream solution via Server-Sent Events

    GET /tickets/{id}/solution/stream
    Response: text/event-stream
    """

    ticket_id = event['pathParameters']['id']

    # Get ticket from DynamoDB
    ticket = get_ticket(ticket_id)
    if not ticket:
        return {
            'statusCode': 404,
            'body': json.dumps({'error': 'Ticket not found'})
        }

    # Check if solution ready
    if ticket.get('status') != 'READY':
        return {
            'statusCode': 409,
            'body': json.dumps({'error': 'Solution not ready yet'})
        }

    # Stream from Bedrock
    return stream_solution(ticket)

def stream_solution(ticket):
    """Stream response from Bedrock with SSE format"""

    # Bedrock streaming invocation
    response = bedrock_runtime.invoke_model_with_response_stream(
        modelId='anthropic.claude-3-sonnet-20240229-v1:0',
        body=json.dumps({
            'anthropic_version': 'bedrock-2023-05-31',
            'max_tokens': 2000,
            'messages': [{
                'role': 'user',
                'content': f"Generate solution for: {ticket['symptom_text']}"
            }]
        })
    )

    # Process stream
    def generate_sse():
        """Generator for SSE events"""

        # Start event
        yield format_sse_event('start', {
            'ticket_id': ticket['ticket_id'],
            'status': 'streaming'
        })

        # Stream chunks
        for event in response['body']:
            chunk = json.loads(event['chunk']['bytes'])

            if chunk['type'] == 'content_block_delta':
                text = chunk['delta']['text']
                yield format_sse_event('chunk', {
                    'type': 'text',
                    'content': text
                })

            elif chunk['type'] == 'message_stop':
                # Complete event
                yield format_sse_event('complete', {
                    'status': 'completed',
                    'total_tokens': chunk.get('usage', {}).get('output_tokens')
                })

    # Return streaming response
    # Note: API Gateway non supporta streaming response nativo
    # Serve usare Lambda Function URL o ALB
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'text/event-stream',
            'Cache-Control': 'no-cache',
            'Connection': 'keep-alive'
        },
        'body': generate_sse()  # Generator
    }

def format_sse_event(event_type, data):
    """Format Server-Sent Event"""
    return f"event: {event_type}\ndata: {json.dumps(data)}\n\n"
```

**Note**: API Gateway REST API non supporta streaming response. Usare:
- **Lambda Function URL** (supporta response streaming)
- **Application Load Balancer** → Lambda
- **API Gateway HTTP API** (preview support)

---

### Esempio 3: POST /kb/search - Knowledge Base Search

**Lambda handler**:

```python
# kb_search/handler.py

import os
import json
import boto3
from opensearchpy import OpenSearch, RequestsHttpConnection
from requests_aws4auth import AWS4Auth
from utils import parse_request, success_response, error_response

# Global OpenSearch client
_opensearch_client = None

def get_opensearch_client():
    global _opensearch_client
    if _opensearch_client is None:
        credentials = boto3.Session().get_credentials()
        awsauth = AWS4Auth(
            credentials.access_key,
            credentials.secret_key,
            os.environ['AWS_REGION'],
            'es',
            session_token=credentials.token
        )

        _opensearch_client = OpenSearch(
            hosts=[{'host': os.environ['OPENSEARCH_ENDPOINT'], 'port': 443}],
            http_auth=awsauth,
            use_ssl=True,
            verify_certs=True,
            connection_class=RequestsHttpConnection,
            pool_maxsize=10
        )

    return _opensearch_client

def lambda_handler(event, context):
    """
    Search knowledge base

    POST /kb/search
    Request: {query, filters, max_results}
    Response: {results: [...]}
    """

    try:
        request_data = parse_request(event)

        # Validate
        if not request_data.get('query'):
            return error_response("Missing query", status_code=400)

        # Search OpenSearch
        results = search_knowledge_base(
            query=request_data['query'],
            filters=request_data.get('filters', {}),
            max_results=request_data.get('max_results', 5)
        )

        return success_response({
            'query': request_data['query'],
            'results': results,
            'total_results': len(results)
        })

    except Exception as e:
        logger.exception("Search failed")
        return error_response("Search failed", status_code=500)

def search_knowledge_base(query, filters, max_results):
    """Hybrid search: BM25 + k-NN"""

    opensearch = get_opensearch_client()

    # Build query
    search_body = {
        'size': max_results,
        'query': {
            'bool': {
                'should': [
                    # Full-text search (BM25)
                    {
                        'multi_match': {
                            'query': query,
                            'fields': ['title^2', 'content'],
                            'type': 'best_fields'
                        }
                    },
                    # Vector search (k-NN) - requires embedding
                    # {
                    #     'knn': {
                    #         'embedding': {
                    #             'vector': generate_embedding(query),
                    #             'k': max_results
                    #         }
                    #     }
                    # }
                ],
                'filter': build_filters(filters)
            }
        },
        'highlight': {
            'fields': {
                'content': {
                    'fragment_size': 150,
                    'number_of_fragments': 1
                }
            }
        }
    }

    # Execute search
    response = opensearch.search(
        index='knowledge-base',
        body=search_body
    )

    # Format results
    return [
        {
            'id': hit['_id'],
            'source_title': hit['_source']['title'],
            'snippet': hit.get('highlight', {}).get('content', [''])[0],
            'score': hit['_score'],
            'metadata': hit['_source'].get('metadata', {})
        }
        for hit in response['hits']['hits']
    ]

def build_filters(filters):
    """Build OpenSearch filter clause"""
    filter_clauses = []

    if filters.get('product_model'):
        filter_clauses.append({
            'term': {'metadata.product_model': filters['product_model']}
        })

    if filters.get('error_code'):
        filter_clauses.append({
            'term': {'metadata.error_code': filters['error_code']}
        })

    if filters.get('doc_type'):
        filter_clauses.append({
            'terms': {'metadata.doc_type': filters['doc_type']}
        })

    return filter_clauses
```

## Riferimenti

### Documentazione Interna
- [API Specification](../05-api-specification.md) - Dettagli endpoint REST
- [Architecture Overview](../02-architecture/overview.md) - Architettura generale
- [Deployment Topology](../02-architecture/deployment.md#lambda-distribution) - Deploy Lambda Multi-AZ
- [Security Architecture](../02-architecture/security.md) - Authentication, IAM, encryption

### AWS Documentation
- [API Gateway Developer Guide](https://docs.aws.amazon.com/apigateway/latest/developerguide/)
- [Lambda Developer Guide](https://docs.aws.amazon.com/lambda/latest/dg/)
- [Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- [Lambda VPC Networking](https://docs.aws.amazon.com/lambda/latest/dg/configuration-vpc.html)
- [Lambda Layers](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html)
- [API Gateway Request Validation](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-method-request-validation.html)

### Community Resources
- [Serverless Framework](https://www.serverless.com/framework/docs)
- [AWS SAM](https://docs.aws.amazon.com/serverless-application-model/)
- [CDK Patterns - API Gateway + Lambda](https://cdkpatterns.com/)
- [AWS Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning)

### Tools
- [LocalStack](https://localstack.cloud/) - Local AWS testing
- [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html) - Local Lambda testing
- [Artillery](https://www.artillery.io/) - Load testing
- [Lumigo](https://lumigo.io/) - Serverless monitoring

---

**Versione**: 1.0
**Ultimo aggiornamento**: 2025-11-18
**Maintainer**: Tech Lead
**Target Audience**: Middle-Senior Developers
