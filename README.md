# Enterprise RAG Knowledge Assistant

A production-style **Retrieval-Augmented Generation (RAG)** application that ingests enterprise documents and generates **source-attributed responses from retrieved document context** through a conversational interface.

**Technology stack:** FastAPI, Streamlit, LangChain, ChromaDB, FastEmbed, Google Gemini, pytest, GitHub Actions

## Project Overview

This system implements an **enterprise knowledge assistant** that converts unstructured policy and handbook content into a searchable vector index. Users upload documents (PDF, TXT, DOCX), submit document-specific queries, and receive responses synthesized from semantically retrieved passages with **citation metadata** (source file and page, where applicable).

The solution separates concerns across a REST API layer, an ingestion/embedding pipeline, a vector store, and a lightweight web dashboard for document management and Q&A.

---

## Business Problem

Organizations store critical operational knowledge in distributed documents: HR handbooks, security policies, finance guidelines, and clinical/compliance manuals. Employees face:

- **Information fragmentation** across PDFs, Word files, and plain-text repositories
- **High lookup latency** when policies change or span multiple documents
- **Compliance risk** when answers are guessed instead of sourced from authoritative text
- **Poor scalability** of manual FAQ maintenance and helpdesk triage

This assistant addresses the gap by providing **on-demand semantic retrieval** over private document corpora, reducing time-to-answer while improving **traceability** through source-linked responses.

---

## Architecture

```mermaid
flowchart LR
  subgraph Client
    UI[Streamlit Dashboard]
  end

  subgraph API
    FAST[FastAPI REST Layer]
  end

  subgraph Ingestion
    LOAD[Document Loaders]
    SPLIT[Text Chunking]
    EMB[FastEmbed Embeddings]
  end

  subgraph Storage
    CHROMA[(ChromaDB Vector Store)]
    FILES[(Upload Storage)]
  end

  subgraph Inference
    RET[Semantic Retrieval top-k]
    LLM[Google Gemini LLM]
  end

  UI -->|HTTP| FAST
  FAST --> LOAD
  LOAD --> SPLIT
  SPLIT --> EMB
  EMB --> CHROMA
  FAST --> FILES
  FAST --> RET
  RET --> CHROMA
  RET --> LLM
  LLM --> FAST
  FAST --> UI
```

| Layer | Component | Responsibility |
|-------|-----------|----------------|
| Presentation | Streamlit (`frontend/`) | Upload UX, chat interface, source inspection |
| API | FastAPI (`app/main.py`) | Ingestion endpoints, query orchestration, health checks |
| Ingestion | LangChain loaders + splitter (`app/rag/ingest.py`) | Parse documents, chunk text, attach metadata |
| Embeddings | FastEmbed `BAAI/bge-small-en-v1.5` | Local dense vector encoding (no embedding API key) |
| Vector DB | ChromaDB | Persistent similarity search over document chunks |
| Generation | Google Gemini (`gemini-2.5-flash`) | Response synthesis from retrieved document context |

**Project structure**

```text
app/                 FastAPI backend and RAG pipeline
  main.py            REST endpoints
  rag/ingest.py      Document parsing and chunking
  rag/vectorstore.py Embedding and vector persistence
  rag/chain.py       Retrieval + LLM orchestration
frontend/            Streamlit dashboard
tests/               Unit and API integration tests
data/sample_docs/    Seed corpus for demos
.github/workflows/   CI pipeline
```

---

## Workflow

1. **Document ingestion** - User uploads PDF, TXT, or DOCX via dashboard or `POST /upload`.
2. **Parsing** - Loaders extract text; PDF page numbers are preserved in metadata.
3. **Chunking** - Recursive character splitting (800 characters, 120-character overlap) creates retrieval units.
4. **Embedding** - Chunks are encoded with FastEmbed and indexed in ChromaDB.
5. **Query** - User submits a document-specific query through the chat interface or `POST /ask`.
6. **Retrieval** - Top-k semantic search (`k=4`) fetches the most relevant chunks.
7. **Augmented generation** - Gemini synthesizes an answer using only retrieved context.
8. **Response delivery** - Answer and source excerpts are returned with file/page attribution.

If the LLM is unavailable, rate-limited, slow, or unconfigured, the system falls back to **retrieval-only mode** and surfaces ranked source passages (with page numbers when available) instead of returning an error.

---

## Dataset

The repository includes a **synthetic enterprise corpus** for demonstration and testing:

| Document | Domain | Content focus |
|----------|--------|---------------|
| `healthcare_handbook.txt` | Healthcare | Remote work, HIPAA, clinical policies |
| `hr_employee_handbook.txt` | Human Resources | PTO, benefits, workplace policies |
| `it_security_policy.txt` | Information Security | Passwords, VPN, data handling |
| `finance_expense_policy.txt` | Finance | Travel limits, reimbursement rules |

**Supported formats:** PDF, TXT, DOCX  
**Runtime storage:** Uploaded files are written to `data/uploads/`; the vector index is written to `data/chroma_db/`.

Users can extend the knowledge base with any domain-specific documents using the same ingestion pipeline.

---

## Running the Application

Use this procedure to run the application locally for development and demonstration.

**Prerequisites**

- Python 3.12
- pip
- A Google Gemini API key from [Google AI Studio](https://aistudio.google.com/apikey)
---
