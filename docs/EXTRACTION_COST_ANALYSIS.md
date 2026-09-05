# Extraction Method: Cost vs Latency vs Accuracy

This document answers the central extraction question: **how do you extract relationships across 500K documents affordably — and is Name-Entity Recognition (NER) sufficient?**

It presents a head-to-head comparison of extraction approaches and a recommendation for a **tiered pipeline** that cuts cost by 65–83%.

---

## 1. Dataset Assumptions

| Metric | Value |
|--------|-------|
| Documents | 500,000 |
| Avg pages / doc | 10 |
| Avg chars / page | 1,000 |
| Total characters | 5 billion |
| Total tokens (est.) | ~1.25 billion |
| Total sentences (est.) | ~50 million |

---

## 2. Head-to-Head Comparison

### 2.1 spaCy NER (Entities Only)

| Metric | Value |
|--------|-------|
| **Cost** | **~$70–140** (self-hosted CPU) |
| **Latency** | **<1 ms / doc** → ~8 min total |
| **Entity F1** | 87–90% |
| **Relation F1** | ❌ None |
| **Throughput** | 200–10,000 docs/sec |
| **Setup** | Low |

**Outcome:** Cheapest for entities only. Cannot extract relationships. Best used as a fast first pass.

### 2.2 GLiNER / GLiNER-Relex (Entities + Relations)

| Metric | Value |
|--------|-------|
| **Cost** | **~$10–50** (1× A100 GPU, ~7 hrs) |
| **Latency** | **20–100 ms / doc** → ~3–14 hrs total |
| **Entity F1** | 82–87% |
| **Relation F1** | 70–80% (GLiNER-Relex) |
| **Throughput** | 10–50 docs/sec |
| **Setup** | Medium |

**Outcome:** Best cost-accuracy for **joint entity + relation extraction** — runs locally, no API cost. Fixed relationship types (fine-tunable on 220+ Wikidata types + custom).

### 2.3 Azure AI Language NER (Entities Only)

| Metric | Value |
|--------|-------|
| **Cost** | **~$5,000** (~$1 / 1,000 text records) |
| **Latency** | **100–500 ms / doc** → ~14–70 hrs |
| **Entity F1** | 85–90% |
| **Relation F1** | ❌ None |
| **Throughput** | 2–10 docs/sec |
| **Setup** | Low (managed) |

**Outcome:** Expensive at 500K scale. No relationship extraction. Fits Azure-integrated workflows only.

### 2.4 GPT-4o-mini (Full Extraction)

| Metric | Value |
|--------|-------|
| **Cost** | **~$750** (on-demand) / **~$375** (Batch API) |
| **Latency** | **1–5 s / doc** → ~6–29 days |
| **Entity F1** | 85–92% |
| **Relation F1** | 75–85% |
| **Throughput** | 0.2–1 docs/sec |
| **Setup** | Low |

**Outcome:** High quality but expensive at 500K scale. Batch API halves cost; adds ~24 hr latency.

### 2.5 GPT-4.1-nano (Extraction-Optimized)

| Metric | Value |
|--------|-------|
| **Cost** | **~$500** (on-demand) / **~$250** (Batch API) |
| **Latency** | **<1 s / doc** → ~6–14 days |
| **Entity F1** | 80–88% |
| **Relation F1** | 70–80% |
| **Throughput** | 1–5 docs/sec |
| **Setup** | Low |

**Outcome:** Best LLM price-performance for extraction (20× cheaper per output token than GPT-4.1 full).

### 2.6 REBEL (Joint Entity + Relation)

| Metric | Value |
|--------|-------|
| **Cost** | **~$10–30** (1× GPU, ~5 hrs) |
| **Latency** | **20–100 ms / doc** → ~3–14 hrs |
| **Entity F1** | 82–87% |
| **Relation F1** | 70–80% |
| **Throughput** | 10–50 docs/sec |
| **Setup** | Medium |

**Outcome:** Similar to GLiNER but less actively maintained. 220 pre-trained relationship types.

---

## 3. Summary Comparison Table

| Method | Cost (est.) | Latency / doc | Entity F1 | Relation F1 | Total Time |
|--------|-------------|---------------|-----------|-------------|------------|
| **spaCy** | $70–140 | <1 ms | 87–90% | ❌ None | ~8 min |
| **GLiNER** | $10–50 | 20–100 ms | 82–87% | 70–80% | 3–14 hrs |
| **Azure AI** | ~$5,000 | 100–500 ms | 85–90% | ❌ None | 14–70 hrs |
| **GPT-4o-mini** | $375–750 | 1–5 s | 85–92% | 75–85% | 6–29 days |
| **GPT-4.1-nano** | $250–500 | <1 s | 80–88% | 70–80% | 6–14 days |
| **REBEL** | $10–30 | 20–100 ms | 82–87% | 70–80% | 3–14 hrs |

