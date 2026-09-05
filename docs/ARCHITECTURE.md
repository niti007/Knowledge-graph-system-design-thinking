# Fine-Grained Entity Recognition

## 1. Introduction

This document presents the end-to-end architecture for building a knowledge graph over **~500,000 highly interrelated documents** across four sources (Azure Blob Storage, SharePoint, local drives, Microsoft Docs / OneDrive). The system powers Graph-RAG retrieval, knowledge discovery, and conversational AI with a sub-second query latency target.

The architecture is composed of **five layers**:

1. **Ingestion & Extraction** — pull documents from 4 sources, parse, and chunk them.
2. **Knowledge Extraction** — extract entities, relationships, events, and map to an ontology.
3. **Graph Construction** — build the graph in Azure Cosmos DB (Gremlin) and vector index in Azure AI Search.
4. **Retrieval & Query** — route queries and combine graph + vector retrieval.
5. **Presentation** — chat interface, knowledge explorer, analytics dashboard.

---

## 2. High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DATA SOURCES (500K Documents)                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│  │  Azure   │  │SharePoint│  │  Local   │  │ Microsoft│                   │
│  │   Blob   │  │          │  │  Drive   │  │   Docs   │                   │
│  │ Storage  │  │          │  │          │  │          │                   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘                   │
│       │              │              │              │                         │
└───────┼──────────────┼──────────────┼──────────────┼─────────────────────────┘
        │              │              │              │
        ▼              ▼              ▼              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     LAYER 1: INGESTION & EXTRACTION                         │
│                                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Azure Data  │  │  Document    │  │   Content    │  │   Azure      │  │
│  │  Factory /   │──│  Parser      │──│  Chunker     │──│  AI Document  │  │
│  │  Pipeline    │  │  (Unstructured│  │  (Semantic)  │  │  Intelligence │  │
│  └──────────────┘  │   Library)   │  └──────────────┘  └──────────────┘  │
│                    └──────────────┘                                        │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     LAYER 2: KNOWLEDGE EXTRACTION                           │
│                                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Entity     │  │ Relationship │  │    Event     │  │   Domain     │  │
│  │  Extraction  │  │  Extraction  │  │  Extraction  │  │  Ontology    │  │
│  │  (NER/LLM)  │  │  (LLM/RE)    │  │  (Temporal)  │  │  Mapper      │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                    Azure AI Language / OpenAI GPT-4o                  │  │
│  │            (Batch processing with Azure Container Instances)         │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     LAYER 3: GRAPH CONSTRUCTION                             │
│                                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                     │
│  │  Ontology    │  │   Graph      │  │  Document    │                     │
│  │  Schema      │──│  Builder     │──│  Embedding   │                     │
│  │  Manager     │  │  (Gremlin)   │  │  Store       │                     │
│  └──────────────┘  └──────────────┘  └──────────────┘                     │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │              Azure Cosmos DB (Gremlin API)                           │  │
│  │         + Azure AI Search (Vector Index)                             │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     LAYER 4: RETRIEVAL & QUERY                              │
│                                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   Query      │  │  Graph       │  │   Vector     │  │   Hybrid     │  │
│  │  Analyzer    │──│  Retriever   │──│  Retriever   │──│  Combiner    │  │
│  │  (Intent)    │  │  (Gremlin)   │  │  (Cosmos)    │  │              │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │              Azure Container Apps (API + Frontend)                   │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                     LAYER 5: PRESENTATION                                   │
│                                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                     │
│  │  Chat        │  │  Knowledge   │  │  Analytics   │                     │
│  │  Interface   │  │  Explorer    │  │  Dashboard   │                     │
│  └──────────────┘  └──────────────┘  └──────────────┘                     │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │              Azure Static Web Apps + React/TypeScript                 │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Layer 1 — Ingestion & Extraction

### 3.1 Data Source Connectors

| Source | Connector | Mechanism |
|--------|-----------|-----------|
| Azure Blob Storage | Azure SDK | Enumerate & download files |
| SharePoint | Microsoft Graph API | OAuth2, Document Libraries, Lists |
| Local drive | ADF self-hosted runtime | Or sync to Blob first |
| Microsoft Docs | Microsoft Graph API | OneDrive / live documents |

**Key decision:** For local drive + Microsoft Docs, **sync to Azure Blob Storage first** to have a single uniform ingestion point. This avoids maintaining four separate connectors in the processing pipeline.

### 3.2 Document Parsing

Use the **`unstructured`** library for format-agnostic parsing:

```python
from unstructured.partition.pdf import partition_pdf
from unstructured.partition.docx import partition_pdf
from unstructured.partition.csv import partition_csv

# Outputs: List[Element] with Title, NarrativeText, Table, ListItem, etc.
# Handles: OCR for scanned PDFs, table extraction, metadata extraction
```

### 3.3 Semantic Chunking

Instead of fixed-size chunks, chunk by document structure:

