# Full-Fledged RAG System Architecture

This document outlines the end-to-end architecture and workflow for a robust Retrieval-Augmented Generation (RAG) system.

## 1. Data Ingestion & Preprocessing
The foundation of RAG is the quality of the data it retrieves.
- **Data Loading:** Extracting text from various sources (PDFs, Markdown, Databases, APIs).
- **Cleaning:** Removing noise (HTML tags, boilerplate, special characters) and normalizing text.
- **Chunking:** Splitting documents into smaller, manageable pieces. 
    - *Strategies:* Fixed-size, overlapping windows, or semantic chunking (splitting based on meaning/structure).
- **Metadata Attachment:** Adding source URLs, timestamps, or categories to chunks to improve filtering during retrieval.

## 2. Embedding Generation
Transforming text chunks into numerical representations that capture semantic meaning.
- **Model Selection:** Using models like OpenAI's `text-embedding-3-small`, HuggingFace's `all-MiniLM-L6-v2`, or Cohere's multilingual embeddings.
- **Consistency:** Ensuring the same embedding model is used for both indexing and runtime queries.

## 3. Vector Database Storage
Storing embeddings in a specialized database for efficient similarity searches.
- **Indexing:** Using algorithms like HNSW (Hierarchical Navigable Small World) or IVF (Inverted File Index) for fast high-dimensional search.
- **Providers:** Pinecone, Weaviate, Milvus, ChromaDB, or pgvector.
- **Hybrid Storage:** Combining vector search with traditional keyword search (BM25) for better accuracy.

## 4. Retrieval Pipeline
Finding the most relevant information based on a user query.
- **Query Transformation:** Rewriting the user's query to be more search-friendly (e.g., Multi-Query, HyDE - Hypothetical Document Embeddings).
- **Similarity Search:** Calculating distance (Cosine Similarity, Euclidean Distance) between the query vector and stored vectors.
- **Reranking:** Passing the top-K results through a more expensive "cross-encoder" model (like Cohere Rerank) to prioritize the most relevant context.

## 5. Augmentation & Prompt Engineering
Contextualizing the LLM with retrieved data.
- **Prompt Construction:** Creating a template that includes the retrieved chunks as "Context" and the original question as "Question."
- **Context Window Management:** Ensuring the total tokens of the prompt and retrieved chunks fit within the LLM's limits.

## 6. Generation (LLM Inference)
The final step where the model synthesizes an answer.
- **Model Selection:** GPT-4o, Claude 3.5 Sonnet, or local models like Llama 3.
- **Instruction Following:** Explicitly telling the model to "Only use the provided context" or "State if the information is missing."

## 7. Evaluation & Observability (The "Full-Fledged" Layer)
Ensuring the system performs reliably over time.
- **RAGAS/TruLens:** Frameworks to measure:
    - *Faithfulness:* Is the answer derived solely from the context?
    - *Relevancy:* Does the context actually answer the question?
    - *Answer Correctness:* Accuracy against a ground-truth dataset.
- **Logging:** Tracking queries, retrieved chunks, and generated responses for debugging and refinement.

---

## System Components & File Structure

A production-grade RAG system should be modularized to handle each stage independently.

```text
rag-system/
├── data/                   # Raw documents (PDFs, JSON, CSV)
├── src/
│   ├── ingestion/          # Data loading, cleaning, and chunking logic
│   ├── embedding/          # Logic for connecting to embedding APIs/Models
│   ├── vector_store/       # Database connection and indexing scripts
│   ├── retrieval/          # Similarity search and reranking logic
│   ├── generator/          # LLM prompt engineering and inference
│   └── eval/               # Evaluation scripts (RAGAS, metrics)
├── tests/                  # Unit and integration tests
├── config.yaml             # Configuration (Models, Chunk size, DB URLs)
├── requirements.txt        # Dependencies (LangChain, LlamaIndex, etc.)
└── main.py                 # Entry point for the RAG query pipeline
```

## Technical Implementation Guide

### 1. Ingestion Phase
- **Chunk Size:** 512-1024 tokens is typical for general use.
- **Chunk Overlap:** 10-20% overlap ensures context isn't lost at the boundaries of chunks.

### 2. Retrieval Phase
- **K-Value:** Typically retrieve 5-10 chunks.
- **Reranking:** If you retrieve 20 chunks, use a Cross-Encoder to rerank them and take the top 5 for the final prompt. This significantly improves precision.

### 3. Generation Phase
- **Temperature:** Set to `0.0` or `0.1` for RAG to ensure factual and consistent answers rather than creative ones.
- **System Prompt:** "You are a helpful assistant. Use ONLY the provided context to answer the user's question. If you don't know the answer based on the context, say you don't know."

### 4. Evaluation Phase (RAGAS Metrics)
- **Faithfulness:** (Retrieved Context ∩ Answer) / Answer.
- **Answer Relevance:** (Query ∩ Answer) / Answer.
- **Context Precision:** (Retrieved Context ∩ Ground Truth) / Retrieved Context.

---

## High-Level Workflow Diagram
1. **User Query** -> **Embedding Model** -> **Query Vector**
2. **Query Vector** -> **Vector DB** -> **Top-K Chunks**
3. **Top-K Chunks** -> **Reranker** -> **Refined Chunks**
4. **Refined Chunks** + **User Query** -> **LLM Prompt**
5. **LLM** -> **Final Answer**
