# 📊 Smart Data Governance & Risk Management Platform — NDI

**Course:** Modern Data Engineering for Advanced AI Systems
**Organization:** SDAIA Academy — https://github.com/SDAIAAcademy

A Python-based intelligent platform designed to support data governance analysis, evidence retrieval, compliance-oriented assessment, and technical and administrative risk analysis based on the **National Data Index (NDI)** issued by the **SDAIA**.

The platform integrates modern data processing, data quality, governance metadata, persistent vector storage, and advanced Retrieval-Augmented Generation (RAG) into a unified evidence-based pipeline for data governance analysis.

---
## 📝 Project Description — Problem & Solution

### ❌ The Business Problem

Organizations often face difficulties in understanding and navigating national data governance requirements issued by the **Saudi Data and Artificial Intelligence Authority (SDAIA)**, including the **NDI**.

The official governance documentation can be extensive and contain numerous requirements, controls, and guidelines. Manually searching through these documents to find the relevant requirement for a specific question can be time-consuming and may make it difficult for users to quickly identify the exact supporting evidence.

This creates a need for an intelligent system that can help users **quickly find, understand, and reference the relevant information within the official NDI documentation**.

### 💡 The Technical Solution

The project implements an **Advanced Retrieval-Augmented Generation (RAG) system in Python** that transforms the official **SDAIA NDI documentation** into an interactive, searchable knowledge base.

The system:

* Ingests the official NDI PDF document.
* Extracts and processes its content through a staged (bronze → silver → gold) pipeline.
* Splits the document into context-preserving chunks.
* Generates multilingual semantic embeddings.
* Stores the knowledge base persistently in **ChromaDB**.
* Retrieves relevant evidence using semantic and lexical retrieval techniques.
* Applies **Maximal Marginal Relevance (MMR)** to improve retrieval diversity.
* Generates evidence-grounded answers using an LLM.
* Provides source and page references for the retrieved evidence.
* Accepts live governance/risk events through an asynchronous pipeline and indexes accepted events into the same knowledge base in real time.

Through the interactive **Gradio UI**, users can ask questions such as:

> *What are the requirements for data governance?*

> *What are the requirements related to data quality?*

> *What controls are related to compliance?*

The system searches the official NDI knowledge base and provides an **evidence-grounded response with supporting source and page references**, helping users understand the requirements without manually searching through the entire document.

The platform therefore combines **data processing, data quality, governance metadata, persistent vector search, and Advanced RAG** into a unified AI-powered knowledge retrieval system for NDI-related inquiries.

---

# 🏗️ System Architecture

The platform integrates six core components into a unified data and AI pipeline:

1. Data Ingestion & Processing (Bronze / Silver / Gold)
2. Real-Time Event Processing (integrated with the vector knowledge base)
3. Vector Database & Semantic Knowledge Base
4. Advanced RAG
5. Data Quality
6. Data Governance & Role-Based Access Control

---

## 1. Data Ingestion & Processing (Bronze / Silver / Gold)

Document ingestion follows a staged, layered pipeline so that each processing step is traceable and reproducible rather than being an in-memory, one-shot conversion:

```text
NDI PDF
   │
   ▼
BRONZE  — raw extracted text per page, written to bronze/<file>_raw.jsonl
   │
   ▼
SILVER  — cleaned text, split into context-preserving chunks with stable IDs,
          written to silver/<file>_chunks.jsonl
   │
   ▼
GOLD    — chunks + embeddings indexed into ChromaDB, with an indexing
          manifest written to gold/<file>_indexed_manifest.json
```

* **Bronze** preserves the original extracted text exactly as read from the PDF, before any cleaning — useful for re-processing or auditing what the source actually contained.
* **Silver** holds the cleaned, chunked, deduplicated-by-ID text that is ready for embedding, independent of any specific vector store.
* **Gold** is the fully processed, embedded, and indexed layer that the RAG pipeline queries directly, along with a manifest recording which chunk IDs were indexed under which asset.

