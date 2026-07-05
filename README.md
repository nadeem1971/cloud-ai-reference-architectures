# Cloud AI Reference Architectures

> Architecture patterns for enterprise Agentic AI across GCP, AWS Bedrock, and Azure OpenAI.  
> Built from real production deployments — not theoretical diagrams.

[![GCP](https://img.shields.io/badge/GCP-Vertex_AI-4285F4?logo=googlecloud)](https://cloud.google.com/vertex-ai)
[![AWS](https://img.shields.io/badge/AWS-Bedrock-FF9900?logo=amazonaws)](https://aws.amazon.com/bedrock)
[![Azure](https://img.shields.io/badge/Azure-OpenAI-0078D4?logo=microsoftazure)](https://azure.microsoft.com/en-us/products/ai-services/openai-service)

---

## Architecture 1 — GCP Agentic AI Platform

**Status: Production deployed** · Cloud Run asia-south1 · June 2026  
**Reference implementation:** [retail-agentic-ai-platform](https://github.com/nadeem1971/retail-agentic-ai-platform)

```
┌─────────────────────────────────────────────────────────┐
│                    CLIENT APP                           │
│              Web · Mobile · Chat · In-store             │
└──────────────────────┬──────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────┐
│         CLOUD LOAD BALANCING + CLOUD ARMOR              │
│         WAF · DDoS Protection · SSL Termination         │
└──────────────────────┬──────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────┐
│                  API GATEWAY                            │
│          JWT Auth · Rate Limiting · Routing             │
└──────────────────────┬──────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────┐
│         CLOUD RUN — LANGGRAPH CONTROL PLANE             │
│                                                         │
│  Governor Agent (keyword-first routing)                 │
│      ├── Search Executor → Vertex AI Search             │
│      ├── Transaction Executor → Risk Engine             │
│      ├── RAG Agent → BigQuery policy_chunks             │
│      └── Chat Node → Gemini direct                      │
│                    ↓                                    │
│           Generator (Gemini 2.5 Flash)                  │
│                    ↓                                    │
│         Policy/Watcher (compliance gate)                │
└──────────────────────┬──────────────────────────────────┘
                       ↓
┌──────────────┬───────────────┬──────────────────────────┐
│  Vertex AI   │  Gemini 2.5   │     Model Armor          │
│  Search      │  Flash        │     Content Safety       │
└──────────────┴───────────────┴──────────────────────────┘
                       ↓
┌──────────────┬───────────────┬──────────────────────────┐
│  BigQuery    │  Firestore    │  Cloud DLP               │
│  Analytics   │  State/HITL   │  PII Scanning            │
└──────────────┴───────────────┴──────────────────────────┘
```

**Key design decisions:**

| Decision | Rationale |
|----------|-----------|
| LangGraph over custom orchestration | State machine with conditional edges — easier to debug and extend |
| Keyword-first Governor | Zero LLM cost on 80% of queries — only ambiguous queries hit Gemini |
| Vertex AI Search over custom search | Managed semantic search with personalization, no embedding pipeline needed |
| Cloud DLP on all outputs | PII leakage prevention at the infrastructure layer, not the prompt layer |
| Firestore for HITL queue | Real-time updates, TTL for GDPR, serverless scaling |

---

## Architecture 2 — AWS Bedrock Multi-Agent Platform

**Status: Production delivered** · AWS Lambda + API Gateway · Wipro (2026)

```
┌─────────────────────────────────────────────────────────┐
│                    CLIENT APP                           │
└──────────────────────┬──────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────┐
│              API GATEWAY (REST)                         │
│         JWT Auth · Rate Limiting · WAF                  │
└──────────────────────┬──────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────┐
│              AWS LAMBDA — ORCHESTRATOR                  │
│                                                         │
│  Extraction Agent (Claude 3.5 Sonnet)                  │
│      → Parses unstructured documents                    │
│      → Extracts structured entities                     │
│                                                         │
│  Inference Agent (Claude 3.5 Sonnet)                   │
│      → Applies business rules                           │
│      → Generates compliance assessment                  │
│                                                         │
│  HITL Validator                                         │
│      → Confidence < threshold → human review            │
│      → Audit trail → DynamoDB                          │
└──────────────────────┬──────────────────────────────────┘
                       ↓
┌──────────────┬───────────────┬──────────────────────────┐
│  Bedrock     │  DynamoDB     │  S3                      │
│  Claude 3.5  │  Audit Trail  │  Document Store          │
└──────────────┴───────────────┴──────────────────────────┘
```

**Key design decisions:**

| Decision | Rationale |
|----------|-----------|
| Two specialized agents over one | Separation of concerns — extraction errors don't corrupt inference |
| Lambda over containers | Compliance workloads are bursty — serverless scales to zero when idle |
| DynamoDB for audit | Sub-millisecond writes, TTL for data retention, no schema lock-in |
| Confidence-based HITL | Not all decisions need human review — only those below the confidence threshold |

**Outcomes achieved:**
- 60% reduction in manual compliance processing
- 100% audit traceability — zero regulatory findings

---

## Architecture 3 — Azure OpenAI Enterprise Platform

**Status: Production delivered** · AKS + Azure ML · Ericsson (2022-2024)

```
┌─────────────────────────────────────────────────────────┐
│                    CLIENT APP                           │
└──────────────────────┬──────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────┐
│         AZURE API MANAGEMENT                            │
│       Auth · Rate Limiting · Monitoring                 │
└──────────────────────┬──────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────────┐
│              AKS — ML INFERENCE CLUSTER                 │
│                                                         │
│  RAG Pipeline                                           │
│      → Pinecone vector retrieval                        │
│      → Azure OpenAI embedding                           │
│      → GPT-4 grounded generation                        │
│                                                         │
│  MLOps Layer                                            │
│      → MLflow experiment tracking                       │
│      → Drift monitoring                                 │
│      → Automated retraining triggers                    │
└──────────────────────┬──────────────────────────────────┘
                       ↓
┌──────────────┬───────────────┬──────────────────────────┐
│  Azure       │  Databricks   │  Azure Monitor           │
│  OpenAI      │  Delta Lake   │  Telemetry               │
└──────────────┴───────────────┴──────────────────────────┘
```

**Key design decisions:**

| Decision | Rationale |
|----------|-----------|
| AKS over Azure Functions | ML workloads need persistent GPU resources — serverless cold starts unacceptable |
| Pinecone for vector storage | At 100k+ documents, managed vector DB outperforms in-memory FAISS |
| MLflow for experiment tracking | Open-source, portable across clouds, integrates with Databricks |
| Databricks Delta Lake | ACID transactions on ML feature data — critical for audit traceability |

**Outcomes achieved:**
- 50% faster KYC processing
- 12% fraud detection accuracy improvement
- 15% cloud cost optimisation via serverless design

---

## Cloud Provider Comparison

| Capability | GCP | AWS | Azure |
|-----------|-----|-----|-------|
| Best LLM | Gemini 2.5 Flash | Claude 3.5 Sonnet | GPT-4o |
| Managed Search | Vertex AI Search | Kendra | Azure Cognitive Search |
| Vector DB | BigQuery Vector Search | OpenSearch | Azure AI Search |
| Serverless | Cloud Run | Lambda | Azure Functions |
| Orchestration | LangGraph on Cloud Run | Bedrock Agents | Azure AI Studio |
| Data Platform | BigQuery | Redshift | Synapse Analytics |
| Compliance | Cloud DLP + Model Armor | Macie + Guardrails | Azure Content Safety |
| Best for | GCP-native, retail/search | Enterprise, regulated | Microsoft shop, FinTech |

---

## When to Use Each Cloud

```
Choose GCP when:
→ You need Vertex AI Search for Commerce
→ BigQuery is your data platform
→ LangGraph + Gemini is the target stack
→ GCP-native data sovereignty required

Choose AWS when:
→ Claude 3.5 Sonnet is the required model
→ Existing AWS infrastructure
→ Lambda/serverless-first architecture
→ Strong Bedrock Agents ecosystem preferred

Choose Azure when:
→ Microsoft enterprise agreements in place
→ GPT-4 with Azure compliance controls needed
→ Teams/Office 365 integration required
→ European data residency with GDPR focus
```

---

## Author

**Nadeem Ahmad** — Principal Enterprise AI Architect  
15+ years across GCP, AWS, and Azure in telecom, fintech, and retail  
📧 nadeem.ahmad.arch@gmail.com  
🔗 [LinkedIn](https://www.linkedin.com/in/nadeem-ahmad-0ba5b328/)  
📅 [Book a session](https://topmate.io/nadeem_ahmad17)