> **Key insight:** Traditional NER (spaCy / Azure AI Language) is *only sufficient when you need entities and can afford a subsequent rule- or LLM-based pass for relationships.* For relationship extraction at scale, self-hosted joint models (GLiNER, REBEL) dominate on cost/latency; LLMs dominate on flexibility.

---

## 4. Recommended: Tiered Pipeline (65–83% Cost Reduction)

The optimal design **combines** methods — using cheap/fast extraction everywhere and expensive LLMs only where needed:

```
┌─────────────────────────────────────────────────────────────────┐
│                    TIERED EXTRACTION PIPELINE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Tier 1: GLiNER-Relex (All 500K docs)                          │
│  ├── Extract standard entities (Person, Org, Location, Date)   │
│  ├── Extract relationships using GLiNER-Relex                  │
│  ├── Cost: ~$10–50 (GPU compute only)                          │
│  └── Coverage: 70–80% entities, 60–70% relations               │
│                                                                  │
│  Tier 2: GPT-4.1-nano (Low confidence only, ~30% of docs)     │
│  ├── Re-extract where GLiNER confidence < 0.7                 │
│  ├── Extract domain-specific entities not in GLiNER vocab      │
│  ├── Cost: ~$75–150                                            │
│  └── Coverage: +15–20% entities                                │
│                                                                  │
│  Tier 3: GPT-4o-mini (Complex relations only, ~10% of docs)   │
│  ├── Extract implicit / cross-document relationships           │
│  ├── Entity disambiguation and coreference resolution          │
│  ├── Cost: ~$37–75                                             │
│  └── Coverage: +10–15% high-quality relations                  │
│                                                                  │
│  Total Cost: ~$125–275   (vs ~$750+ for LLM-only)             │
│  Total Time: ~2–3 days   (vs 6–29 days for LLM-only)          │
│  Accuracy:   ~90%+       (vs 85–92% for LLM-only)             │
└─────────────────────────────────────────────────────────────────┘
```

### Why the tiered approach wins

| Dimension | LLM-Only (GPT-4o-mini) | Tiered Pipeline | Improvement |
|-----------|-------------------------|-----------------|-------------|
| Total cost | ~$750 | ~$125–275 | **65–83% cheaper** |
| Processing time | 6–29 days | 2–3 days | **70–90% faster** |
| Entity F1 | 85–92% | ~90% | Comparable |
| Relation F1 | 75–85% | ~75% | Comparable |

---

## 5. Additional Cost Optimization Strategies

### 5.1 Batch API (50% discount)
```python
# Use OpenAI Batch API for Tier 2/3
# Processes overnight at 50% off
# Trade-off: ~24 hr latency (fine for one-time ingestion)
```

### 5.2 Prompt Caching (40–90% discount on repeated input)
```python
# Cache system prompts and few-shot examples
# Reduces repeated input token cost
# Ideal for extraction prompts that repeat every call
```

### 5.3 Semantic Chunking (reduce tokens processed)
```python
# Process only high-value sections
# Skip boilerplate, headers, footers
# Reduces total tokens by 30–50%
```

### 5.4 Graphability Indexing (advanced)
```python
# From Proxy-Pointer RAG research:
# Predict which sections yield relationships
# Skip low-yield sections entirely
# Reduces LLM calls by 60–70%
```

### 5.5 Incremental Processing
```python
# Process only new/changed documents
# Use document hashing to detect changes
# Reduces ongoing costs by ~90%
```

---

## 6. Is NER Sufficient?

**No — not alone.** NER finds *entities* but cannot extract *relationships*, which is the entire point of building a knowledge graph over interrelated documents.

| If you need… | Use… |
|--------------|------|
| Just entities, max speed | spaCy `en_core_web_trf` (or GLiNER for custom types) |
| Entities + relationships, low cost | GLiNER-Relex or REBEL (self-hosted) |
| Entities + relationships, high accuracy/flexibility | LLM (GPT-4.1-nano / GPT-4o-mini) |
| Both at best cost | **Tiered pipeline** (GLiNER → GPT-nano → GPT-mini) |

**Recommendation for 500K interrelated documents:** Use the **tiered pipeline**, starting with GLiNER-Relex for all documents and reserving LLMs for low-confidence and complex-relationship cases.

---

## 7. Next Steps

1. Test GLiNER-Relex on **100 sample documents** from your corpus.
2. Measure real precision/recall for your domain.
3. Determine what % of content falls through to Tier 2/3.
4. Tune confidence thresholds based on results.
5. Validate entity resolution on a human-reviewed gold set.
