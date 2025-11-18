# API Specification v1

## Base URL

```
Production: https://api.example.com/v1
Staging:    https://api-staging.example.com/v1
```

## Authentication

Tutte le API richiedono autenticazione via **JWT Bearer Token** ottenuto da Cognito.

```http
Authorization: Bearer <jwt_token>
```

## Endpoint Principali

| Metodo | Endpoint | Descrizione | Auth | Response |
|--------|----------|-------------|------|----------|
| **POST** | `/tickets` | Crea nuovo ticket | Required | `202 Accepted` |
| **GET** | `/tickets/{id}` | Stato e metadata ticket | Required | `200 OK` |
| **GET** | `/tickets/{id}/solution` | Soluzione generata | Required | `200 OK` |
| **GET** | `/tickets/{id}/solution/stream` | SSE streaming | Required | `Event Stream` |
| **POST** | `/tickets/{id}/feedback` | Feedback operatore | Required | `204 No Content` |
| **GET** | `/tickets` | Lista ticket (paginata) | Required | `200 OK` |
| **POST** | `/kb/search` | Ricerca knowledge base | Required | `200 OK` |
| **POST** | `/kb/documents` | Upload documento KB | Admin | `202 Accepted` |
| **GET** | `/health` | Health check | None | `200 OK` |

---

## POST /tickets

Crea un nuovo ticket e avvia la pipeline di processing asincrona.

### Request

```http
POST /v1/tickets HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbG...
Content-Type: application/json
```

```json
{
  "customer": {
    "id": "C123",
    "name": "ACME Corp",
    "contact_email": "support@acme.com"
  },
  "asset": {
    "product_type": "EV_CHARGER",
    "model": "XC-200",
    "serial": "SN123456",
    "firmware_version": "v1.2.0"
  },
  "symptom_text": "Errore E029 durante ricarica trifase. Il caricabatterie si spegne dopo 5 minuti.",
  "error_code": "E029",
  "attachments": [
    {
      "id": "att-001",
      "type": "PDF",
      "uri": "s3://bucket/uploads/report.pdf",
      "description": "Log di sistema"
    }
  ],
  "priority": "P2",
  "lang": "it-IT",
  "policies": {
    "safe_instructions": true,
    "grounded_only": true
  },
  "metadata": {
    "source": "ITSM",
    "created_by": "operator-123"
  }
}
```

### Response (Success)

```http
HTTP/1.1 202 Accepted
Content-Type: application/json
Location: /v1/tickets/tkt_67890
```

```json
{
  "ticket_id": "tkt_67890",
  "status": "PROCESSING",
  "created_at": "2025-11-18T10:30:00Z",
  "estimated_completion": "2025-11-18T10:32:00Z",
  "status_url": "/v1/tickets/tkt_67890"
}
```

### Response (Validation Error)

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json
```

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request payload",
    "details": [
      {
        "field": "symptom_text",
        "message": "Must be at least 10 characters"
      }
    ]
  }
}
```

---

## GET /tickets/{id}

Recupera lo stato corrente e i metadata di un ticket.

### Request

```http
GET /v1/tickets/tkt_67890 HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbG...
```

