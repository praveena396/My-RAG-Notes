# Loading PDF Files (LangChain)

## Why PDFs Need Separate Loaders

PDFs are one of the most common real-world data sources for RAG (companies have large PDF archives), but unlike plain text files, PDFs have complex internal structure (layout, fonts, images, encoding) — so specialized loader libraries are needed to extract clean, structured content.

## 1. PyPDFLoader (most common, recommended default)

```python
from langchain_community.document_loaders import PyPDFLoader

try:
    pdf_loader = PyPDFLoader("data/pdf/attention.pdf")
    pdf_docs = pdf_loader.load()
except Exception as e:
    print(f"Error: {e}")

print(len(pdf_docs))                    # number of pages loaded
print(pdf_docs[0].page_content[:100])   # preview first page content
print(pdf_docs[0].metadata)             # producer, creator, author, page, source, etc.
```

**Key behavior:**
- One `Document` is created per page (not per file) — e.g. a 15-page PDF → 15 `Document` objects
- `metadata` includes rich PDF-specific fields: producer, creator, author, page number, source path
- Preserves page numbers, which is useful for citation later

## 2. PyMuPDFLoader (faster alternative)

```python
from langchain_community.document_loaders import PyMuPDFLoader

pymupdf_loader = PyMuPDFLoader("data/pdf/attention.pdf")
pymupdf_docs = pymupdf_loader.load()
```

Requires separate install:

```bash
uv add pymupdf
```

## 3. Comparison

| Loader | Strengths | Weaknesses | Best for |
|---|---|---|---|
| PyPDFLoader | Simple, reliable, preserves page numbers | Basic text extraction — struggles with images/encoded characters | Most standard text-based PDFs |
| PyMuPDFLoader | Fast processing, good text + image extraction | — | When speed and richer extraction matter |

**General guidance:** start with `PyPDFLoader` for standard PDFs; use `PyMuPDFLoader` when speed or image-aware extraction is needed.

A third option, `UnstructuredPDFLoader`, exists for more complex/mixed-content PDFs — worth a separate note since it handles layout-heavy or non-standard PDFs differently.

## Core Concept Recap

Regardless of loader used, output is still the same standardized `Document` structure (`page_content` + `metadata`) — this consistency is what lets the same downstream pipeline (text splitter → embeddings → vector store) work identically regardless of source file type.
