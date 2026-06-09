# AI Inference + RAG Platform Architecture

## Purpose

This project is a practical AI infrastructure portfolio project.

The goal is to demonstrate the ability to deploy and operate a Kubernetes-based AI application stack using:

- Kubernetes
- vLLM for LLM inference
- TEI for embeddings
- Qdrant for vector retrieval
- RabbitMQ for async document ingestion
- Postgres for application state
- S3-compatible object storage for files
- A custom FastAPI orchestration layer

The user-facing demo is a private company chatbot that can ingest documents and answer questions from those documents. The deeper goal is to demonstrate AI platform operations competence, not to build a feature-heavy chatbot product.

## Design Principles

1. Use existing model-serving and infrastructure components.
2. Write custom glue only where it clarifies system boundaries.
3. Keep Phase 1 focused on Kubernetes, vLLM, RAG, and async ingestion.
4. Defer production observability, advanced auth, and complex routing to later phases.
5. Avoid overarchitecting, but preserve realistic service boundaries.

## Phase 1 Scope

Phase 1 should prove:

> A user can upload a document, the system indexes it asynchronously, and the user can ask a question that is answered by an LLM running behind vLLM on Kubernetes using retrieved document context.

## High-Level Components

### Service Delivery Core

| Component | Purpose | Initial Choice |
|---|---|---|
| Frontend / Web App | User-facing chatbot and document upload UI | TBD, mostly AI-generated |
| API / Orchestrator | Coordinates RAG query flow and application logic | FastAPI + custom glue code |
| Embedding Runtime | Converts text and queries into vectors | TEI |
| Embedding Model | Model used by TEI | BGE-small, likely `BAAI/bge-small-en-v1.5` or current equivalent |
| Vector Database | Stores vectors, chunk text, and retrieval metadata | Qdrant |
| Application Database | Users, documents, request history, job status, metadata | Postgres |
| Object Storage | Uploaded files, generated files, extracted text artifacts | S3-compatible storage / MinIO / cloud S3 |
| Message Broker | Async document ingestion jobs | RabbitMQ |
| Worker Service | Consumes ingestion jobs and updates vector/app stores | Custom worker code |
| LLM Runtime | Serves generation model behind API | vLLM |
| Generation Model | Main text generation model | Qwen or Llama, TBD |

### Operations / Access

| Component | Purpose | Initial Choice |
|---|---|---|
| Kubernetes Control Plane | Cluster scheduling and control | TBD |
| Bastion Host | Administrative SSH entry point | Separate from control plane |
| Secrets Management | Avoid storing secrets directly in environment or repo | TBD |
| Monitoring / Logging | Deferred to Phase 3 | Kubernetes-native logs initially |

## Request Flows

## Document Ingestion Flow

Document ingestion is asynchronous.

```text
User uploads document
        ↓
Web/API stores file in object storage
        ↓
Postgres document record created: uploaded
        ↓
Ingestion job published to RabbitMQ
        ↓
Worker consumes job
        ↓
Worker downloads file from object storage
        ↓
Worker extracts text
        ↓
Worker chunks text
        ↓
Worker sends chunks to TEI
        ↓
TEI returns embeddings
        ↓
Worker upserts vectors + chunk text + metadata into Qdrant
        ↓
Worker updates Postgres document status: ready or failed
```

## Question-Answer Flow

Question-answer retrieval and generation are synchronous in Phase 1.

```text
User asks question
        ↓
Web/API sends request to orchestrator
        ↓
Orchestrator calls TEI to embed query
        ↓
TEI returns query vector
        ↓
Orchestrator searches Qdrant with vector + user/org/document filters
        ↓
Qdrant returns relevant chunks, source metadata, and similarity scores
        ↓
Orchestrator builds prompt from question + retrieved context
        ↓
Orchestrator calls vLLM
        ↓
vLLM generates response
        ↓
Orchestrator returns answer to user
```

## Important Architectural Boundaries

### Embedding vs Generation