### Response (Processing)

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "ticket_id": "tkt_67890",
  "status": "PROCESSING",
  "created_at": "2025-11-18T10:30:00Z",
  "updated_at": "2025-11-18T10:30:45Z",
  "customer": {
    "id": "C123",
    "name": "ACME Corp"
  },
  "asset": {
    "product_type": "EV_CHARGER",
    "model": "XC-200",
    "serial": "SN123456"
  },
  "symptom_text": "Errore E029 durante ricarica trifase...",
  "classification": {
    "category": "ELECTRICAL_FAULT",
    "subcategory": "PHASE_IMBALANCE",
    "confidence": 0.92,
    "model_version": "v2.3.1"
  },
  "progress": {
    "current_step": "generating_solution",
    "completed_steps": ["validate", "classify", "retrieve_kb"],
    "percent_complete": 75
  }
}
```

### Response (Completed)

```json
{
  "ticket_id": "tkt_67890",
  "status": "READY",
  "created_at": "2025-11-18T10:30:00Z",
  "updated_at": "2025-11-18T10:32:15Z",
  "completed_at": "2025-11-18T10:32:15Z",
  "customer": { "id": "C123", "name": "ACME Corp" },
  "asset": { "product_type": "EV_CHARGER", "model": "XC-200", "serial": "SN123456" },
  "symptom_text": "Errore E029 durante ricarica trifase...",
  "classification": {
    "category": "ELECTRICAL_FAULT",
    "subcategory": "PHASE_IMBALANCE",
    "confidence": 0.92
  },
  "solution_available": true,
  "solution_url": "/v1/tickets/tkt_67890/solution"
}
```

---

## GET /tickets/{id}/solution

Recupera la soluzione generata per il ticket.

### Request

```http
GET /v1/tickets/tkt_67890/solution HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbG...
```

### Response

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "ticket_id": "tkt_67890",
  "status": "READY",
  "answer": {
    "steps": [
      {
        "order": 1,
        "title": "Causa Probabile",
        "text": "Errore E029 indica uno sbilanciamento di corrente sulle fasi L1-L2-L3 superiore al 20%. Questo può essere causato da:\n- Morsetti allentati\n- Cavi danneggiati\n- Guasto interno alla scheda di potenza",
        "severity": "HIGH"
      },
      {
        "order": 2,
        "title": "Verifiche Richieste",
        "text": "1. Controllare serraggio morsetti L1, L2, L3\n2. Misurare tensione sulle tre fasi (deve essere 380-400V)\n3. Verificare corrente su ciascuna fase (differenza < 10%)\n4. Controllare log eventi per guasti precedenti",
        "severity": "MEDIUM"
      },
      {
        "order": 3,
        "title": "Soluzione",
        "text": "- Se morsetti allentati: riserrare con coppia 2.5 Nm\n- Se tensione sbilanciata: contattare fornitore energia\n- Se corrente sbilanciata: aggiornare firmware a v1.2.3 (fix bug calibrazione)\n- Se problema persiste: sostituire scheda potenza (P/N: PWR-XC200-V2)",
        "severity": "HIGH"
      }
    ],
    "citations": [
      {
        "id": "cite-1",
        "source_uri": "s3://kb/manuals/XC-200-technical-manual.pdf#page=42",
        "source_title": "XC-200 Technical Manual",
        "snippet": "...verifica del bilanciamento fasi tramite misura corrente L1-L2-L3...",
        "relevance_score": 0.89,
        "page": 42
      },
      {
        "id": "cite-2",
        "source_uri": "s3://kb/bulletins/TB-2024-015.pdf",
        "source_title": "Technical Bulletin 2024-015",
        "snippet": "...firmware v1.2.3 corregge errore calibrazione sensore corrente...",
        "relevance_score": 0.85,
        "page": 2
      }
    ],
    "safety_flags": [],
    "alternative_actions": [
      "Se cliente non può accedere fisicamente: programmare intervento tecnico on-site",
      "Verificare se asset è in garanzia prima di sostituire componenti"
    ]
  },
  "routing": {
    "capability": "tech_troubleshoot",
    "provider": "bedrock",
    "model": "claude-3-sonnet-20240229",
    "model_version": "1.0",
    "fallback_used": false
  },
  "quality_metrics": {
    "groundedness_score": 0.87,
    "citation_coverage": 0.95,
    "confidence": 0.89
  },
  "performance": {
    "latency_ms": 2140,
    "tokens_input": 1200,
    "tokens_output": 850
  },
  "created_at": "2025-11-18T10:32:15Z"
}
```

---

## GET /tickets/{id}/solution/stream

Streaming della soluzione via Server-Sent Events.

### Request

```http
GET /v1/tickets/tkt_67890/solution/stream HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbG...
Accept: text/event-stream
```

### Response

```http
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```

