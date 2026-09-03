# Text Splitting Strategies (LangChain)

## Why Chunking Exists

LLMs and embedding models have context/token limits, so large documents must be split into smaller chunks before embedding and storage. This is the Document Splitter / Text Splitter stage of the RAG pipeline — it comes right after document loading.

## 1. `split_text()` vs `split_documents()`

- `split_text(text)` → input is a raw string, output is a list of strings
- `split_documents(documents)` → input is a list of `Document` objects, output is a list of `Document` objects (preserves metadata)

Use whichever matches your input type — if you already have raw text (e.g. `document.page_content`), `split_text` is simpler.

## 2. CharacterTextSplitter

Splits based on a single separator, with a hard character-count limit.

```python
from langchain.text_splitter import CharacterTextSplitter

char_splitter = CharacterTextSplitter(
    separator="\n",       # split point
    chunk_size=200,       # max characters per chunk
    chunk_overlap=20,     # characters repeated between chunks
    length_function=len   # how chunk size is measured
)

char_chunks = char_splitter.split_text(text)
print(len(char_chunks))     # number of chunks created
print(char_chunks[0])       # first chunk
```

**Key insight:** overlap is only visible when the splitter is forced to cut mid-content. If the separator (e.g. `\n`) already produces clean natural breaks, chunks won't show overlap even though `chunk_overlap` is set — the splitter respects the separator boundary first, and overlap only kicks in when a chunk needs to be forcibly cut mid-way.

**Pros:** simple, predictable. **Cons:** can break mid-sentence. **Use when:** text has clear, consistent delimiters.

## 3. RecursiveCharacterTextSplitter (default / most recommended)

Tries a list of separators in priority order — falls back to the next one only if the chunk is still too large.

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

recursive_splitter = RecursiveCharacterTextSplitter(
    separators=["\n\n", "\n", " ", ""],  # tried in this order
    chunk_size=200,
    chunk_overlap=20,
    length_function=len
)

recursive_chunks = recursive_splitter.split_text(text)
```

**Key insight:** overlap visibility depends on separator granularity.
- Clean separators (clear line breaks) → little/no visible overlap, since natural boundaries already isolate chunks well
- Coarse separators (e.g. only spaces, no punctuation/newlines) → the splitter is forced to cut mid-content, producing clearly visible word-level overlap between consecutive chunks

**Why overlap matters:** if relevant information spans a chunk boundary, overlap ensures the same context appears in both adjacent chunks — increasing the odds a similarity search retrieves the complete context rather than a fragment cut in half.

**Pros:** respects text structure, general-purpose default. **Cons:** slightly slower. **Use when:** no strong reason to pick otherwise — it's the safe default.

## 4. TokenTextSplitter

Splits based on token count, not raw character count — tokens roughly correspond to words/sub-words.

```python
from langchain.text_splitter import TokenTextSplitter

token_splitter = TokenTextSplitter(
    chunk_size=50,
    chunk_overlap=10
)

token_chunks = token_splitter.split_text(text)
```

**Why this exists:** LLMs and embedding models operate on tokens, not characters — so token-based splitting aligns chunk boundaries with what the model actually "sees" as one unit, making it more accurate for token-limited models than character-based splitting. Character count ≠ token count, especially across languages/punctuation.

## 5. Comparison

| Splitter | Cuts on | Pros | Cons | Best for |
|---|---|---|---|---|
| CharacterTextSplitter | 1 fixed separator | Simple, predictable | May break mid-sentence | Clearly delimited text |
| RecursiveCharacterTextSplitter | Multiple separators (fallback order) | Structure-aware, general-purpose | Slightly slower | Default choice |
| TokenTextSplitter | Token count | Matches model's real input unit | Slowest (counts tokens) | Token-limited models |

## 6. Why This Matters for Retrieval (the payoff)

Once chunks are embedded and stored in the vector database:

1. A user query is embedded and compared via similarity search against all stored chunk vectors
2. Only the top-matching chunks (e.g. 3–4 out of 1000) are retrieved
3. Overlap increases the odds that split context still gets retrieved together — if a key fact spans a chunk boundary, both neighboring chunks partially contain it, so a query is more likely to catch the full context instead of missing half of it
4. Retrieved chunks are passed to the LLM as augmented context to generate the final answer

**Core takeaway:** chunk size, separator choice, and overlap directly control retrieval quality — this is a tunable design decision, not a fixed default. Different document types/use cases warrant different splitter configurations.
