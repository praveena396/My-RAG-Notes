# RAG Data Ingestion — Reading Text Files

## Core Concept

Every document loader in LangChain converts a raw source (txt, pdf, etc.) into a `Document` object with two components:

- `page_content` → the actual text
- `metadata` → info about where it came from (e.g. file path)

This is the standard interface every downstream RAG step (chunking, embedding) expects — regardless of source type.

## 1. TextLoader — Single File

Reads one text file and returns it as a list containing one `Document`.

```python
from langchain_community.document_loaders import TextLoader

loader = TextLoader("data/text_files/python_intro.txt", encoding="utf-8")
documents = loader.load()

documents[0].page_content   # full text of the file
documents[0].metadata       # {'source': 'data/text_files/python_intro.txt'}
```

**Key behavior:**
- `.load()` always returns a list, even for one file — keeps the interface consistent with multi-file loaders
- `metadata['source']` auto-populates with the file path — important later for citing sources in RAG answers
- Always pass `encoding="utf-8"` explicitly to avoid platform-dependent encoding bugs

## 2. DirectoryLoader — Multiple Files

Applies a specified loader class across every matching file in a directory, batching the results into one list of `Document`s.

```python
from langchain_community.document_loaders import DirectoryLoader

dir_loader = DirectoryLoader(
    path="data/text_files",
    glob="**/*.txt",
    loader_cls=TextLoader,
    loader_kwargs={'encoding': 'utf-8'},
    show_progress=True
)

documents = dir_loader.load()
```

**Parameter roles:**

| Parameter | Role |
|---|---|
| `path` | Root directory to scan |
| `glob` | File-matching pattern (`**/*.txt` = all .txt files, including subfolders) |
| `loader_cls` | Which loader to apply per matched file (must match file type) |
| `loader_kwargs` | Arguments forwarded to each individual loader instance |
| `show_progress` | Progress bar during load |

Conceptually: `DirectoryLoader` is a wrapper — it doesn't parse files itself, it just discovers files matching the pattern and delegates parsing to `loader_cls` for each one.

## 3. Design Trade-off: Single vs Batch Loading

| | TextLoader | DirectoryLoader |
|---|---|---|
| Scope | One file | Many files, same type |
| Error handling | Fails on that one file | One bad file can affect the whole batch (limited per-file error isolation) |
| Memory | Low | Can be high for large directories (loads everything into memory) |
| Use case | Known single source | Bulk ingestion of homogeneous file types |

`DirectoryLoader` trades convenience for control — good for quick bulk ingestion, but in production you'd typically want per-file error handling and possibly streaming/batching rather than loading an entire large directory into memory at once.

## Big Picture

This is the ingestion stage of a RAG pipeline — the very first step before chunking and embedding. The consistent `Document` structure (`page_content` + `metadata`) is what makes every later stage (splitter, embedder, vector store) source-agnostic — the pipeline doesn't need to know or care whether the original data came from a `.txt`, PDF, or web page, since it's all normalized into the same object shape by this point.