```
event: start
data: {"ticket_id": "tkt_67890", "status": "streaming"}

event: chunk
data: {"type": "text", "content": "Errore E029 indica uno sbilanciamento"}

event: chunk
data: {"type": "text", "content": " di corrente sulle fasi L1-L2-L3..."}

event: citation
data: {"source_uri": "s3://kb/manuals/XC-200.pdf", "snippet": "...verifica bilanciamento..."}

event: chunk
data: {"type": "text", "content": "Soluzione: riserrare morsetti..."}

event: complete
data: {"status": "completed", "total_tokens": 850, "latency_ms": 2100}
```

---

## POST /tickets/{id}/feedback

Invio feedback dell'operatore sulla soluzione generata.

### Request

```http
POST /v1/tickets/tkt_67890/feedback HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbG...
Content-Type: application/json
```

```json
{
  "operator_id": "op-123",
  "rating": 5,
  "was_helpful": true,
  "accuracy": "ACCURATE",
  "completeness": "COMPLETE",
  "applied_solution": true,
  "resolved_issue": true,
  "comments": "Soluzione precisa e ben documentata. Ha risolto il problema al primo tentativo.",
  "corrections": null,
  "time_to_resolution_minutes": 15
}
```

### Response

```http
HTTP/1.1 204 No Content
```

---

## POST /kb/search

Ricerca diretta nella knowledge base.

### Request

```http
POST /v1/kb/search HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbG...
Content-Type: application/json
```

```json
{
  "query": "Come risolvere errore E029 su caricabatterie XC-200",
  "filters": {
    "product_model": "XC-200",
    "error_code": "E029",
    "doc_type": ["MANUAL", "BULLETIN"]
  },
  "max_results": 5,
  "include_snippets": true
}
```

### Response

```http
HTTP/1.1 200 OK
Content-Type: application/json
```

```json
{
  "query": "Come risolvere errore E029...",
  "results": [
    {
      "id": "chunk-12345",
      "source_uri": "s3://kb/manuals/XC-200.pdf#page=42",
      "source_title": "XC-200 Technical Manual",
      "doc_type": "MANUAL",
      "snippet": "Errore E029 - Sbilanciamento fasi. Verificare tensione L1-L2-L3...",
      "score": 0.89,
      "metadata": {
        "product_model": "XC-200",
        "error_code": "E029",
        "section": "Troubleshooting",
        "page": 42,
        "updated_at": "2024-03-15"
      }
    }
  ],
  "total_results": 12,
  "took_ms": 45
}
```

---

## Error Responses

### Standard Error Format

```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human readable message",
    "details": {},
    "request_id": "req-abc123",
    "timestamp": "2025-11-18T10:30:00Z"
  }
}
```

### Error Codes

| Code | HTTP | Descrizione |
|------|------|-------------|
| `VALIDATION_ERROR` | 400 | Invalid request payload |
| `UNAUTHORIZED` | 401 | Missing or invalid auth token |
| `FORBIDDEN` | 403 | Insufficient permissions |
| `NOT_FOUND` | 404 | Resource not found |
| `RATE_LIMITED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Server error |
| `SERVICE_UNAVAILABLE` | 503 | Temporary unavailability |

---

## Rate Limiting

```
Tier           | Requests/sec | Requests/hour | Burst
---------------|--------------|---------------|-------
Free           | 10           | 1,000         | 20
Standard       | 100          | 10,000        | 200
Enterprise     | 1,000        | 100,000       | 2,000
```

**Headers**:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1637251200
```

---

## Pagination

Liste paginata con cursor-based pagination:

```http
GET /v1/tickets?limit=20&cursor=eyJpZCI6InRrdF82Nzg5MCJ9
```

Response:
```json
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6InRrdF82Nzg5MSJ9",
    "has_more": true,
    "total_count": 150
  }
}
```

---

## Webhooks (Future)

Configurazione webhook per notifiche asincrone:

```http
POST /v1/webhooks
```

```json
{
  "url": "https://customer.com/webhook",
  "events": ["ticket.completed", "ticket.failed"],
  "secret": "whsec_..."
}
```

## Riferimenti

- [Data Models](./06-data-models.md)
- [Authentication](./02-architecture/security.md#authentication)
- [Rate Limiting](./02-architecture/security.md#rate-limiting)
