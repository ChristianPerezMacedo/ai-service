# Data Models

## TypeScript Interfaces

### Ticket

```typescript
interface Ticket {
  // Primary key
  ticket_id: string;                    // Format: tkt_<uuid>

  // Timestamps
  created_at: string;                   // ISO8601
  updated_at: string;                   // ISO8601
  completed_at?: string;                // ISO8601

  // Customer info
  customer: {
    id: string;
    name: string;
    contact_email?: string;
    account_tier?: 'FREE' | 'STANDARD' | 'ENTERPRISE';
  };

  // Asset info
  asset: {
    product_type: 'INVERTER' | 'EV_CHARGER' | 'HVAC';
    model: string;
    serial: string;
    firmware_version?: string;
    installation_date?: string;
  };

  // Problem description
  symptom_text: string;                 // Min 10, max 5000 chars
  error_code?: string;                  // Format: E\d{3,4}
  attachments?: Attachment[];

  // Classification
  category_predicted?: string;
  subcategory_predicted?: string;
  confidence?: number;                  // 0.0 - 1.0
  classification_model_version?: string;

  // Status
  status: TicketStatus;
  progress?: TicketProgress;

  // Solution
  solution?: Solution;

  // Feedback
  operator_feedback?: Feedback;

  // Routing
  routing?: RoutingInfo;

  // Metadata
  priority: 'P1' | 'P2' | 'P3' | 'P4';
  lang: string;                         // ISO 639-1
  policies: {
    safe_instructions: boolean;
    grounded_only: boolean;
  };
  metadata?: {
    source?: string;
    created_by?: string;
    tags?: string[];
    [key: string]: any;
  };
}

type TicketStatus =
  | 'NEW'           // Just created
  | 'PROCESSING'    // In Step Functions workflow
  | 'READY'         // Solution available
  | 'FAILED'        // Processing failed
  | 'ESCALATED';    // Requires human intervention

interface TicketProgress {
  current_step: string;
  completed_steps: string[];
  percent_complete: number;             // 0-100
  estimated_completion?: string;        // ISO8601
}
```

### Attachment

```typescript
interface Attachment {
  id: string;
  type: 'PDF' | 'IMAGE' | 'LOG' | 'VIDEO';
  uri: string;                          // S3 URI
  filename?: string;
  size_bytes?: number;
  mime_type?: string;
  description?: string;
  extracted_text?: string;              // From Textract
  metadata?: {
    pages?: number;
    ocr_confidence?: number;
    [key: string]: any;
  };
}
```

### Solution

```typescript
interface Solution {
  // Answer structure
  steps: SolutionStep[];
  citations: Citation[];
  safety_flags: string[];
  alternative_actions?: string[];

  // Routing info
  routing: RoutingInfo;

  // Quality metrics
  quality_metrics: {
    groundedness_score: number;         // 0.0 - 1.0
    citation_coverage: number;          // 0.0 - 1.0
    confidence: number;                 // 0.0 - 1.0
  };

  // Performance
  performance: {
    latency_ms: number;
    tokens_input: number;
    tokens_output: number;
  };

  // Metadata
  created_at: string;                   // ISO8601
  model_version: string;
  prompt_template_version: string;
}

interface SolutionStep {
  order: number;
  title: string;
  text: string;
  severity?: 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL';
  estimated_time_minutes?: number;
  requires_tools?: string[];
  requires_parts?: Part[];
}

interface Part {
  part_number: string;
  description: string;
  quantity: number;
  in_stock?: boolean;
}
```

### Citation

```typescript
interface Citation {
  id: string;
  source_uri: string;                   // S3 URI with anchor
  source_title: string;
  source_type: 'MANUAL' | 'BULLETIN' | 'FAQ' | 'SOP';
  snippet: string;
  relevance_score: number;              // 0.0 - 1.0
  page?: number;
  section?: string;
  metadata?: {
    version?: string;
    published_date?: string;
    author?: string;
    [key: string]: any;
  };
}
```

