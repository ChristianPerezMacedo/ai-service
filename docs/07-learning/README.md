# Learning Guides

Questa directory contiene **guide di approfondimento** su argomenti complessi del progetto AI Technical Support System.

## Obiettivo

Fornire documentazione tecnica approfondita per sviluppatori/ingegneri middle-senior che lavorano sul progetto o vogliono comprendere in dettaglio specifici aspetti dell'implementazione.

## Struttura Guide

Ogni guida segue questo formato:
- **Overview**: Introduzione e contesto
- **Concetti Fondamentali**: Teoria e terminologia
- **Implementazione Pratica**: Esempi di codice funzionanti
- **Best Practices**: Do's and Don'ts
- **Troubleshooting**: Problemi comuni e soluzioni
- **Esempi Reali dal Progetto**: Applicazioni concrete

## Guide Disponibili

### Infrastructure & Orchestration
- [ ] `step-functions-orchestration.md` - Orchestrazione workflow con Step Functions
- [ ] `eventbridge-patterns.md` - Event-driven architecture patterns

### AI/ML
- [ ] `rag-implementation.md` - Retrieval-Augmented Generation deep dive
- [ ] `vector-search-opensearch.md` - Vector search con OpenSearch k-NN
- [ ] `bedrock-integration-patterns.md` - Bedrock LLM integration patterns
- [ ] `sagemaker-mlops.md` - SageMaker MLOps pipeline

### Data & APIs
- [ ] `dynamodb-data-modeling.md` - Advanced DynamoDB patterns
- [ ] `api-gateway-lambda-patterns.md` - Serverless API patterns

### Operations
- [ ] `observability-guide.md` - CloudWatch, X-Ray, monitoring
- [ ] `cost-optimization.md` - AWS cost optimization strategies
- [ ] `disaster-recovery-ha.md` - DR and high availability
- [ ] `cicd-pipeline.md` - CI/CD best practices

### Security
- [ ] `security-deep-dive.md` - IAM, encryption, secrets management

### Testing
- [ ] `testing-strategies.md` - Unit, integration, load testing

## Come Contribuire

1. Vedi i task disponibili in `/LEARNING_TASKS.md`
2. Scegli un task o viene assegnato
3. Segui il template specificato nel task
4. Crea il file nella directory corretta
5. Assicurati che il codice sia funzionante
6. Collega la guida ai file esistenti del progetto
7. Richiedi review

## Target Audience

**Livello**: Middle-Senior Developer/Engineer

**Background atteso**:
- Familiarità con AWS fundamentals
- Esperienza con Python/TypeScript
- Conoscenza di base di ML/AI concepts
- Comprensione di architetture serverless

**Non richiesto**:
- Deep expertise in ogni servizio AWS
- Background accademico in ML
- Anni di esperienza specifica

## Stile e Convenzioni

### Codice
- Python: PEP 8, type hints
- TypeScript: ESLint + Prettier
- CloudFormation/CDK: Commenti inline
- Shell: ShellCheck compliant

### Markdown
- Headers con emoji (quando appropriato)
- Code blocks con syntax highlighting
- Tabelle per comparazioni
- Mermaid diagrams quando utili
- Link interni relativi

### Esempi
- Self-contained (eseguibili)
- Commentati abbondantemente
- Coprono happy path + edge cases
- Include error handling

## Risorse Esterne

### AWS Documentation
- [Step Functions Developer Guide](https://docs.aws.amazon.com/step-functions/)
- [Bedrock User Guide](https://docs.aws.amazon.com/bedrock/)
- [OpenSearch Service Guide](https://docs.aws.amazon.com/opensearch-service/)
- [DynamoDB Developer Guide](https://docs.aws.amazon.com/dynamodb/)

### Community Resources
- [AWS Samples GitHub](https://github.com/aws-samples)
- [AWS Architecture Blog](https://aws.amazon.com/blogs/architecture/)
- [re:Post](https://repost.aws/)

### Tools
- [LocalStack](https://localstack.cloud/) - Local AWS testing
- [moto](https://github.com/getmoto/moto) - Mock AWS services
- [CDK Patterns](https://cdkpatterns.com/) - Reusable CDK patterns

## FAQ

**Q: Quanto deve essere lunga una guida?**
A: 2000-5000 parole tipicamente. Prioritizza chiarezza su lunghezza.

**Q: Devo includere tutte le feature di un servizio?**
A: No, focus su quello che usiamo nel progetto + best practices essenziali.

**Q: Posso usare esempi esterni al progetto?**
A: Sì, ma collega sempre al nostro use case specifico.

**Q: Come gestisco breaking changes AWS?**
A: Aggiungi data di ultimo update e nota su deprecations.

**Q: Serve approvazione per nuove guide?**
A: Sì, crea PR e richiedi review da Tech Lead.

---

**Maintainer**: Tech Lead
**Last Updated**: 2025-11-18
**Status**: Active Development