This gives the platform a lightweight but functional Medallion-style layering, rather than only creating empty directories.

Incoming governance and risk-related **events** (as opposed to the source document) follow a separate, lighter validation flow:

```text
Incoming Event
      │
      ▼
Schema Validation
      │
      ▼
Data Quality Checks
      │
      ▼
Normalization
      │
      ▼
Validated Event
```

---

# 2. Real-Time Data Ingestion Pipeline

The platform implements an asynchronous event-processing pipeline using **`asyncio.Queue`**.

The pipeline simulates a real-time ingestion environment in which governance and risk events can be received, validated, and processed asynchronously, and now **feeds directly into the same vector knowledge base used for RAG retrieval** rather than running as an isolated demo.

The current processing flow includes:

* Event ingestion through an asynchronous queue.
* Schema validation.
* Data-quality validation.
* Event normalization.
* Duplicate detection.
* Embedding of the validated event and upsert into the shared ChromaDB collection, tagged as `type: real_time_event`.
* Structured processing results, including the indexed document ID when successful.

Because accepted events are embedded and written into the same collection queried by the RAG pipeline, real-time events become part of the retrievable knowledge base immediately — closing the loop between the streaming layer and the semantic search layer.

The architecture provides a foundation for future integration with production streaming technologies such as message brokers or event-streaming platforms.

---

# 3. Vector Database & Semantic Knowledge Base

The platform integrates **ChromaDB** as a persistent local vector database for the SDAIA NDI knowledge base.

The document ingestion pipeline:

* Reads the original NDI PDF.
* Extracts document text using PyPDF (written to the Bronze layer).
* Cleans extracted text and splits it into context-preserving chunks (written to the Silver layer).
* Generates multilingual semantic embeddings.
* Stores embeddings in ChromaDB, together with an indexing manifest (the Gold layer).
* Preserves document metadata such as source and page information.
* Uses stable identifiers for indexed chunks.

Real-time governance/risk events validated by the asynchronous pipeline are embedded and upserted into this same collection, so the vector store reflects both the static NDI document and live event data.

The vector database is persisted locally through ChromaDB's persistent storage mechanism.

The embedding model used is:

```text
paraphrase-multilingual-MiniLM-L12-v2
```

The current chunking configuration uses approximately **900 characters per chunk with a 150-character overlap**, helping preserve contextual continuity between neighboring sections.

---

# 4. Advanced RAG Pipeline

The platform implements an evidence-grounded **Retrieval-Augmented Generation (RAG)** architecture designed to improve retrieval relevance and reduce unsupported AI-generated claims.

The pipeline consists of several stages.

### 🔎 Query Expansion

User queries can be expanded with related terminology to improve retrieval coverage, particularly for governance concepts that may appear under different terms in the source documentation.

### 🔀 Hybrid Retrieval

The retrieval stage combines:

* Semantic vector similarity.
* Lexical relevance signals.

This allows the system to retrieve both semantically related evidence and evidence containing important terminology or keywords — across both the indexed NDI document (Gold layer) and any indexed real-time events.

### 🧠 Maximal Marginal Relevance (MMR)

Retrieved candidates are re-ranked using **Maximal Marginal Relevance (MMR)**.

MMR balances relevance and diversity, reducing redundant chunks while improving the coverage of different pieces of supporting evidence.

### 📚 Grounded Generation

The retrieved evidence is provided to the language model as the primary context for generating the response.

The system is designed to:

* Answer using retrieved evidence.
* Avoid unsupported claims.
* Indicate when the available evidence is insufficient.
* Provide source and page references where available.
* Avoid presenting AI-generated responses as official SDAIA assessments or regulatory decisions.

When an external LLM is unavailable, the system can fall back to displaying the retrieved evidence rather than fabricating an answer.

---

# 5. Data Quality Engine

A dedicated **Data Quality Engine** validates incoming events and documents before downstream processing.