1. Split by headings / sections.
2. Further split large sections at paragraph boundaries.
3. Maintain per-chunk metadata (source, page, section, format).
4. Target chunk size: 512–1024 tokens for optimal embeddings.

Scale estimate: **500K documents × avg 10 pages × ~3 chunks/page ≈ 15M chunks**.

### 3.4 Batch Processing Architecture

```
500K documents × 10 pages × 3 chunks/page = ~15M chunks

Processing Pipeline:
├── Azure Data Factory (orchestration)
├── Azure Container Instances (parallel batch)
│   ├── 10–20 containers
│   ├── Each processes ~25K documents
│   └── Est. 2–3 weeks at balanced cost
├── Azure Queue Storage (task distribution)
└── Azure Blob Storage (intermediate results)
```

---

## 4. Layer 2 — Knowledge Extraction

### 4.1 Ontology Schema

```yaml
# Core Entity Types
Document:
  properties: [id, title, source, format, created_date, modified_date, url]
  relationships: [CONTAINS, REFERENCES, DERIVED_FROM]

Person:
  properties: [id, name, role, organization, email]
  relationships: [AUTHORED, MENTIONED_IN, WORKS_WITH]

Organization:
  properties: [id, name, type, industry, location]
  relationships: [MENTIONED_IN, PARTNER_OF, SUBSIDIARY_OF]

Project:
  properties: [id, name, status, start_date, end_date, budget]
  relationships: [MENTIONED_IN, DEPENDS_ON, SUPERSEDES]

Concept:
  properties: [id, name, domain, definition]
  relationships: [RELATED_TO, PART_OF, CONTRADICTS]

Event:
  properties: [id, name, date, location, type]
  relationships: [MENTIONED_IN, CAUSED_BY, RESULTED_IN]

# Relationship Types
CONTAINS:      Document → [Person, Organization, Project, Concept]
REFERENCES:    Document → Document
DERIVED_FROM:  Document → Document (versioning, citations)
AUTHORED:      Person → Document
MENTIONED_IN:  [Person, Organization, Project, Concept] → Document
DEPENDS_ON:    Project → Project
RELATED_TO:    Concept → Concept (semantic similarity)
```

### 4.2 Extraction For Investment Decisions

The choice of extraction method is the single largest cost driver. See
[EXTRACTION_COST_ANALYSIS.md](EXTRACTION_COST_ANALYSIS.md) for a full comparison. The recommended approach is a **tiered pipeline**:

1. **Tier 1 — GLiNER-Relex** (all 500K docs): fast, self-hosted, joint entity + relation extraction.
2. **Tier 2 — GPT-4.1-nano (batch)** (low-confidence ~30%): domain-specific entities and re-extraction.
3. **Tier 3 — GPT-4o-mini (batch)** (complex relations ~10%): implicit/cross-document relations, disambiguation.

This cuts extraction cost **65–83%** versus a pure LLM approach while keeping ~90%+ accuracy.

### 4.3 Deduplication & Entity Resolution

```
# Critical step: merge duplicate entities across documents
# 1. Exact match: Same name/ID
# 2. Fuzzy match: Levenshtein distance on names
# 3. Contextual match: LLM-based entity resolution
# 4. Canonical form: Normalize names (Inc vs Incorporated)

# Store entity embeddings; cosine similarity 0.85+ → candidate duplicate
```

---

## 5. Layer 3 — Graph Construction (Azure Cosmos DB)

### 5.1 Cosmos DB Configuration

```
# Azure Cosmos DB Gremlin API
# Container: knowledge-graph
# Partition Key: /entityType   (ensures even distribution)
# Throughput: 10,000 RU/s (autoscale to 100,000)
# Cost: ~$1,500–3,000/month at 10K RU/s autoscale

# Graph size estimate:
# - 500K Document nodes
# - ~2M unique entity nodes (from ~10M mentions)
# - ~30M relationships
# Total: ~20.5M nodes + 30M edges
```

### 5.2 Graph Schema in Gremlin

```python
# Document nodes
g.addV('Document').property('id', 'doc_001').property('title', 'Q4 Report')
  .property('source', 'azure_blob').property('format', 'pdf')
  .property('created_date', '2024-01-15').property('embedding', [0.1, 0.2, ...])

# Person nodes
g.addV('Person').property('id', 'person_001').property('name', 'John Smith')
  .property('role', 'CEO').property('organization', 'Acme Corp')

# Relationships
g.V('doc_001').addE('CONTAINS').to(g.V('person_001'))
  .property('confidence', 0.95).property('context', 'authored this report')

# Document-to-Document references
g.V('doc_001').addE('REFERENCES').to(g.V('doc_002'))
  .property('type', 'citation').property('page', 15)
```

### 5.3 Vector Index (Azure AI Search)

```
# Index: document-embeddings
# Fields:
#   - document_id    (key)
#   - chunk_text     (searchable)
#   - embedding      (vector, 1536-dim via text-embedding-3-small)
#   - metadata       (filterable: source, date range, entity type)

# Hybrid search: vector similarity + full-text search
# Supports filters: source, date range, entity type
```

