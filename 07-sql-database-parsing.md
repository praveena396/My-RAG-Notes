# SQL Database Parsing & Processing

## Why SQL Databases Need Their Own Approach

Companies commonly store data in SQL databases (structured, tabular) rather than files. Unlike a NoSQL store like MongoDB — which returns key-value pairs conceptually similar to JSON — a SQL database requires querying tables and joining across them before that data can become a retrievable `Document`. The end goal is the same as every other source type: get to `page_content` + `metadata`, so downstream chunking/embedding stays source-agnostic.

## Setup: A Sample SQLite Database

```python
import sqlite3
import os

os.makedirs("data/databases", exist_ok=True)

connection = sqlite3.connect("data/databases/company.db")
cursor = connection.cursor()

cursor.execute("""
CREATE TABLE IF NOT EXISTS employees (
    id INTEGER PRIMARY KEY,
    name TEXT,
    role TEXT,
    department TEXT,
    salary REAL
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS projects (
    id INTEGER PRIMARY KEY,
    name TEXT,
    status TEXT,
    budget REAL,
    lead_id INTEGER
)
""")

employees = [
    (1, "John Doe", "Senior Developer", "Engineering", 95000),
    # ...
]
projects = [
    (1, "Project Alpha", "Active", 50000, 1),
    # ...
]

cursor.executemany("INSERT OR REPLACE INTO employees VALUES (?,?,?,?,?)", employees)
cursor.executemany("INSERT OR REPLACE INTO projects VALUES (?,?,?,?,?)", projects)

connection.commit()
connection.close()
```

**Key concept:** `projects.lead_id` references `employees.id` — a foreign-key-style relationship. This is what makes SQL fundamentally different from flat CSV rows: records can be related to each other, and a good document strategy should be able to preserve that relationship rather than flatten it away.

## Method 1: SQLDatabase Utility (quick inspection)

```python
from langchain_community.utilities import SQLDatabase

db = SQLDatabase.from_uri("sqlite:///data/databases/company.db")

db.get_usable_table_names()   # ['employees', 'projects']
db.get_table_info()           # DDL + sample rows for each table
```

Useful for a quick look at schema and sample data, but this doesn't produce `Document` objects — it's inspection, not ingestion.

## Method 2: Custom SQL-to-Documents Conversion

Gives full control over how each table becomes retrievable content.

```python
import sqlite3
from typing import List
from langchain_core.documents import Document

def sql_to_documents(db_path: str) -> List[Document]:
    """Convert SQL database to documents with context."""
    connection = sqlite3.connect(db_path)
    cursor = connection.cursor()
    documents = []

    cursor.execute("SELECT name FROM sqlite_master WHERE type='table'")
    tables = cursor.fetchall()

    for table in tables:
        table_name = table[0]

        cursor.execute(f"PRAGMA table_info({table_name})")
        columns = [col[1] for col in cursor.fetchall()]

        cursor.execute(f"SELECT * FROM {table_name}")
        rows = cursor.fetchall()

        table_content = f"Table: {table_name}\nColumns: {columns}\n\nSample records:\n"
        for row in rows:
            table_content += f"{row}\n"

        documents.append(Document(
            page_content=table_content,
            metadata={
                "source": db_path,
                "table_name": table_name,
                "num_rows": len(rows),
                "type": "sql_table"
            }
        ))

    connection.close()
    return documents

documents = sql_to_documents("data/databases/company.db")
```

**Behavior:** one `Document` per table — content includes column names and every row, metadata captures table name, row count, and source path. This is the same "one logical unit = one Document" pattern seen with CSV rows or JSON array items, just applied at the table level instead of the row level.

## Method 3: Relationship / Join Documents

A per-table dump loses the relationship between `employees` and `projects`. A join query captures it directly:

```python
def relationship_documents(db_path: str) -> List[Document]:
    connection = sqlite3.connect(db_path)
    cursor = connection.cursor()

    cursor.execute("""
        SELECT e.name, e.role, p.name AS project_name
        FROM projects p
        JOIN employees e ON e.id = p.lead_id
    """)
    rows = cursor.fetchall()

    content = "Employee-Project Join:\n"
    for row in rows:
        content += f"{row}\n"

    doc = Document(
        page_content=content,
        metadata={"source": db_path, "type": "employee_project_join"}
    )

    connection.close()
    return [doc]
```

**Key concept:** the join type (inner, outer, left) and the specific columns selected are a deliberate content-design choice, not a fixed default — different problem statements call for different joins, and the resulting `page_content` should reflect whatever relationship actually matters for retrieval.

## Table-Dump vs Join Documents

| | Per-table dump | Relationship / join |
|---|---|---|
| Captures | Each table's own rows | Cross-table relationships (e.g. who leads which project) |
| Query | `SELECT * FROM table` | `JOIN ... ON ...` |
| Best for | General lookup of a single entity's records | Questions that span two related entities |
| Limitation | Loses foreign-key relationships | Requires knowing which joins are meaningful upfront |

## Big Picture

SQL parsing reinforces the same principle carried through every source type in this course: automatic/generic extraction (dumping raw rows) is fast but loses structure, while a custom function that models the *actual relationships* in the data (joins, foreign keys) produces `page_content` far better suited to real Q&A. As with CSV/JSON, the resulting documents still need to go through a text splitter afterward if any individual table or join result is large.