### Feedback

```typescript
interface Feedback {
  // Operator info
  operator_id: string;
  submitted_at: string;                 // ISO8601

  // Ratings
  rating: 1 | 2 | 3 | 4 | 5;            // 1=poor, 5=excellent
  was_helpful: boolean;
  accuracy: 'ACCURATE' | 'PARTIALLY_ACCURATE' | 'INACCURATE';
  completeness: 'COMPLETE' | 'PARTIALLY_COMPLETE' | 'INCOMPLETE';

  // Actions taken
  applied_solution: boolean;
  resolved_issue: boolean;
  required_escalation: boolean;

  // Free text
  comments?: string;
  corrections?: string;                 // What should be corrected

  // Metrics
  time_to_resolution_minutes?: number;

  // Training data
  use_for_training: boolean;            // Consent to use for ML
}
```

### KBChunk

Knowledge base chunk for vector search.

```typescript
interface KBChunk {
  // Primary key
  chunk_id: string;                     // Format: chunk_<uuid>

  // Source
  source_uri: string;                   // S3 URI
  source_title: string;
  doc_type: 'MANUAL' | 'BULLETIN' | 'FAQ' | 'SOP' | 'WIKI';

  // Content
  text: string;                         // 512-1500 tokens
  text_hash: string;                    // SHA256 for deduplication

  // Vector
  vector: number[];                     // 768-dim (Titan embeddings)
  embedding_model: string;              // Model used for embedding

  // Filters for hybrid search
  product_model?: string[];
  error_codes?: string[];
  categories?: string[];
  tags?: string[];

  // Metadata
  metadata: {
    version: string;
    language: string;                   // ISO 639-1
    section?: string;
    page?: number;
    updated_at: string;                 // ISO8601
    author?: string;
    review_status?: 'DRAFT' | 'REVIEWED' | 'PUBLISHED';
    [key: string]: any;
  };

  // Timestamps
  indexed_at: string;                   // ISO8601
  last_accessed?: string;
}
```

### RoutingInfo

```typescript
interface RoutingInfo {
  // Model selection
  capability: 'classification' | 'tech_troubleshoot' | 'general_support';
  provider: 'bedrock' | 'sagemaker' | 'openai';
  model: string;
  model_version: string;

  // Fallback
  fallback_used: boolean;
  fallback_reason?: string;

  // Costs (optional)
  cost_usd?: number;
}
```

## DynamoDB Table Schemas

### tickets Table

```yaml
TableName: prod-tickets
BillingMode: ON_DEMAND
PointInTimeRecovery: Enabled
StreamSpecification: NEW_AND_OLD_IMAGES

KeySchema:
  - AttributeName: ticket_id
    KeyType: HASH

GlobalSecondaryIndexes:
  - IndexName: customer-index
    KeySchema:
      - AttributeName: customer_id
        KeyType: HASH
      - AttributeName: created_at
        KeyType: RANGE
    Projection: ALL

  - IndexName: status-index
    KeySchema:
      - AttributeName: status
        KeyType: HASH
      - AttributeName: created_at
        KeyType: RANGE
    Projection: ALL

Attributes:
  ticket_id: S
  customer_id: S
  status: S
  created_at: S
  # ... (all other fields as JSON document)

TTL:
  Enabled: false  # Keep all tickets for audit
```

### idempotency Table

```yaml
TableName: prod-idempotency
BillingMode: PROVISIONED
ReadCapacityUnits: 25
WriteCapacityUnits: 25

KeySchema:
  - AttributeName: idempotency_key
    KeyType: HASH

Attributes:
  idempotency_key: S    # Format: {endpoint}#{request_hash}
  response: S           # Cached response
  status: S
  expiry_time: N

TTL:
  Enabled: true
  AttributeName: expiry_time
  # Auto-delete after 24 hours
```

