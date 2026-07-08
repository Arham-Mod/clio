
# CLIO

![CLIO cover](img/Clio.jpg)

A tool that reads a code repository, understands how it's put together, and answers questions about it with actual references back to the source code.

## What this is

CLIO is a RAG (retrieval-augmented generation) system built specifically for code repositories. RAG in general means retrieving relevant pieces of information from some source and feeding them to an LLM so it answers from that context instead of guessing. Most RAG tools do this with plain text, splitting it into fixed-size chunks. Code doesn't work well when treated that way, since a chunk boundary can easily land in the middle of a function.

CLIO instead parses the repository at the function/class level using Python's AST, keeps track of which functions call which others, and expands retrieval along that call graph before assembling context for the LLM. Every answer returned includes the file, function, and line range it came from.

The problems with generic text-chunking RAG applied to code:

| Issue | Impact |
|-------|-------|
| Code treated as plain text | Loss of structural semantics |
| Random/fixed-size chunking | Broken function boundaries |
| No dependency awareness | Incomplete context |
| No source tracing | No traceability, hallucination risk |

CLIO's pipeline, at a glance:

1. Parse the repo into functions/classes via Python AST
2. Store each chunk with its call relationships intact
3. Retrieve top-k chunks by embedding similarity
4. Expand retrieval with callees/callers from the call graph
5. Assemble grounded context and generate an answer with sources

## How it works, step by step

```
Repository
    ↓
Read and split code into functions/classes (Python AST)
    ↓
Convert each piece into a vector (MiniLM, 384 dimensions)
    ↓
Store vectors in a FAISS index for fast searching
    ↓
Find the closest matches to a question (cosine similarity)
    ↓
Pull in related functions and callers (dependency expansion)
    ↓
Assemble all of it into one context, sized to fit
    ↓
Build a prompt that keeps the model grounded in that context
    ↓
Send it to Groq's Llama3-8B model
    ↓
Return an answer with sources attached
```

## Core components

### AST Parsing
Extracts functions, classes, and methods with precise line ranges. Preserves logical code boundaries. Captures function call dependencies. Handles syntax errors gracefully (skips the file instead of crashing).

### Semantic Vector Retrieval
- Embedder: Sentence Transformers (`all-MiniLM-L6-v2`) → 384-dim vectors
- Storage: FAISS `IndexFlatIP` with L2 normalization
- Similarity: Cosine similarity via normalized inner product
- Performance: Sub-millisecond retrieval for thousands of functions

### Dependency Expansion
Analyzes function call relationships. Expands retrieved chunks with callees and callers so the LLM sees the fuller execution flow, which reduces hallucination compared to isolated snippets.

### Context Assembly
Formats code chunks with metadata (file, function, line range) and packs them token-aware to fit the context window while keeping semantic coherence.

### Grounded LLM Generation
Uses Groq (Llama3-8B) for inference. Enforces an answer-from-context constraint so the model can't fall back on assumptions outside what was retrieved.

### Explainability
Maps every answer back to specific functions, files, and line ranges for debugging and verification.

### Interactive CLI
Query the system directly from the terminal with real-time responses and source traceability.

## Example

**Query:**
```
How does code chunking and indexing work?
```

**Answer:**
```
The code is parsed using Python's AST module, which recursively 
extracts functions, classes, and methods. Each code unit is 
converted to a dense vector (384 dimensions) using Sentence 
Transformers, then indexed in FAISS for fast similarity search.

During retrieval, the query is embedded, searched against FAISS, 
and top-k results are expanded using dependency information to 
provide complete context to the LLM.
```

**Sources:**
```
core/parsing/ast_parser.py → ASTParser.parse_file() [L23-67]
core/indexing/embedder.py → Embedder.embed_batch() [L35-42]
core/indexing/faiss_store.py → FaissVectorStore.search() [L29-38]
core/retrieval/dependency_expander.py → expand_dependencies() [L12-45]
```

## Tech stack

| Component | Technology |
|-----------|-----------|
| Parsing | Python AST |
| Embeddings | Sentence Transformers (MiniLM) |
| Vector Search | FAISS (IndexFlatIP) |
| LLM | Groq (Llama3-8B) |
| CLI | Python argparse |
| Metadata Store | Pickle + Dict |
| Logging | Python logging |

## Project structure

