# Document Loaders

**Document Loaders** are the first stage of the RAG pipeline. They take data from an external source and convert it into LangChain's standard `Document` format so that everything downstream — splitters, embeddings, vector stores, retrievers — can work with it uniformly.

## Where document loaders fit

The full RAG ingestion pipeline:

```text
         Data
          │
          ▼
   Document Loader     ← this chapter
          │
          ▼
      Documents
          │
          ▼
    Text Splitter
          │
          ▼
       Chunks
          │
          ▼
     Embeddings
          │
          ▼
     Vector Store
          │
          ▼
      Retriever
          │
          ▼
 Relevant Documents
          │
          ▼
         LLM
          │
          ▼
        Answer
```

Loaders are the **entry point** — everything that follows depends on their output.

## What is a Document Loader?

A Document Loader takes an external data source and returns a list of LangChain `Document` objects:

```text
External Data
      ↓
Document Loader
      ↓
LangChain Document(s)
```

Your source data might be a PDF, CSV, JSON file, plain text, HTML, a Word document, a database, cloud storage, a wiki, or anything else. The loader's job is to normalize all of these into one shape.

## The LangChain `Document`

A LangChain `Document` has two important fields:

```python
Document(
    page_content="Some text...",
    metadata={
        "source": "example.pdf",
        "page": 5
    }
)
```

### `page_content`

The actual text — the part that will eventually be embedded and searched.

```text
"Employees receive 20 days of annual leave..."
```

### `metadata`

Structured information *about* the text:

```python
{
    "source": "employee_handbook.pdf",
    "page": 12
}
```

Metadata is extremely useful later for:

- **Filtering** — restrict retrieval to a subset (e.g., a specific department).
- **Citations** — show the user where an answer came from.
- **Identifying the source** — debugging retrieval quality.
- **Debugging** — pinpoint which document caused a wrong answer.
- **Access control** — enforce per-user or per-tenant boundaries.

Metadata is not decorative — it is often the difference between a demo and a production RAG system.

## Example: PDF loader

LangChain provides loaders for many formats. A typical PDF example:

```python
from langchain_community.document_loaders import PyPDFLoader

loader = PyPDFLoader("document.pdf")
documents = loader.load()
```

Result:

```text
[
    Document(... page=1),
    Document(... page=2),
    Document(... page=3),
    ...
]
```

Depending on the loader, each page or logical section may become a separate `Document`.

## Common loader categories

LangChain supports a wide range of source types. A conceptual grouping:

```text
                Document Loaders
                       │
     ┌─────────────────┼─────────────────┐
     ↓                 ↓                 ↓
 File-based        Web-based        System-based
     │                 │                 │
  PDF, CSV,       HTML pages,       Databases,
  JSON, TXT,      web scraping,     APIs, cloud
  Markdown,       Sitemaps          storage, wikis
  DOCX, ...
```

The exact loader classes vary, but every one of them ends with the same output: a list of `Document` objects.

## Why do we need a Document abstraction at all?

Without a common shape, each downstream component would need custom handling for every source. With a common `Document`:

```text
PDF Loader   ─┐
Web Loader   ─┤
CSV Loader   ─┼──→ Document ──→ Splitter ──→ Embedding ──→ Store
JSON Loader  ─┤
API Loader   ─┘
```

Splitters, embeddings, and retrievers all know how to work with a `Document`, regardless of where it came from. This is the same standardization idea as [Runnables](./09-runnables.md), just applied to data instead of components.

## Loading is not splitting

A very common source of confusion: **loading and splitting are separate stages.**

- A loader turns a source file/URL/API into `Document`s — often one per page or per record.
- A splitter breaks those `Document`s into smaller **chunks** suitable for embedding.

Even if a PDF has 200 pages and the loader returns 200 Documents, that is still usually **too large** to embed directly. The next chapter — [Text Splitters](./12-text-splitters.md) — is where the actual chunking happens.

| Component           | Input             | Output             |
| ------------------- | ----------------- | ------------------ |
| **Document Loader** | External source   | `Document`s        |
| **Text Splitter**   | `Document`s       | Smaller `Document` chunks |

## When to reach for a Document Loader

Any time you want to make external data available to an LLM via RAG:

- Internal documentation search.
- Q&A over PDFs or long HTML pages.
- Ingesting reports, transcripts, or user-uploaded files.
- Building a knowledge base out of an existing wiki or CMS.

If your data lives outside your codebase, a Document Loader is the way in.

## Practical considerations

- **Character encoding** — text files can arrive in many encodings; some loaders let you specify one.
- **File structure** — a well-structured PDF or Markdown file loads much more cleanly than a scanned PDF of images.
- **Metadata quality** — spend time on metadata early; retrieving without it is painful later.
- **Idempotency** — for large ingestion pipelines, be prepared to re-run loaders when sources change.
- **Rate limits** — web and API loaders often need throttling or caching.

## Where does the Document Loader belong in the mental model?

Think of building a digital library:

> **Document Loader = "Get the books into the library."**

Everything else in the RAG pipeline is about organizing, indexing, and searching the books — but until the loader has done its job, there is nothing to organize.

## Key takeaways

- A Document Loader turns any external data source into LangChain `Document` objects.
- A `Document` has `page_content` and `metadata`; both matter.
- Metadata drives filtering, citations, debugging, and access control.
- Loading is separate from splitting — a loader gives you `Document`s; a splitter chunks them.
- The `Document` abstraction is what lets a splitter, embedding, or retriever work with any source uniformly.

### Final mental model

```text
      External Data
          │
          ▼
   Document Loader
          │
          ▼
  ┌───────────────┐
  │   Document    │
  ├───────────────┤
  │ page_content  │
  │   metadata    │
  └───────────────┘
          │
          ▼
    Text Splitter
```

---

Next: [Text Splitters](./12-text-splitters.md) — how large documents become embedding-sized chunks.