The current framework evaluates four main dimensions:

### Schema Validity

Verifies that the required fields exist in incoming records.

### Completeness

Measures missing values across the required fields.

### De-duplication

Detects duplicate event identifiers to prevent repeated records from affecting downstream processing.

### Validity

Validates structured fields such as:

* Event timestamps.
* Numeric values.
* Source information.

Data-quality checks run both on real-time events (before they are indexed into ChromaDB) and are exercised by automated self-tests covering the core data-quality and governance logic.

---

# 6. Data Governance & Role-Based Access Control

The platform incorporates governance metadata and role-based access control to support governance-aware processing.

## Data Catalog & Governance Metadata

Governed assets can be registered with metadata including:

* Data Asset ID
* Data Owner
* Data Domain
* Classification Level
* Retention Period
* Lineage Information
* Allowed Roles

The current lineage model represents the processing path from the original source through:

```text
Source PDF
   ↓
Bronze (raw extracted text)
   ↓
Silver (cleaned, chunked text)
   ↓
Gold (embeddings + indexing manifest)
   ↓
ChromaDB
   ↓
Retrieval
   ↓
AI Response
```

Real-time events follow a parallel, shorter lineage: **Incoming Event → Validation → Normalization → Embedding → ChromaDB**.

## Role-Based Access Control (RBAC)

The platform checks the user's role before allowing access to the NDI knowledge base.

The current implementation includes example roles such as:

* `data_analyst`
* `governance_officer`
* `auditor`

This provides a foundation for implementing more granular governance-aware access controls in future versions.

---

# 🚀 Technology Stack

| Component              | Technology                              |
| ---------------------- | --------------------------------------- |
| Programming Language   | Python                                  |
| User Interface         | Gradio                                  |
| Vector Database        | ChromaDB                                |
| Semantic Embeddings    | HuggingFace SentenceTransformers        |
| Embedding Model        | `paraphrase-multilingual-MiniLM-L12-v2` |
| Data Processing        | Pandas                                  |
| PDF Processing         | PyPDF                                   |
| Async Processing       | Python `asyncio`                        |
| LLM Gateway            | OpenRouter API                          |
| Retrieval Architecture | Hybrid Retrieval + Vector Search + MMR  |
| AI Architecture        | Retrieval-Augmented Generation (RAG)    |

---

# 📁 Storage Structure

The platform uses persistent ChromaDB storage for the semantic knowledge base, plus a layered bronze/silver/gold structure for document processing artifacts.

The main working directory is structured as:

```text
ndi_capstone/
├── bronze/
│   └── <file>_raw.jsonl                 # raw text extracted per page
├── silver/
│   └── <file>_chunks.jsonl              # cleaned, chunked text with stable IDs
├── gold/
│   └── <file>_indexed_manifest.json     # indexed chunk IDs + asset metadata
└── chroma_db/
    └── chroma.sqlite3                   # persistent vector store (NDI chunks + real-time events)
```

## ⚠️ Important: NDI PDF File Path

Before running the document ingestion and RAG pipeline, make sure that the path to the **National Data Index (NDI) PDF** matches the actual location of the uploaded file in your Google Colab environment.

The current implementation uses:

```text
/content/National-Data-Index_v1.0_AR.PDF
```

If the PDF is uploaded to a different location, **update the file path in the relevant notebook cell before running the PDF ingestion process**.

For example, if the file is stored in Google Drive, replace the current `/content/` path with the corresponding Google Drive path.

> **Note:** The file path is environment-specific and may need to be changed when the notebook is executed in a different Colab session or environment.

---

# 💬 Interactive AI Governance Assistant — Output

The following screenshot demonstrates the interactive **NDI Governance & RAG Chat** interface and an example of an evidence-grounded response with retrieved source references.

![NDI Governance & RAG Chatbot](chatbot.png)
![NDI Governance & RAG Chatbot1](chatbot1.png)
