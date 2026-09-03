# JSON Parsing & Processing

## Why JSON Matters for RAG

JSON is extremely common in real-world pipelines — it's the standard format APIs return data in. Unlike CSV (flat rows) or plain text, JSON is often nested (objects within objects, arrays within objects), which requires a way to target specific nested fields rather than dumping the whole structure.

## Method 1: JSONLoader with JQ Schema (built-in, targeted extraction)

```python
from langchain_community.document_loaders import JSONLoader

loader = JSONLoader(
    file_path="data/json_files/company_data.json",
    jq_schema=".employees[]",   # targets nested "employees" array
    text_content=False           # False = keep full JSON object per item, not just plain text
)
employee_docs = loader.load()
```

Requires: `jq` library (`uv add jq`)

**Key concept — `jq_schema`:** JQ is a query language for navigating and extracting specific parts of a JSON structure — similar in spirit to XPath for XML or a CSS selector for HTML. `.employees[]` means: "go into the `employees` key, and treat it as a list — extract each item separately." This lets you pull out only the relevant nested section instead of ingesting the entire JSON blob as one chunk.

**`text_content=False` vs `True`:**
- `True` → extracts only plain text content from matched fields
- `False` → keeps the full JSON object (with all its structure/keys) as the `Document`'s content

Result: one `Document` created per matched item (e.g. one employee = one `Document`) — same "one record = one Document" pattern seen with `CSVLoader`'s row-based approach.

## Method 2: Custom JSON Processing (manual control)

```python
import json
from typing import List
from langchain_core.documents import Document

def process_json_custom(file_path: str) -> List[Document]:
    with open(file_path, 'r', encoding='utf-8') as f:
        data = json.load(f)

    documents = []
    for employee in data.get('employees', []):
        content = f"Name: {employee.get('name')}\n"
        content += f"Skills: {employee.get('skills')}\n"

        for project in employee.get('projects', []):
            content += f"Project: {project}\n"

        doc = Document(
            page_content=content,
            metadata={
                "source": file_path,
                "employee_id": employee.get('id'),
                "name": employee.get('name')
            }
        )
        documents.append(doc)

    return documents
```

**Key concept — `.get()` on dictionaries:** using `dict.get('key', default)` instead of `dict['key']` avoids crashes if a field is missing in some records — important for real-world JSON, which is often inconsistent across entries (some employees might lack a `projects` field, for example).

**Why write custom logic instead of relying on JQ schema alone:** full control over how nested fields (like a list of projects inside each employee) get formatted into readable `page_content`, plus precise control over which fields become metadata — same rationale as custom CSV processing.

## JSON vs CSV Parsing — Conceptual Parallel

| | CSV | JSON |
|---|---|---|
| Structure | Flat rows/columns | Often nested (objects, arrays within objects) |
| Built-in loader | CSVLoader (row-based) | JSONLoader + jq_schema (targeted extraction) |
| Targeting specific data | Not usually needed (flat) | Often essential (jq_schema selects a nested branch) |
| Custom processing benefit | Preserves cross-row relationships | Preserves nested relationships (e.g. employee → their projects) |

Consistent principle across the whole course so far: built-in loaders are fast defaults; custom processing functions are needed whenever you want the resulting `page_content`/`metadata` to reflect the actual relationships in the data — not just a flattened, generic dump.

## Big Picture

JSON parsing reinforces the same core skill developed across every file type: reading nested/structured source data and deliberately deciding what becomes `page_content` vs `metadata`, rather than blindly using default extraction. This decision directly impacts retrieval quality later, since chunk content and metadata design determine what a similarity search can actually match against.
