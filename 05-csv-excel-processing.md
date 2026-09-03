# CSV & Excel Processing (Structured Data)

## Core Concept: Structured vs. Unstructured Data

Everything covered so far (txt, PDF, Word) was unstructured data — free-flowing text. CSV/Excel are structured data — organized into rows/columns with a defined schema. This changes the parsing approach fundamentally: instead of extracting "text," you're deciding how to represent tabular records as retrievable text chunks.

## Method 1: CSVLoader (row-based, built-in)

```python
from langchain_community.document_loaders import CSVLoader

loader = CSVLoader(
    file_path="data/structured_files/product.csv",
    encoding="utf-8",
    csv_args={
        "delimiter": ",",
        "quotechar": '"'
    }
)
csv_docs = loader.load()
```

**Behavior:** creates one `Document` per row. Each row's columns get formatted into `page_content` as key-value text, and `metadata` automatically includes source + row index.

**Core limitation:** row-by-row conversion treats each record in isolation — any relationships between rows (e.g. a product referencing a category defined elsewhere) are lost, since each chunk has no awareness of other rows.

## Method 2: Custom CSV Processing (more control)

Building your own processor instead of relying on the built-in loader lets you control exactly what goes into `page_content` and `metadata`.

```python
import pandas as pd
from typing import List
from langchain_core.documents import Document

def process_csv_intelligently(file_path: str) -> List[Document]:
    df = pd.read_csv(file_path)
    documents = []

    for idx, row in df.iterrows():
        content = f"Product: {row['product']}, Category: {row['category']}, Price: {row['price']}, Info: {row['description']}"

        doc = Document(
            page_content=content,
            metadata={
                "source": file_path,
                "row_index": idx,
                "product": row['product'],
                "category": row['category'],
                "price": row['price']
            }
        )
        documents.append(doc)

    return documents
```

**Key concept — `df.iterrows()`:** a pandas method that lets you loop through a DataFrame row by row, giving you both the row's index and its data (similar in spirit to `enumerate()`, but pandas-specific).

**Why write a custom processor instead of using `CSVLoader` directly:** the built-in loader gives you a structure, but a custom function gives you control over how content is phrased and which fields become metadata — critical when you want retrieval to work well for domain-specific Q&A rather than generic row dumps.

## CSV Strategy Comparison

| Strategy | Behavior | Pros | Cons | Best for |
|---|---|---|---|---|
| Row-based (CSVLoader) | One row → one Document, auto-generated | Simple, fast, no code needed | Loses relationships between rows, generic phrasing | Quick record lookups |
| Intelligent/custom processing | Manually crafted content + metadata per row (or grouped rows) | Preserves relationships, richer metadata, can create summaries | Requires custom code | Q&A systems, when context/relationships matter |

Same trade-off as structural vs. character-based chunking (Word doc notes): automatic/generic parsing is fast but loses nuance; custom parsing costs more effort but produces chunks better suited to how the data will actually be queried.

## Excel Processing

### Method 1: Manual processing with pandas (full control)

```python
import pandas as pd
from langchain_core.documents import Document

def process_excel_with_pandas(file_path: str) -> List[Document]:
    excel_file = pd.ExcelFile(file_path)
    documents = []

    for sheet_name in excel_file.sheet_names:
        df = pd.read_excel(file_path, sheet_name=sheet_name)
        sheet_content = df.to_string()  # or custom formatting per row

        doc = Document(
            page_content=sheet_content,
            metadata={
                "source": file_path,
                "sheet_name": sheet_name,
                "num_rows": len(df)
            }
        )
        documents.append(doc)

    return documents
```

**Key concept:** Excel files can have multiple sheets — a structural dimension CSV doesn't have. Processing needs to iterate over `sheet_names` and can choose to treat each sheet as one `Document`, or go row-by-row within each sheet (same trade-off as CSV).

### Method 2: UnstructuredExcelLoader (built-in, structure-aware)

```python
from langchain_community.document_loaders import UnstructuredExcelLoader

loader = UnstructuredExcelLoader("data/structured_files/inventory.xlsx")
docs = loader.load()
```

**Trade-off:** handles complex Excel features (merged cells, formatting) and preserves more structural info automatically, but is slower — the same speed-vs-structure trade-off seen throughout (Word, PDF, CSV, Excel all repeat this pattern).
