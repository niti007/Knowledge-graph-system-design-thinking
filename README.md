# Knowledge Graph System Design Thinking

End-to-end architecture and design for building a **knowledge graph over ~500,000 highly interrelated documents** to power Graph-RAG retrieval, knowledge discovery, and conversational AI.

This repository is a system-design deep-dive — a worked thinking exercise covering the full lifecycle: ingestion, extraction, graph construction, retrieval, and cost optimization. It does **not** contain runnable application code; it documents the architecture, trade-offs, and decisions needed to design such a system.

---

## The Problem Statement

You have **~500,000 documents** — PDFs, CSVs, DOCX, and more — scattered across four sources:

| Source | Example |
|--------|---------|
| Azure Blob Storage | Archived reports, exports |
| SharePoint | Team document libraries, wikis |
| Local/computer drives | Loose files, working copies |
| Microsoft Docs / OneDrive | Live, collaborated documents |

**The core problem:** these documents are **highly interrelated**. A single report references a project, which depends on another project, which was authored by a person who also appears in a contract, which cites yet another document. Traditional RAG (vector search over chunks) answers "find similar text" well but **fails at relationship-based questions** like:

> *"Which projects depend on Project Alpha, and who has authored documents about each of them?"*

> *"Show me all documents, across every source, that reference both 'Acme Corp' and the 'Q4 budget'."*

To solve this, we need a **knowledge graph** (Azure Cosmos DB Gremlin API) that models entities (people, organizations, projects, concepts, events) and the relationships between them **and between documents themselves**.

The hard questions this design answers:

1. **How do you ingest 500K documents from 4 heterogeneous sources?**
2. **How do you extract entities and relationships at scale, affordably?**
3. **How do you resolve duplicate entities across millions of mentions?**
4. **How do you query the graph in <1 second at production latency?**
5. **What is the cheapest way to extract relationships — is NER sufficient?**

---

## Why Graph RAG (and not just Traditional RAG)

| Capability | Traditional Vector RAG | Graph RAG |
|------------|------------------------|-----------|
| Semantic similarity search | ✅ Strong | ✅ Via vector index |
| Multi-hop relationships (X → Y → Z) | ❌ Weak | ✅ Strong |
| Cross-document entity aggregation | ❌ Weak | ✅ Strong |
| Provenance / citation chains | ❌ Weak | ✅ Strong |
| Graph analytics (centrality, paths) | ❌ None | ✅ Native |

The winning design is **hybrid**: vector search for semantic recall + graph traversal for relational reasoning, combined through a query router.

---

## Repository Structure

```
Knowledge-graph-system-design-thinking/
├── README.md                          # This file — problem statement & overview
└── docs/
    ├── ARCHITECTURE.md                # 5-layer end-to-end architecture
    └── EXTRACTION_COST_ANALYSIS.md    # Cost / latency / accuracy trade-off for extraction
```

---

## Quick Navigation

- **[Architecture](docs/ARCHITECTURE.md)** — The 5-layer architecture (sources → ingestion → extraction → graph construction → retrieval → presentation), Azure Cosmos DB graph schema, ontology, and implementation phases.
- **[Extraction Cost Analysis](docs/EXTRACTION_COST_ANALYSIS.md)** — Is NER sufficient? A head-to-head cost/latency/accuracy comparison of spaCy, GLiNER, REBEL, Azure AI Language, and LLM-based extraction — plus the recommended **tiered pipeline** that cuts cost 65–83%.

---

## Key Design Decisions (Summary)

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Graph database | Azure Cosmos DB (Gremlin) | Native Azure integration, multi-region replication, serverless |
| Vector index | Azure AI Search | Hybrid keyword + vector, filterable metadata |
| Extraction engine | Tiered (GLiNER → GPT-nano → GPT-mini) | 65–83% cheaper than LLM-only, comparable accuracy |
| Chunking | Semantic (structure-aware) | Preserves document context, better embeddings |
| Processing | Azure batch (containers + queues) | Cost-efficient for finite 500K corpus |
| Entity resolution | Canonical form + fuzzy + LLM merge | Dedupe millions of mentions accurately |

---

## Cost at a Glance (Azure)

- **One-time ingestion pipeline:** ~$18K (dominated by LLM extraction)
- **Ongoing monthly infrastructure:** ~$3.7K (Cosmos DB, AI Search, Containers, Cache, LLM queries)
- **Extraction-only cost:** ~$125–275 with tiered pipeline (**vs ~$750+ for LLM-only**)

See [EXTRACTION_COST_ANALYSIS.md](docs/EXTRACTION_COST_ANALYSIS.md) for the detailed breakdown.

---

## License

This repository is documentation/design content. See the `docs/` files for details.
