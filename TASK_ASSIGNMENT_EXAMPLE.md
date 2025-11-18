# Task Assignment Example

Questo è un esempio di come assegnare un task specifico da `LEARNING_TASKS.md` a un AI agent.

---

## Esempio 1: Task Prioritario (RAG Implementation)

```markdown
# Task Assignment: RAG Implementation Guide

**Task ID**: TASK-2
**Task Name**: Retrieval-Augmented Generation (RAG) Deep Dive
**Priority**: HIGH
**Estimated Effort**: 4-6 hours
**Assignee**: AI Agent / Developer Name

## Obiettivo

Creare una guida completa sull'implementazione di RAG (Retrieval-Augmented Generation) per production, seguendo le specifiche dettagliate in LEARNING_TASKS.md - TASK 2.

## Istruzioni Dettagliate

1. **Leggi attentamente**:
   - `LEARNING_TASKS.md` - Sezione TASK 2
   - `docs/04-data-flows/ticket-processing.md` - Sezione RAG retrieval
   - `docs/13-prompt-templates.md` - Template RAG generation
   - `docs/06-data-models.md` - KBChunk schema

2. **File di Output**:
   - Path: `docs/07-learning/rag-implementation.md`
   - Formato: Markdown con syntax highlighting
   - Struttura: Seguire il template standard (Overview, Concetti, Implementazione, Best Practices, etc.)

3. **Esempi di Codice Richiesti** (TUTTI obbligatori):
   - ✅ Document Chunking con overlap
   - ✅ Embedding Generation batch con Bedrock
   - ✅ Hybrid Search (keyword + semantic) con OpenSearch
   - ✅ Re-ranking con cross-encoder
   - ✅ Context Window Management
   - ✅ Groundedness Verification algoritmo

4. **Approfondimenti Specifici da Coprire**:
   - Chunk size optimization (trade-off precision/recall)
   - Embedding dimensionality (768 vs 1536)
   - Similarity metrics (cosine, dot product, euclidean)
   - Metadata filtering strategies
   - Caching embeddings
   - Multi-hop reasoning

## Contesto del Progetto

Il nostro sistema usa RAG per generare risposte tecniche verificabili:
- **Input**: Ticket con problema tecnico (es. "Errore E029 su EV Charger")
- **Knowledge Base**: 10K+ documenti tecnici (manuali, bulletins, FAQ)
- **Output**: Soluzione step-by-step con citazioni verificabili

**Performance Target**:
- Retrieval: < 800ms (p95)
- Groundedness score: > 0.85
- Citation coverage: > 0.90

## Criteri di Accettazione

- [ ] Struttura del documento completa (tutte le sezioni)
- [ ] Tutti i 6 esempi di codice implementati e funzionanti
- [ ] Codice Python con type hints e commenti
- [ ] CloudFormation/CDK per configurazione OpenSearch
- [ ] Sezione troubleshooting con ≥5 problemi comuni
- [ ] Collegamenti a ≥3 file esistenti del progetto
- [ ] Diagrammi Mermaid (almeno 1 per architettura RAG)
- [ ] Best practices con esempi Do's/Don'ts
- [ ] Markdown ben formattato (headers, code blocks, tabelle)

## Esempi di Qualità Attesa

### Esempio: Document Chunking

```python
from typing import List, Dict
import tiktoken

class DocumentChunker:
    """
    Chunker semantico con overlap per preservare contesto.

    Strategia: Split su paragrafi, poi merge fino a target size,
    con overlap di N token per evitare perdita di contesto ai bordi.
    """

    def __init__(
        self,
        chunk_size: int = 1000,
        overlap: int = 200,
        encoding_name: str = "cl100k_base"
    ):
        self.chunk_size = chunk_size
        self.overlap = overlap
        self.encoder = tiktoken.get_encoding(encoding_name)

    def chunk_document(
        self,
        text: str,
        metadata: Dict
    ) -> List[Dict]:
        """
        Chunka documento preservando struttura semantica.

        Args:
            text: Testo completo del documento
            metadata: Metadata da propagare ai chunks

        Returns:
            Lista di chunks con text, metadata, e token count
        """
        # Implementation...
        pass
```

**Nota**: Questo è solo un esempio di snippet. La guida deve avere implementazioni complete e funzionanti.

## Deadline

**Target**: [Inserire data se applicabile, es. 2025-11-25]

## Domande / Chiarimenti

Se hai dubbi su:
- Scope: Contatta Tech Lead
- Dettagli tecnici: Vedi file di riferimento
- Esempi poco chiari: Chiedi chiarimenti prima di iniziare

## Review Process

1. Self-review checklist completata
2. Codice testato localmente
3. PR creata con link a LEARNING_TASKS.md
4. Review da Tech Lead o Senior Engineer
5. Merge dopo approvazione

---

**Created**: 2025-11-18
**Last Updated**: 2025-11-18
**Status**: 🔴 Not Started
```

---

## Esempio 2: Task Infrastruttura (Step Functions)

```markdown
# Task Assignment: Step Functions Orchestration Guide

**Task ID**: TASK-1
**Task Name**: Step Functions Orchestration Patterns
**Priority**: HIGH
**Assignee**: AI Agent / Developer Name

## Obiettivo Rapido

Creare guida pratica su orchestrazione workflow con Step Functions, con focus sul nostro ticket processing pipeline.

## File Output

`docs/07-learning/step-functions-orchestration.md`

## Riferimenti Critici

- `LEARNING_TASKS.md` - TASK 1
- `docs/04-data-flows/ticket-processing.md` - Il nostro workflow attuale
- AWS Step Functions Developer Guide

## Must-Have

1. **5 Pattern Examples**:
   - Sequential (il nostro ticket flow)
   - Parallel (RAG da multiple sources)
   - Map State (batch documents)
   - Choice State (routing by confidence)
   - Error Handling (retry + fallback)

2. **CloudFormation Template**:
   - State machine completa per ticket processing
   - IAM roles
   - Error handling configuration
   - CloudWatch alarms

3. **Testing & Debugging**:
   - Come testare state machine localmente
   - Step Functions Local setup
   - Debug con X-Ray
   - Cost optimization tips

## Criteri Successo

- [ ] 5 esempi di pattern implementati
- [ ] CloudFormation deployabile
- [ ] Sezione debugging completa
- [ ] Cost analysis (execution count vs state transitions)

**Deadline**: [Data]
**Status**: 🟡 In Progress
```

