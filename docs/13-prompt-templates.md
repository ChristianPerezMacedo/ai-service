# Prompt Engineering Templates

## Overview

Questa sezione documenta i template di prompt utilizzati per l'interazione con i modelli LLM. I prompt sono ottimizzati per massimizzare accuracy, groundedness e citation quality.

## Template Classificazione

### Versione: v2.1

```markdown
System: Sei un assistente tecnico specializzato in sistemi HVAC, inverter e stazioni di ricarica per veicoli elettrici.

Task: Classifica il seguente ticket tecnico in UNA delle categorie predefinite.

# Input

Prodotto: {product_type} {model}
Serial Number: {serial}
Codice Errore: {error_code}
Descrizione Problema:
{symptom_text}

Allegati OCR:
{extracted_text_from_attachments}

# Categorie Disponibili

{categories_list}

# Output Richiesto

Rispondi SOLO in formato JSON con questa struttura:
{
  "category": "CATEGORIA_ESATTA",
  "confidence": 0.XX,
  "reasoning": "breve spiegazione (max 100 caratteri)",
  "top_3": [
    {"category": "...", "confidence": 0.XX},
    {"category": "...", "confidence": 0.XX},
    {"category": "...", "confidence": 0.XX}
  ]
}

# Regole

1. Usa SOLO le categorie fornite nella lista
2. Confidence deve essere tra 0.0 e 1.0
3. Se confidence < 0.7, considera "UNKNOWN"
4. Considera sia il codice errore che la descrizione
5. Allegati OCR sono contesto supplementare
```

**Parametri**:
- `temperature`: 0.1 (bassa variabilità)
- `max_tokens`: 200
- `top_p`: 0.9

**Esempi**:

<details>
<summary>Esempio 1: Electrical Fault</summary>

```json
Input:
{
  "product_type": "EV_CHARGER",
  "model": "XC-200",
  "error_code": "E029",
  "symptom_text": "Caricabatterie si spegne dopo 5 minuti. Errore E029 sul display."
}

Output:
{
  "category": "ELECTRICAL_FAULT_PHASE_IMBALANCE",
  "confidence": 0.92,
  "reasoning": "E029 indica sbilanciamento fasi, sintomo coerente",
  "top_3": [
    {"category": "ELECTRICAL_FAULT_PHASE_IMBALANCE", "confidence": 0.92},
    {"category": "ELECTRICAL_FAULT_OVERCURRENT", "confidence": 0.65},
    {"category": "HARDWARE_FAULT_POWER_BOARD", "confidence": 0.45}
  ]
}
```

</details>

---

## Template RAG Generation

### Versione: v3.2

```markdown
System: Sei un tecnico esperto che genera soluzioni VERIFICABILI per problemi tecnici.
Devi rispondere ESCLUSIVAMENTE basandoti sui documenti forniti. NON inventare informazioni.

# Contesto Knowledge Base

{context_chunks}

# Problema del Cliente

**Prodotto**: {product_type} {model} (S/N: {serial})
**Firmware**: {firmware_version}
**Codice Errore**: {error_code}

**Descrizione Problema**:
{symptom_text}

**Categoria Predetta**: {predicted_category} (confidence: {confidence})

# Task

Genera una soluzione tecnica strutturata in 3 sezioni:

1. **Causa Probabile**: Identifica la causa più probabile basandoti sui documenti
2. **Verifiche Richieste**: Elenca i passi diagnostici da seguire
3. **Soluzione**: Fornisci la procedura di risoluzione passo-passo

# Vincoli CRITICI

1. ✅ CITA SEMPRE la fonte per ogni affermazione (usa [1], [2], ecc.)
2. ✅ Se i documenti non contengono info, scrivi "Informazione non disponibile nella documentazione"
3. ✅ Usa linguaggio tecnico ma chiaro
4. ✅ Numera i passi delle procedure
5. ✅ Indica strumenti e parti necessarie
6. ❌ NON speculare su cause non documentate
7. ❌ NON fornire consigli medici o di sicurezza non verificati
8. ❌ NON menzionare prodotti concorrenti

# Output Formato JSON

{
  "steps": [
    {
      "order": 1,
      "title": "Causa Probabile",
      "text": "...[1] ...[2]",
      "severity": "HIGH|MEDIUM|LOW",
      "estimated_time_minutes": 5
    },
    {
      "order": 2,
      "title": "Verifiche Richieste",
      "text": "1. ...[1]\n2. ...[3]\n3. ...",
      "requires_tools": ["Multimetro", "Cacciavite Torx T20"],
      "severity": "MEDIUM"
    },
    {
      "order": 3,
      "title": "Soluzione",
      "text": "...[2] ...[4]",
      "requires_parts": [
        {"part_number": "PWR-123", "description": "Scheda potenza", "quantity": 1}
      ],
      "severity": "HIGH",
      "estimated_time_minutes": 30
    }
  ],
  "citations": [
    {
      "id": 1,
      "source_uri": "{source_uri_from_chunk}",
      "source_title": "...",
      "snippet": "...",
      "page": 42
    }
  ],
  "safety_flags": [],
  "alternative_actions": [
    "Se il cliente non può eseguire le verifiche: programmare intervento on-site"
  ]
}

# Lingua

Rispondi in: {language}
```

