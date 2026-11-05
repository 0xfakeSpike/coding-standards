# RAG engineering standards

This document sets rules for indexing, chunking, and evaluation in retrieval-augmented generation. For using search engines, see [Document and search storage](../data/document-and-search.md). For agents and prompt injection protection, see [Agent engineering standards](agents.md).

## Choosing an approach

### Decide first whether you need RAG [Recommended]

When the knowledge base is small enough to fit entirely in context and the cost and latency are acceptable, put it in directly and do not build a retrieval pipeline.

**Source:** Anthropic Contextual Retrieval

## Indexing

### The index is rebuildable derived data [Recommended]

Source documents are the single source of truth. The index stores only content that can be regenerated from them. The index records the chunking configuration and embedding model that produced it. Query vectors and document vectors must come from the same model.

**Why:** Vectors from different models are not comparable. Mixing them raises no error, and results just quietly get worse.

### Use lexical and vector retrieval together [Recommended]

Vector retrieval is good at semantically similar content but can miss exact matches. Lexical retrieval is good at identifiers and technical terms. Merge the two result sets, rerank, and give the model a small number of the most relevant chunks. Determine the recall count and final chunk count through evaluation.

**Why:** Too few chunks easily miss information. Too many distract the model and add cost.

**Source:** Anthropic Contextual Retrieval

### Filter permissions at retrieval time [Required]

When data has tenants or access levels, every chunk carries this information, and multi-tenant data is partitioned by permission. The caller passes in permission conditions based on the current user, and they take effect inside the search. Do not filter with the model or prompts after results return.

**Why:** With a shared vector store, one tenant's content can appear in another tenant's query results.

**Example:**

```json
{"query": "refund processing time", "filter": {"tenant_id": "t_9", "access_level": ["public", "staff"]}, "top_k": 50}
```

**Source:** OWASP LLM Top 10

### Validate content before ingestion [Recommended]

Accept documents only from trusted sources and validate them before ingestion. Text extraction must be able to expose hidden content, such as white text and invisible layers.

**Why:** Instructions hidden in documents enter the context through retrieval and affect model output.

**Source:** OWASP LLM Top 10

### Keep the index in sync with source documents [Required]

**Applies to:** production service

Use content hashes to detect document changes and process only the changed parts. When a document is updated, replace all its old chunks. When a document is deleted, delete all its chunks at the same time.

**Why:** Leftover chunks keep getting cited, and deleted sensitive content can leak.

## Chunking

### Chunk by document structure [Recommended]

Prefer chunking by headings, paragraphs, tables, and code blocks. Do not cut in the middle of a sentence or table. Keep chunk size and overlap in configuration, and set them by comparing against an evaluation set, not by copying rule-of-thumb values.

**Source:** Anthropic Contextual Retrieval

### Keep context in chunks [Recommended]

For chunks whose meaning is unclear apart from the original text, add the document title, section path, or a short note at indexing time.

**Example:** "Revenue grew 3% in the quarter" does not show the subject on its own. Add the company and quarter at indexing time.

**Source:** Anthropic Contextual Retrieval

### Keep source information in chunks [Recommended]

Each chunk stores the document identifier and document version. When you need to locate the original text, also store the section and position.

**Why:** Without source information, stale chunks cannot be identified after a document update.

**Example:**

```json
{"chunk_id": "c_18", "document_id": "d_3", "document_version": "a91f"}
```

## Citations

### Validate citations from the model [Required]

When the model outputs a citation, confirm it corresponds to a chunk actually provided this time, and do not show ones that do not exist. The system generates links from chunk sources. Do not show URLs output by the model directly.

**Why:** Models can fabricate links, and injected content can also use links to steer users to malicious addresses.

## Evaluation

### Evaluate retrieval and generation separately [Recommended]

Looking only at the final answer cannot tell you whether the problem is in retrieval or generation. Cover at least the four items in the table below, with metrics computed by a tool such as Ragas or by your own scripts.

| Stage | Metric | Meaning |
|---|---|---|
| Retrieval | Context recall | How many claims in the reference answer are supported by retrieval results |
| Retrieval | Context precision | Whether relevant chunks rank near the top |
| Generation | Faithfulness | How many claims in the answer are supported by retrieved content |
| Generation | Answer relevancy | Whether the answer is on topic, not whether the facts are correct |

When tuning only retrieval, you can also compare top-K recall directly, which requires labeling relevant chunks.

**Tooling:** Ragas

**Source:** Ragas Docs, Anthropic Contextual Retrieval

### Maintain a fixed evaluation set [Recommended]

The evaluation set consists of real questions, reference answers, and relevant documents, and must include unanswerable and easily confused questions. A few dozen items are enough to start. Keep adding error cases found in production.

**Why:** Early changes differ greatly in effect, and a small sample is enough to see it. Without a fixed set, two changes cannot be compared.

**Source:** Anthropic: Multi-Agent Systems, Ragas Docs

### Attach evaluation results to changes [Recommended]

When adjusting chunking, the embedding model, recall count, reranking, or prompts, compare metrics before and after the change on the same evaluation set. For how to organize tests, see [Backend testing](../testing.md).

**Source:** Anthropic Contextual Retrieval

### Validate model scoring with human review [Recommended]

Model-based metrics call a model, so scores fluctuate. Fix the model and prompt used for scoring, and spot-check by hand regularly. Human review must also cover edge cases automated evaluation cannot catch.

**Source:** Ragas Docs, Anthropic: Multi-Agent Systems

### Record the retrieval process in production [Recommended]

Each retrieval produces a `gen_ai.retrieval` span that records `gen_ai.retrieval.top_k`, duration, and hit count. Do not record query text and document content by default, and enable it explicitly when needed. Retrieval logs are also used to detect suspicious access.

**Source:** OTel GenAI Semantic Conventions, OWASP LLM Top 10