---

## Esempio 3: Task Security (per esperto)

```markdown
# Task Assignment: Security Deep Dive

**Task ID**: TASK-8
**Task Name**: Security Deep Dive - IAM, Encryption, Secrets
**Priority**: MEDIUM
**Assignee**: Security Engineer / Senior Developer
**Expertise Required**: AWS Security best practices

## Obiettivo

Guida completa su security implementation nel progetto, con focus su:
- IAM least privilege
- KMS encryption
- Secrets management
- VPC security

## Output

`docs/07-learning/security-deep-dive.md`

## Special Requirements

Questa guida richiede:
1. **Security expertise**: Familiarità con AWS security services
2. **Compliance knowledge**: GDPR considerations
3. **Threat modeling**: Identificare attack vectors

## Esempi Critici

1. **IAM Policy Generator**: Script per creare least-privilege policies
2. **Secrets Rotation**: Lambda completa per rotation
3. **KMS Key Management**: Setup multi-region con grants
4. **Security Audit Script**: Automated security checks

## Security Checklist da Includere

- [ ] IAM roles review process
- [ ] Encryption at rest verification
- [ ] Secrets rotation status
- [ ] VPC security group audit
- [ ] CloudTrail log analysis
- [ ] GuardDuty findings review

## Threat Scenarios da Coprire

1. Compromised API credentials
2. Lambda function privilege escalation
3. S3 bucket public exposure
4. DynamoDB data exfiltration
5. KMS key deletion

**Deadline**: [Data]
**Reviewer**: Security Team Lead
**Status**: 🔴 Not Started
```

---

## Template Vuoto per Nuovi Assignment

```markdown
# Task Assignment: [Nome Task]

**Task ID**: TASK-[numero]
**Task Name**: [Nome completo]
**Priority**: [High/Medium/Low]
**Assignee**: [Nome o "AI Agent"]

## Obiettivo

[1-2 frasi che descrivono l'obiettivo della guida]

## File Output

`docs/07-learning/[filename].md`

## Riferimenti

- `LEARNING_TASKS.md` - TASK [numero]
- [Altri file rilevanti]

## Esempi Richiesti

1. [Esempio 1]
2. [Esempio 2]
3. [Esempio 3]

## Criteri di Successo

- [ ] [Criterio 1]
- [ ] [Criterio 2]
- [ ] [Criterio 3]

**Deadline**: [Data]
**Status**: 🔴 Not Started
```

---

## Tips per Assignment Efficaci

### ✅ Do's

1. **Sii Specifico**: "Implementa RAG con re-ranking" non "Spiega RAG"
2. **Fornisci Contesto**: Link ai file di progetto rilevanti
3. **Definisci Output**: Path esatto, formato, struttura
4. **Criteri Chiari**: Checklist verificabile
5. **Esempi Concreti**: "5 esempi di codice funzionanti"

### ❌ Don'ts

1. ~~Istruzioni vaghe~~: "Scrivi qualcosa su Step Functions"
2. ~~Scope illimitato~~: "Tutto su AWS Security"
3. ~~Nessun riferimento~~: Senza link a documentazione esistente
4. ~~Output non definito~~: "Metti dove ti pare"
5. ~~Criteri soggettivi~~: "Deve essere bello"

---

## Tracking Multiple Assignments

Se assegni task in parallelo a più AI agents:

```markdown
# Parallel Task Assignment - Batch 1

**Date**: 2025-11-18
**Coordinator**: Tech Lead

| Task ID | Topic | Assignee | Priority | ETA | Status |
|---------|-------|----------|----------|-----|--------|
| TASK-2 | RAG Implementation | AI-Agent-1 | HIGH | 2025-11-20 | 🟡 In Progress |
| TASK-3 | Vector Search | AI-Agent-2 | HIGH | 2025-11-20 | 🟡 In Progress |
| TASK-4 | Bedrock Integration | AI-Agent-3 | HIGH | 2025-11-21 | 🔴 Not Started |
| TASK-1 | Step Functions | Developer-A | MEDIUM | 2025-11-22 | 🔴 Not Started |

**Notes**:
- AI-Agent-1,2,3 running in parallel (no dependencies)
- TASK-1 può iniziare mentre altri in corso
- Review meeting: 2025-11-23
```

---

## Review Checklist per Completed Tasks

Quando un task è completato, verifica:

- [ ] **File creato nel path corretto**
- [ ] **Struttura template seguita**
- [ ] **Tutti gli esempi presenti e funzionanti**
- [ ] **Codice con syntax highlighting**
- [ ] **Collegamenti interni funzionanti**
- [ ] **Markdown ben formattato**
- [ ] **No typos / grammatica corretta**
- [ ] **Immagini/diagrammi se necessari**
- [ ] **Sezione troubleshooting completa**
- [ ] **Riferimenti esterni validi**

**Approval Process**:
1. Self-review ✅
2. Automated checks (markdown lint) ✅
3. Tech review ✅
4. Merge to main ✅

---

**Questo documento è un template**. Copia e adatta per ogni assignment specifico.