**Parametri**:
- `temperature`: 0.3 (creatività controllata)
- `max_tokens`: 2000
- `top_p`: 0.95

**Esempi**:

<details>
<summary>Esempio 2: EV Charger Fault</summary>

```json
Context Chunks:
[
  {
    "source_uri": "s3://kb/manuals/XC-200.pdf#page=42",
    "text": "Errore E029 indica sbilanciamento corrente fasi > 20%. Verificare serraggio morsetti L1-L2-L3 con coppia 2.5 Nm. Se problema persiste, aggiornare firmware a v1.2.3 che corregge bug calibrazione sensore."
  }
]

Output:
{
  "steps": [
    {
      "order": 1,
      "title": "Causa Probabile",
      "text": "Errore E029 indica uno sbilanciamento di corrente sulle fasi L1-L2-L3 superiore al 20% [1]. Le cause più comuni sono morsetti allentati o un bug nel firmware pre-v1.2.3 [1].",
      "severity": "HIGH",
      "estimated_time_minutes": 5
    },
    {
      "order": 2,
      "title": "Verifiche Richieste",
      "text": "1. Spegnere il caricabatterie e disconnettere alimentazione\n2. Verificare serraggio morsetti L1, L2, L3 [1]\n3. Verificare versione firmware attuale",
      "requires_tools": ["Chiave dinamometrica", "Multimetro"],
      "severity": "MEDIUM"
    },
    {
      "order": 3,
      "title": "Soluzione",
      "text": "1. Riserrare i morsetti L1-L2-L3 con coppia di 2.5 Nm [1]\n2. Se firmware < v1.2.3: aggiornare a v1.2.3 che corregge bug calibrazione sensore corrente [1]\n3. Riavviare caricabatterie e testare",
      "severity": "HIGH",
      "estimated_time_minutes": 30
    }
  ],
  "citations": [
    {
      "id": 1,
      "source_uri": "s3://kb/manuals/XC-200.pdf#page=42",
      "source_title": "XC-200 Technical Manual",
      "snippet": "Errore E029 indica sbilanciamento corrente fasi > 20%...",
      "page": 42
    }
  ],
  "safety_flags": [],
  "alternative_actions": [
    "Se problema persiste dopo firmware update: sostituire scheda potenza (P/N: PWR-XC200-V2)"
  ]
}
```

</details>

---

## Template Embedding

### Query Embedding

```python
def create_query_embedding(query_text, filters):
    """
    Genera embedding per ricerca semantica
    """
    # Arricchisci query con filtri per migliorare retrieval
    enriched_query = f"""
    Prodotto: {filters.get('product_model', 'N/A')}
    Errore: {filters.get('error_code', 'N/A')}
    Query: {query_text}
    """

    response = bedrock.invoke_model(
        modelId='amazon.titan-embed-text-v1',
        body=json.dumps({
            'inputText': enriched_query
        })
    )

    embedding = json.loads(response['body'])['embedding']
    return embedding  # 768-dim vector
```

### Document Embedding

```python
def create_document_embedding(chunk_text, metadata):
    """
    Genera embedding per documento KB
    """
    # Prefix con metadata per migliorare retrieval
    prefixed_text = f"""
    Tipo: {metadata.get('doc_type')}
    Prodotto: {metadata.get('product_model', 'GENERAL')}
    Sezione: {metadata.get('section', '')}

    {chunk_text}
    """

    response = bedrock.invoke_model(
        modelId='amazon.titan-embed-text-v1',
        body=json.dumps({
            'inputText': prefixed_text
        })
    )

    embedding = json.loads(response['body'])['embedding']
    return embedding
```