```
clio/
├── core/
│   ├── ingestion/
│   │   ├── repo_loader.py       # Recursively load Python files
│   │   └── github_fetcher.py    # Fetch repos from GitHub
│   │
│   ├── parsing/
│   │   ├── ast_parser.py        # AST-based code extraction
│   │   └── chunk_model.py       # CodeChunk dataclass
│   │
│   ├── indexing/
│   │   ├── embedder.py          # MiniLM vector generation
│   │   ├── faiss_store.py       # FAISS index + search
│   │   └── metadata_store.py    # Vector ID to metadata mapping
│   │
│   ├── retrieval/
│   │   ├── retriever.py         # Semantic search pipeline
│   │   └── dependency_expander.py # Expand call dependencies
│   │
│   ├── context_builder/
│   │   └── build_context.py     # Assemble context from chunks
│   │
│   └── generation/
│       ├── llm_client.py        # Groq LLM interface
│       └── prompt_builder.py    # Construct grounded prompts
│
├── utils/
│   └── logging/
│       └── logger.py            # Logging config
│
├── data/
│   └── raw/                     # Repository storage
│
├── app.py                       # Main pipeline
├── cli_app.py                   # Interactive CLI
├── requirements.txt
└── README.md
```

## Design decisions

### Function-Level Chunking
- Preserves semantic meaning: functions are logical, intent-preserving units
- Improves retrieval: precise boundaries reduce noise
- Enables explainability: easy to map answers to specific functions

### Dependency Expansion
- Reflects code structure: functions rarely work in isolation
- Provides complete context: LLM receives full call sequences
- Reduces hallucination: explicit dependencies prevent fabrication

### FAISS and MiniLM Selection
- Efficient: sub-millisecond search on thousands of chunks
- Local execution: no external API dependency
- Cost-effective: small, fast model suitable for embedded systems

### Grounded Prompting
- Ensures accountability: model answers only from retrieved context
- Maintains verifiability: answers are traceable to source code
- Builds trust: users can audit reasoning

## Requirements

```
Python 3.10+
faiss-cpu (or faiss-gpu)
sentence-transformers
groq
numpy
python-dotenv
```

## Installation and usage

1. Clone the repository and install dependencies:
```bash
git clone <repo_url>
cd clio
pip install -r requirements.txt
```

2. Set up your environment variables in a `.env` file:
```
GROQ_API_KEY=your_groq_api_key_here
GITHUB_TOKEN=your_github_token_here  # (optional, for GitHub fetching)
```

3. Run it:

Batch analysis of a local repository:
```bash
python app.py
```

Interactive query mode:
```bash
python cli_app.py
```

Fetch a repository from GitHub:
```bash
python repoload_test.py --repo_url https://github.com/user/repo
```

## Performance

| Metric | Value |
|--------|-------|
| Embedding Speed | ~100 functions/sec (MiniLM) |
| Search Speed | <1ms per query (FAISS) |
| Memory (Indexed) | ~40KB per function (384-dim float32) |
| Context Window | ~2K tokens (adjustable) |
| Inference Time | ~500ms to 2s (Groq Llama3-8B) |

## Key Insights

1. Structure over scale: function-level parsing outperforms naive chunking regardless of LLM capacity
2. Retrieval quality is critical: most of the answer quality derives from retrieved context
3. Dependencies matter: call graph expansion significantly improves context completeness
4. Explainability builds trust: source attribution is essential for enterprise deployments
5. Modularity simplifies complexity: decomposing into ingestion, parsing, indexing, retrieval, and generation stages makes the system maintainable

## Future Improvements

- Hybrid retrieval: combine semantic and keyword search (BM25)
- Multi-language support: Tree-sitter integration for Go, Rust, JavaScript
- Smart ranking: learning-to-rank for context prioritization
- Web interface: visualization of retrieved code and dependencies
- Runtime call tracing: dynamic dependency analysis
- Incremental indexing: support for streaming code updates
- Caching layer: cache frequent queries and dependencies

## Limitations

- Python-only: currently supports Python repositories
- Static analysis: no runtime tracing (AST-based)
- Embedding model constraints: retrieval quality limited by MiniLM
- LLM dependency: requires Groq API access
- CLI-only interface: no IDE integration

## Capabilities

- Understands code structure and relationships
- Retrieves relevant context with high precision
- Expands context intelligently using dependencies
- Generates grounded, verifiable answers
- Provides complete source attribution
- Scales to thousands of functions
- Operates locally without external infrastructure

## More examples

### Module analysis
```bash
$ python cli_app.py
Query: What functions are exported from the parsing module?

Answer: The parsing module exports the ASTParser class, which 
has two main methods: parse_file() for individual files, and 
parse_repository() for batch processing.

Sources:
core/parsing/ast_parser.py → ASTParser [L1-120]
```

### Pipeline analysis
```bash
Query: How does the retrieval pipeline work?

Answer: The retrieval pipeline:
1. Embeds the query using Embedder.embed_text()
2. Searches FAISS with FaissVectorStore.search()
3. Expands dependencies using DependencyExpander
4. Assembles context with ContextBuilder
5. Builds grounded prompt with PromptBuilder
6. Queries Groq LLM with assembled context

Sources:
core/retrieveal/retriever.py → retrieve() [L10-45]
core/retrieval/dependency_expander.py [L15-80]
core/context_builder/build_context.py [L5-30]
```



## License
Licensed under the Apache License, Version 2.0. See the LICENSE file for details.