# Overview del Progetto

## Executive Summary

**Obiettivo**: Automatizzare l'assistenza tecnica per sistemi Inverter/EV Charger/HVAC attraverso AI/ML, con analisi ticket, classificazione intelligente, ricerca knowledge base, generazione risposte verificabili e apprendimento continuo.

**Stack Tecnologico**: AWS (Bedrock, SageMaker, Step Functions, OpenSearch) con architettura LLM-agnostica e approccio API-first.

## Descrizione del Progetto

Il sistema di **Assistenza Tecnica AI** è progettato per automatizzare il processo di supporto tecnico per equipaggiamenti industriali (inverter, stazioni di ricarica EV, sistemi HVAC). Il sistema utilizza tecnologie di intelligenza artificiale per:

- **Classificare** automaticamente i ticket in entrata per categoria e urgenza
- **Recuperare** informazioni rilevanti dalla knowledge base aziendale
- **Generare** soluzioni tecniche verificabili basate su documentazione esistente
- **Apprendere** continuamente dal feedback degli operatori
- **Fornire** API per integrazione con sistemi ITSM esistenti

## Valore Aggiunto

### Per il Business
- **Riduzione tempi di risoluzione**: Da ore a minuti per problemi comuni
- **Deflection rate**: 40-60% dei ticket risolti senza escalation a L2
- **Disponibilità 24/7**: Supporto sempre disponibile
- **Scalabilità**: Gestione di picchi senza aumentare headcount

### Per gli Operatori
- **Assistente intelligente**: Suggerimenti basati su casi simili precedenti
- **Knowledge centralizzata**: Accesso rapido a tutta la documentazione
- **Riduzione errori**: Soluzioni verificate e citate dalle fonti ufficiali
- **Focus su casi complessi**: Tempo liberato per problemi ad alto valore

### Per i Clienti
- **Risposte più rapide**: Riduzione SLA da 24h a 2h per problemi comuni
- **Maggiore consistenza**: Risposte standardizzate e verificate
- **Tracciabilità**: Ogni soluzione citata da fonti ufficiali
- **Qualità**: Soluzioni basate su best practices documentate

## Tech Stack

### Cloud Platform
- **AWS** come cloud provider principale
- **Regione**: eu-south-1 (Milano) per compliance GDPR
- **Multi-AZ**: Alta disponibilità su 3 availability zones

### AI/ML Services
- **Amazon Bedrock**: LLM per generazione risposte (Claude, Llama)
- **SageMaker**: Training e deployment modelli classificazione
- **Textract**: OCR per estrazione testo da PDF e immagini
- **Bedrock Embeddings**: Titan per embedding semantici

### Data Services
- **DynamoDB**: Database NoSQL per stato ticket e metadati
- **OpenSearch**: Vector database per knowledge base e ricerca semantica
- **S3**: Object storage per documenti, modelli, logs
- **RDS PostgreSQL**: Database relazionale per analytics

### Orchestration
- **Step Functions**: Orchestrazione workflow asincroni
- **Lambda**: Compute serverless per business logic
- **EventBridge**: Event bus per integrazione asincrona
- **API Gateway**: API management e autenticazione

### Monitoring & Security
- **CloudWatch**: Logging, metriche, allarmi
- **X-Ray**: Distributed tracing
- **Cognito**: User authentication e authorization
- **KMS**: Encryption key management
- **Secrets Manager**: Gestione credenziali

## Pattern Architetturali

Il sistema adotta i seguenti pattern:

1. **API-First**: Tutte le funzionalità esposte via API REST
2. **Event-Driven**: Comunicazione asincrona tra componenti
3. **Microservices**: Servizi specializzati e disaccoppiati
4. **RAG (Retrieval Augmented Generation)**: LLM + Knowledge Base esterna
5. **MLOps**: Pipeline automatizzate per ML lifecycle
6. **Multi-tenant**: Supporto multi-cliente con isolamento dati
7. **Observability**: Monitoring, logging, tracing su tutti i livelli

## Metriche di Successo

### Metriche Tecniche
- **Precision/Recall classificazione**: > 90%
- **Groundedness score**: > 85% (risposta basata su fonti)
- **API latency P95**: < 3 secondi
- **System availability**: > 99.9%

### Metriche Business
- **Resolution rate**: > 50% ticket auto-risolti
- **Time to resolution**: < 2 ore (media)
- **Customer satisfaction**: > 4.5/5
- **Cost per ticket**: Riduzione del 60%

## Roadmap di Sviluppo

### Fase 1: MVP (Sprint 1-2)
- Setup infrastruttura base AWS
- Pipeline Textract → OpenSearch
- Classificatore baseline
- RAG + LLM via Bedrock
- API v1 core endpoints

### Fase 2: Hardening (Sprint 3-4)
- Guardrails e PII redaction
- UI feedback per operatori
- Job retraining automatici
- SSE streaming risposte

### Fase 3: Automation (Sprint 5-6)
- Playbook automation
- Advanced RAG (re-ranking)
- Multi-model routing
- Caching semantico

### Fase 4: Scale & Optimize (Sprint 7+)
- A/B testing framework
- Multi-language support
- Predictive maintenance
- Voice/chat integration

## Riferimenti

- [Architettura](./02-architecture/overview.md)
- [Servizi AWS](./03-aws-services/README.md)
- [API Specification](./05-api-specification.md)
- [Data Models](./06-data-models.md)
- [Security](./10-security-compliance.md)
- [Costi](./11-cost-estimation.md)