---

## Template Summarization (Future)

### Email Summary

```markdown
System: Genera un riassunto conciso di un ticket tecnico per email al cliente.

# Ticket

{ticket_metadata}

# Soluzione Generata

{solution_steps}

# Task

Genera un'email professionale (max 300 parole) che:
1. Conferma la ricezione del ticket
2. Riassume il problema
3. Fornisce next steps chiari
4. Include timeline stimata

Tono: Professionale, rassicurante, tecnico ma accessibile
Lingua: {language}
```

---

## Best Practices

### 1. Prompt Engineering Principles

**Clarity**:
- Istruzioni specifiche e non ambigue
- Esempi concreti quando possibile
- Output format esplicito (JSON schema)

**Context**:
- Fornisci contesto rilevante (product, error code)
- Limita context a informazioni necessarie (max 4K tokens)
- Ordina chunks per relevance

**Constraints**:
- Esplicita cosa NON fare (es. "NON speculare")
- Usa emoji per enfatizzare (✅ ❌)
- Definisci limiti chiari (max tokens, formato)

### 2. Citation Format

**Inline Citations**:
```
Il manuale specifica una coppia di 2.5 Nm [1]. Il firmware v1.2.3 corregge il bug [2].
```

**Citation Block**:
```json
{
  "id": 1,
  "source_uri": "s3://kb/path/file.pdf#page=42",
  "source_title": "XC-200 Manual",
  "snippet": "exact text from source...",
  "page": 42
}
```

### 3. Error Handling

**Low Groundedness**:
```
Se grounding score < 0.75:
  → Aggiungi disclaimer: "La seguente soluzione è basata su informazioni limitate..."
  → Flag per revisione umana
```

**No Relevant Context**:
```
Se top retrieval score < 0.5:
  → Risposta: "Non ho trovato informazioni specifiche per questo problema nella documentazione disponibile. Consiglio di escalare a un tecnico specializzato."
```

**Conflicting Information**:
```
Se documenti contraddittrori:
  → Cita entrambi: "Il manuale v1.0 [1] suggerisce X, ma l'aggiornamento v2.0 [2] raccomanda Y. Utilizzare la procedura più recente [2]."
```

### 4. Language Consistency

**Termini Tecnici**:
- Mantieni termini tecnici in inglese quando standard (es. "firmware", "power board")
- Traduci descrizioni ma non codici errore
- Usa unità di misura locali (Nm, mm, °C per EU)

**Tone**:
- Formale ma accessibile
- Evita gergo eccessivo
- Spiega acronimi alla prima occorrenza

---

## Prompt Versioning

### Changelog

| Version | Date | Changes | Impact |
|---------|------|---------|--------|
| v1.0 | 2024-01-15 | Initial prompts | Baseline |
| v2.0 | 2024-03-20 | Added citation requirements | +15% citation accuracy |
| v2.1 | 2024-05-10 | Improved classification structure | +5% F1 score |
| v3.0 | 2024-08-01 | JSON output format | +20% parsing reliability |
| v3.1 | 2024-10-05 | Added safety constraints | -50% unsafe outputs |
| v3.2 | 2024-11-06 | Multilingual support | Supports 5 languages |

### Testing Protocol

Per ogni nuova versione:
1. **Regression test**: 100 golden examples
2. **A/B test**: 10% traffic per 1 settimana
3. **Metrics validation**: Groundedness, accuracy, latency
4. **Manual review**: 50 random outputs
5. **Gradual rollout**: 10% → 50% → 100%

---

## Monitoring

### Prompt Performance Metrics

```python
metrics = {
    'groundedness_score': 0.87,  # How grounded in context
    'citation_rate': 0.95,       # % responses with citations
    'avg_citations': 3.2,        # Citations per response
    'parsing_success': 0.98,     # Valid JSON output
    'token_efficiency': 850,     # Avg output tokens
    'safety_flag_rate': 0.02     # % flagged responses
}
```

### CloudWatch Dashboard

- **Groundedness trend** (target: > 0.85)
- **Citation coverage** (target: > 0.90)
- **Parsing errors** (target: < 2%)
- **Avg response length** (target: 500-1500 tokens)
- **Safety flags** (target: < 5%)

---

## Riferimenti

- [Bedrock Service](./03-aws-services/bedrock.md)
- [Data Flows](./04-data-flows/ticket-processing.md)
- [API Specification](./05-api-specification.md)
