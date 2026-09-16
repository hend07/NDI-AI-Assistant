# 📊 Smart Data Governance & Risk Management Platform — NDI

**Course:** Modern Data Engineering for Advanced AI Systems
**Organization:** SDAIA Academy — https://github.com/SDAIAAcademy

A Python-based platform for evidence-based governance analysis on the **National Data Index (NDI)** issued by **SDAIA**, combining data processing, data quality, governance, persistent vector search, and Advanced RAG into one unified pipeline.

---

## 📝 Problem & Solution

**Problem:** SDAIA's NDI documentation is extensive, making it time-consuming to manually find the exact requirement or evidence needed to answer a specific question.

**Solution:** An **Advanced RAG system** that ingests the official NDI PDF, chunks and embeds it, stores it in **ChromaDB**, and answers questions in Arabic through an interactive **Gradio chat** — always grounded in retrieved evidence, with page-level source references, and never fabricating an answer when evidence is insufficient.

Example questions users can ask:
> ما متطلبات حوكمة البيانات؟ | ما متطلبات جودة البيانات؟ | ما الضوابط المتعلقة بالامتثال؟

---

## 🏗️ System Architecture — Six Integrated Components

| # | Component | What it does |
|---|---|---|
| 1 | **Data Ingestion (Bronze/Silver/Gold)** | PDF text flows through raw extraction (Bronze) → cleaned & chunked (Silver) → embedded & indexed (Gold), each stage written to disk for traceability. |
| 2 | **Real-Time Pipeline** | An `asyncio.Queue`-based pipeline validates and normalizes incoming events, then embeds and upserts accepted ones directly into the same ChromaDB collection used for retrieval. |
| 3 | **Vector Database** | **ChromaDB** (persistent, local) stores NDI chunks and real-time events using the `paraphrase-multilingual-MiniLM-L12-v2` embedding model (~900-char chunks, 150-char overlap). |
| 4 | **Advanced RAG** | Query expansion → hybrid retrieval (semantic + lexical) → MMR re-ranking for diversity → grounded generation with page citations and a safe fallback when the LLM is unavailable. |
| 5 | **Data Quality** | Validates schema, completeness, duplicates, and field validity (timestamps, numeric values, source) before any record is indexed. Covered by automated self-tests. |
| 6 | **Governance & RBAC** | A data catalog tracks owner, classification, retention, and lineage per asset. Access is checked against roles (`data_analyst`, `governance_officer`, `auditor`) before any query runs — a foundation for more granular access control later. |

**Lineage (document path):** `Source PDF → Bronze → Silver → Gold → ChromaDB → Retrieval → AI Response`
**Lineage (real-time path):** `Incoming Event → Validation → Normalization → Embedding → ChromaDB`

---

## 🚀 Technology Stack

| Component | Technology |
|---|---|
| Language / UI | Python / Gradio |
| Vector DB | ChromaDB |
| Embeddings | SentenceTransformers — `paraphrase-multilingual-MiniLM-L12-v2` |
| Processing | Pandas, PyPDF, `asyncio` |
| LLM Gateway | OpenRouter API |
| Retrieval | Hybrid Search + MMR (RAG) |

---

## 📁 Storage Structure

```text
ndi_capstone/
├── bronze/    <file>_raw.jsonl                 # raw text per page
├── silver/    <file>_chunks.jsonl               # cleaned, chunked text
├── gold/      <file>_indexed_manifest.json      # indexed chunk IDs + metadata
└── chroma_db/ chroma.sqlite3                    # persistent vector store
```

> ⚠️ **PDF path:** the notebook expects the NDI PDF at `/content/National-Data-Index_v1.0_AR.PDF`. Update this path in the ingestion cell if your file is located elsewhere (e.g., Google Drive).

---

## 💬 Output

The **NDI Governance & RAG Chat** interface returns evidence-grounded answers with source/page references.

![NDI Governance & RAG Chatbot](chatbot.png)
![NDI Governance & RAG Chatbot1](chatbot1.png)