### feedback Table

```yaml
TableName: prod-feedback
BillingMode: ON_DEMAND

KeySchema:
  - AttributeName: ticket_id
    KeyType: HASH
  - AttributeName: submitted_at
    KeyType: RANGE

GlobalSecondaryIndexes:
  - IndexName: operator-index
    KeySchema:
      - AttributeName: operator_id
        KeyType: HASH
      - AttributeName: submitted_at
        KeyType: RANGE
    Projection: ALL

Attributes:
  ticket_id: S
  submitted_at: S
  operator_id: S
  # ... (feedback fields)
```

## OpenSearch Index Schema

### kb-chunks Index

```json
{
  "settings": {
    "index": {
      "number_of_shards": 5,
      "number_of_replicas": 1,
      "knn": true,
      "knn.algo_param.ef_search": 512
    },
    "analysis": {
      "analyzer": {
        "italian_technical": {
          "type": "custom",
          "tokenizer": "standard",
          "filter": ["lowercase", "italian_stop", "italian_stemmer"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "chunk_id": { "type": "keyword" },
      "source_uri": { "type": "keyword" },
      "source_title": { "type": "text", "analyzer": "italian_technical" },
      "doc_type": { "type": "keyword" },
      "text": { "type": "text", "analyzer": "italian_technical" },
      "text_hash": { "type": "keyword" },
      "vector": {
        "type": "knn_vector",
        "dimension": 768,
        "method": {
          "name": "hnsw",
          "engine": "nmslib",
          "parameters": {
            "ef_construction": 512,
            "m": 16
          }
        }
      },
      "product_model": { "type": "keyword" },
      "error_codes": { "type": "keyword" },
      "categories": { "type": "keyword" },
      "tags": { "type": "keyword" },
      "metadata": {
        "properties": {
          "version": { "type": "keyword" },
          "language": { "type": "keyword" },
          "section": { "type": "text" },
          "page": { "type": "integer" },
          "updated_at": { "type": "date" }
        }
      },
      "indexed_at": { "type": "date" }
    }
  }
}
```

## S3 Bucket Structure

### raw-data Bucket

```
s3://prod-raw-data-eus1/
├── uploads/                    # User uploads
│   ├── {ticket_id}/
│   │   ├── report.pdf
│   │   └── photo.jpg
├── kb-sources/                 # KB source documents
│   ├── manuals/
│   │   ├── XC-200-manual-v1.2.pdf
│   │   └── ...
│   ├── bulletins/
│   └── faqs/
└── logs/                       # System logs
    └── {date}/
```

### processed-data Bucket

```
s3://prod-processed-data-eus1/
├── textract-output/
│   ├── {job_id}/
│   │   ├── response.json
│   │   └── tables.csv
├── embeddings/
│   ├── {chunk_id}.npy
└── kb-chunks/
    ├── {doc_id}/
    │   ├── chunk-001.json
    │   └── chunk-002.json
```

### ml-models Bucket

```
s3://prod-ml-models-eus1/
├── classifiers/
│   ├── v1.0.0/
│   │   ├── model.tar.gz
│   │   └── metadata.json
│   └── v2.0.0/
├── embeddings/
└── experiments/
```

## Validation Rules

### Ticket Creation

- `symptom_text`: 10-5000 characters
- `error_code`: Optional, regex `^[A-Z]\d{3,4}$`
- `priority`: One of P1, P2, P3, P4
- `lang`: Valid ISO 639-1 code
- `customer.id`: Required, non-empty
- `asset.product_type`: Required, valid enum

### Solution Quality

- `groundedness_score`: Must be >= 0.75
- `citations`: At least 1 citation required
- `safety_flags`: Empty for auto-publish

## Riferimenti

- [API Specification](./05-api-specification.md)
- [DynamoDB Design](./03-aws-services/dynamodb.md)
- [OpenSearch Configuration](./03-aws-services/opensearch.md)