---

## 6. Layer 4 — Retrieval & Query

### 6.1 Query Router

```python
# Three-way routing based on query intent:
# 1. "graph"   → Gremlin traversal (relationship queries)
# 2. "vector"  → Azure AI Search (semantic similarity)
# 3. "hybrid"  → Combined (most complex queries)

# "Who authored the Q4 report?"                     → graph
# "Find documents similar to this contract"          → vector
# "What projects depend on Project Alpha?"           → graph
# "Summarize all docs about machine learning"        → hybrid
```

### 6.2 Graph Retriever (Gremlin)

```python
# Example: "Find all people who worked on projects that depend on Project Alpha"
g.V().has('Project', 'name', 'Project Alpha')
  .out('DEPENDS_ON').as_('dep_project')
  .in_('MENTIONED_IN').as_('doc')
  .out('CONTAINS').hasLabel('Person')
  .path().by('name')

# Limit traversal depth; cache frequent traversals in Azure Cache for Redis
```

### 6.3 Hybrid Combiner

```python
def hybrid_retrieve(query: str) -> dict:
    # 1. Vector: top-10 similar document chunks
    vector_results = vector_search(query, top_k=10)

    # 2. Graph: extract entities from query, traverse to neighbors
    query_entities = extract_entities(query)
    graph_context = graph_traverse(query_entities, max_hops=2)

    # 3. Merge, dedupe, generate grounded answer
    combined_context = merge_contexts(vector_results, graph_context)
    answer = llm.generate(query, combined_context)
    return {"answer": answer, "sources": ..., "relationships": ..., "confidence": ...}
```

### 6.4 Caching for <1s Latency

```
- Azure Cache for Redis
- Cache: entity lookups, frequent traversals, classified query intents
- TTL-based invalidation on document ingestion events
```

---

## 7. Implementation Phases (~10 weeks)

| Phase | Duration | Deliverables |
|-------|----------|--------------|
| 1. Foundation | Weeks 1–2 | Azure infra, ingestion pipeline, semantic chunking; test on 1K docs |
| 2. Extraction | Weeks 3–4 | Ontology schema, tiered extraction, entity resolution; 50K docs |
| 3. Graph Construction | Weeks 5–6 | Cosmos DB builder, AI Search index, schema optimization; remaining 450K |
| 4. Retrieval & Query | Weeks 7–8 | Query router, graph retriever, hybrid combiner, caching |
| 5. Frontend & Polish | Weeks 9–10 | Chat UI, knowledge explorer, analytics, performance tuning |

---

## 8. Cost Estimates

### 8.1 One-Time Ingestion Pipeline

| Component | Configuration | Cost |
|-----------|---------------|------|
| Azure Data Factory | 500 pipeline runs | ~$200 |
| Azure Container Instances | 10 containers × 2 weeks | ~$3,000 |
| Azure OpenAI (extraction) | Tiered pipeline (15M chunks) | ~$125–275 |
| Azure Blob Storage | 500K docs × 10MB avg | ~$100/month |
| **Total One-Time** | | **~$3,400–3,600** |

> Note: A pure LLM (GPT-4o-mini) extraction would cost ~$750+ for extraction alone; the tiered pipeline reduces this as described in the cost analysis doc.

### 8.2 Ongoing Monthly Infrastructure

| Component | Configuration | Cost |
|-----------|---------------|------|
| Azure Cosmos DB | 10K RU/s autoscale | ~$2,000 |
| Azure AI Search | S1 tier, 50GB | ~$700 |
| Azure Container Apps | 2 vCPU, 4GB | ~$300 |
| Azure Cache for Redis | C1 tier | ~$100 |
| Azure Blob Storage | 5TB archive | ~$100 |
| Azure OpenAI (queries) | 100K queries/month | ~$500 |
| **Total Monthly** | | **~$3,700** |

---

## 9. Risk Mitigation

| Risk | Mitigation |
|------|------------|
| LLM extraction quality | Human-in-the-loop validation on 5% sample |
| Schema evolution | Versioned ontology + migration scripts |
| Processing failures | Checkpoint + resume from last batch |
| Cost overruns | Budget alerts, autoscale limits |
| Query performance | Path indexing, cache, query hints |

---

## 10. Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Graph DB | Azure Cosmos DB Gremlin | Native Azure, multi-region, serverless |
| Vector store | Azure AI Search | Hybrid keyword + vector, filtering |
| Extraction | Tiered (GLiNER → GPT-nano → GPT-mini) | 65–83% cheaper, comparable accuracy |
| Chunking | Semantic / structure-aware | Preserve context, better embeddings |
| Processing | Azure batch (containers + queues) | Cost-efficient finite corpus |
| Entity resolution | Canonical + fuzzy + LLM | Dedupe millions of mentions |
| Latency | Cache + path indexing | Sub-second target |