Embedding and generation are separate model-serving concerns.

- TEI + BGE-small handles embeddings.
- vLLM + Qwen/Llama handles text generation.

The orchestrator should not hide model execution inside its own process.

### Retrieval vs Reasoning

Qdrant retrieves relevant chunks. It does not reason.

vLLM generates the final response. It does not search the vector database directly.

The orchestrator coordinates both.

### Vector DB vs Application DB

Qdrant stores vector search data:

- embeddings
- chunk text
- document metadata useful for retrieval
- similarity-search payloads

Postgres stores application state:

- users
- organizations
- documents
- document status
- chat sessions
- request history
- job status
- permissions

### Object Storage vs Vector Storage

Object storage stores source artifacts:

- original document files
- generated output files
- optionally extracted text

Qdrant should not be treated as the system of record for uploaded files.

## Security Notes

- The Kubernetes control plane should not be directly exposed.
- Administrative access should go through a bastion host.
- User identity should be included in the web/API layer.
- Qdrant searches must be filtered by user/org/document scope.
- Secrets should not be committed to the repo or stored directly in environment variables as the long-term strategy.
- Final secrets-management decision is TBD.

## Deferred / Phase 2+ Ideas

These are intentionally not Phase 1 requirements.

- Async Q&A request handling with polling
- SSE or WebSocket streaming
- Advanced document permissions
- Multi-tenant organization model
- Reranking
- Retrieval routing
- Model switching
- Usage billing
- Advanced model lifecycle management
- Autoscaling policies
- Full observability stack
- Production log aggregation
- Distributed tracing

## Deferred / Phase 3 Operations

The initial project should avoid spending too much effort re-proving general production operations experience.

Phase 3 may add:

- Prometheus
- Grafana
- GPU metrics exporter
- Loki
- alerting
- structured logs
- request tracing
- queue depth dashboards
- vLLM tokens/sec dashboards
- VRAM/GPU utilization dashboards

## Remaining Decisions

The next concrete decisions are:

1. Kubernetes target:
   - local k3s/kind
   - cloud Kubernetes
   - rented GPU host with Kubernetes
   - managed Kubernetes plus GPU node pool

2. Generation model:
   - Qwen
   - Llama
   - Gemma
   - other open model

3. GPU target:
   - local GPU
   - rented GPU VM
   - cloud GPU node

4. Object storage:
   - MinIO
   - AWS S3
   - Cloudflare R2
   - other S3-compatible storage

5. Secrets management:
   - Kubernetes Secrets for first cut
   - Sealed Secrets
   - External Secrets Operator
   - Vault
   - cloud secret manager

6. Web application stack:
   - simple generated frontend
   - Open WebUI adaptation
   - custom minimal app

7. Deployment style:
   - raw Kubernetes manifests
   - Helm
   - Kustomize
   - GitOps later

## Current Architecture Summary

```text
User
 ↓
Frontend / Web App
 ↓
FastAPI Orchestrator
 ├── Postgres
 ├── Object Storage
 ├── RabbitMQ
 ├── TEI Embedding Service → BGE-small
 ├── Qdrant Vector DB
 └── vLLM Inference Service → Qwen/Llama
```

Async ingestion path:

```text
Upload
 ↓
Object Storage
 ↓
RabbitMQ
 ↓
Worker
 ↓
Text extraction / chunking
 ↓
TEI
 ↓
Qdrant
 ↓
Postgres status update
```

Synchronous Q&A path:

```text
Question
 ↓
TEI query embedding
 ↓
Qdrant retrieval
 ↓
Prompt construction
 ↓
vLLM generation
 ↓
Answer
```

## Positioning

This project should be described as:

> A Kubernetes-based AI inference and RAG platform using vLLM, TEI, Qdrant, RabbitMQ, and Postgres, with asynchronous document ingestion and a minimal chatbot interface for demonstration.

It should not be positioned primarily as:

> A chatbot app.

The chatbot is the demo. The platform is the point.
