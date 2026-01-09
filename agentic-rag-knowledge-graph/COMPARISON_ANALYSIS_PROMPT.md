# Agentic RAG Knowledge Graph - Comprehensive System Analysis Prompt

Use this prompt to analyze your "dpiral memory mcp" project against the agentic-rag-knowledge-graph reference implementation. This will help identify architectural gaps, missing features, and improvement opportunities.

---

## SYSTEM OVERVIEW

The reference system is a **hybrid RAG + Knowledge Graph** implementation that combines:
- **Vector Database** (PostgreSQL + pgvector) for semantic similarity search
- **Knowledge Graph** (Neo4j + Graphiti) for entity relationships and temporal facts
- **Pydantic AI** as the agent framework
- **FastAPI** for the streaming API layer

### Key Architectural Decisions
1. Dual-storage approach separating concerns between semantic search and relationship traversal
2. Temporal knowledge tracking (facts have valid_at/invalid_at timestamps)
3. LLM-assisted semantic chunking during ingestion
4. Rule-based entity extraction with predefined entity catalogs
5. Configurable LLM/embedding providers (OpenAI, Ollama, OpenRouter, Gemini)

---

## GAP ANALYSIS CHECKLIST

### 1. DATA INGESTION PIPELINE

**Questions to evaluate your system:**

- [ ] **Document Processing**: Does your system support markdown/text document ingestion with metadata extraction (YAML frontmatter)?
- [ ] **Chunking Strategy**:
  - Do you have semantic chunking (LLM-assisted boundary detection)?
  - Do you support configurable chunk sizes (default: 1000 chars, overlap: 200)?
  - Can you split on structural boundaries (headers, paragraphs, lists, code blocks)?
  - Is there fallback to simple character-based chunking?
- [ ] **Entity Extraction**:
  - Do you extract entities during ingestion (companies, technologies, people, locations)?
  - Are entities stored in chunk metadata for later retrieval?
  - Is extraction rule-based or LLM-based?
- [ ] **Embedding Generation**:
  - Do you support multiple embedding providers?
  - Is embedding dimension configurable (1536 for OpenAI, 768 for Ollama nomic)?
- [ ] **Dual Storage**:
  - Do embeddings go to a vector database?
  - Do entities/relationships go to a knowledge graph?
  - Can you skip graph building for faster ingestion (--fast mode)?

**Reference Implementation Details:**
```
Ingestion Flow:
Document → Parse Markdown → Extract Metadata →
Semantic Chunk → Extract Entities → Generate Embeddings →
Store in PostgreSQL (chunks) + Neo4j (episodes/relationships)
```

---

### 2. VECTOR DATABASE LAYER

**Questions to evaluate your system:**

- [ ] **Schema Design**: Do you have separate tables for:
  - `documents` (full content + metadata)
  - `chunks` (content + embedding + metadata + document reference)
  - `sessions` (conversation tracking)
  - `messages` (conversation history)
- [ ] **Vector Search Function**: Do you have a stored procedure for vector similarity search?
- [ ] **Hybrid Search Function**: Can you combine vector similarity + keyword/text search with configurable weights?
- [ ] **Indexing**:
  - Do you use IVFFlat or HNSW for vector indexes?
  - Do you have trigram indexes for text search (pg_trgm)?
- [ ] **Connection Pooling**: Do you use async connection pooling (asyncpg)?

**Reference Schema:**
```sql
-- Key tables
documents (id, title, source, content, metadata, timestamps)
chunks (id, document_id, content, embedding vector(1536), chunk_index, metadata, token_count)
sessions (id, user_id, metadata, timestamps, expires_at)
messages (id, session_id, role, content, metadata, timestamp)

-- Key functions
match_chunks(query_embedding, match_count) - pure vector search
hybrid_search(query_embedding, query_text, match_count, text_weight) - vector + keyword
```

---

### 3. KNOWLEDGE GRAPH LAYER

**Questions to evaluate your system:**

- [ ] **Graph Engine**: Do you use Neo4j or another graph database?
- [ ] **Knowledge Framework**: Do you use Graphiti or similar for temporal knowledge management?
- [ ] **Episode-Based Ingestion**: Can you add "episodes" (content chunks) that automatically extract entities and relationships?
- [ ] **Temporal Facts**:
  - Do facts have `valid_at` timestamps?
  - Can facts be invalidated (`invalid_at`)?
  - Can you query fact timelines?
- [ ] **Semantic Search on Graph**: Can you perform semantic search within the knowledge graph itself?
- [ ] **Entity Relationship Queries**: Can you traverse relationships from a central entity?
- [ ] **Graph Statistics**: Can you retrieve graph statistics (node counts, relationship counts)?

**Reference Graphiti Integration:**
```python
# Adding content to graph
await graphiti.add_episode(
    name=episode_id,
    episode_body=content,
    source=EpisodeType.text,
    source_description=source,
    reference_time=timestamp
)

# Searching the graph
results = await graphiti.search(query)  # Returns facts with temporal data
```

---

### 4. AGENT FRAMEWORK & TOOLS

**Questions to evaluate your system:**

- [ ] **Agent Framework**: Do you use Pydantic AI, LangChain, or another framework?
- [ ] **Tool Registration**: How are tools registered and invoked?
- [ ] **Available Tools**: Does your agent have these capabilities?

| Tool | Purpose | Your System? |
|------|---------|--------------|
| `vector_search` | Semantic similarity search across chunks | [ ] |
| `graph_search` | Query knowledge graph for facts/relationships | [ ] |
| `hybrid_search` | Combined vector + keyword search | [ ] |
| `get_document` | Retrieve full document content | [ ] |
| `list_documents` | Browse available documents | [ ] |
| `get_entity_relationships` | Graph traversal from entity | [ ] |
| `get_entity_timeline` | Temporal fact retrieval for entity | [ ] |

- [ ] **Context/Dependencies**: Does your agent have session context and user preferences?
- [ ] **System Prompt**: Is there guidance on when to use which tool?

**Reference Tool Selection Logic:**
```
- Vector search: For finding similar content, detailed explanations
- Graph search: For understanding relationships between entities
- Hybrid search: When combining semantic + exact matching
- Use graph only when user asks about two+ entities in same question
```

---

### 5. API LAYER

**Questions to evaluate your system:**

- [ ] **Framework**: Do you use FastAPI, Flask, or another framework?
- [ ] **Streaming Responses**:
  - Do you support Server-Sent Events (SSE)?
  - Can you stream agent responses in real-time?
- [ ] **Endpoints**:
  - `/health` - Health check
  - `/chat` - Non-streaming chat
  - `/chat/stream` - Streaming chat
  - `/docs` - OpenAPI documentation
- [ ] **Session Management**:
  - Can you create/retrieve/update sessions?
  - Do sessions have expiration?
- [ ] **Tool Usage Tracking**: Are tool calls reported in responses?

**Reference API Response Structure:**
```python
class ChatResponse:
    message: str
    session_id: str
    tools_used: List[ToolUsage]  # tool_name, parameters, result_count

class StreamEvent:
    event: str  # "message", "tool_use", "error", "done"
    data: dict
```

---

### 6. SEARCH STRATEGIES

**Questions to evaluate your system:**

- [ ] **Pure Vector Search**:
  - Returns chunks ordered by cosine similarity
  - No minimum threshold (returns top N regardless of score)
- [ ] **Hybrid Search Algorithm**:
  - Combines vector similarity with full-text search (ts_rank_cd)
  - Configurable text_weight parameter (default 0.3)
  - Uses FULL OUTER JOIN to include both vector-only and text-only matches
- [ ] **Graph Search**:
  - Semantic search within knowledge graph
  - Returns facts with temporal validity information
- [ ] **Entity-Focused Queries**:
  - Can search for relationships involving specific entity
  - Can retrieve chronological timeline of facts

**Reference Hybrid Search Formula:**
```sql
combined_score = (vector_similarity * (1 - text_weight)) + (text_similarity * text_weight)
```

---

### 7. CONFIGURATION & FLEXIBILITY

**Questions to evaluate your system:**

- [ ] **LLM Provider Flexibility**:
  - OpenAI
  - Ollama (local)
  - OpenRouter
  - Gemini
  - Custom base URLs
- [ ] **Embedding Provider Flexibility**: Same as above
- [ ] **Configurable Parameters**:
  - Chunk size and overlap
  - Search limits
  - Session timeouts
  - Embedding dimensions
- [ ] **Environment Variables**: Are all secrets and configs externalized?
- [ ] **Ingestion CLI Options**:
  - `--clean` - Wipe and rebuild
  - `--chunk-size` - Custom chunk size
  - `--no-semantic` - Disable LLM chunking
  - `--no-entities` - Disable entity extraction
  - `--fast` - Skip knowledge graph building
  - `--verbose` - Debug logging

---

### 8. DATA MODELS & TYPES

**Questions to evaluate your system:**

- [ ] **Chunk Results**: Do results include?
  - chunk_id, document_id, content
  - similarity/relevance score
  - document_title, document_source
  - metadata
- [ ] **Graph Results**: Do results include?
  - fact (the extracted knowledge)
  - uuid (unique identifier)
  - valid_at, invalid_at (temporal validity)
  - source_node_uuid
- [ ] **Document Metadata**: Do you track?
  - title, source, content
  - file_path, file_size
  - word_count, line_count
  - ingestion_date
  - YAML frontmatter fields
- [ ] **Chunk Metadata**: Do you track?
  - chunk_method (semantic/simple)
  - total_chunks
  - entities extracted
  - token_count

---

### 9. ERROR HANDLING & RESILIENCE

**Questions to evaluate your system:**

- [ ] **Graceful Degradation**:
  - Fallback from semantic to simple chunking on failure
  - Continue processing other chunks if one fails
  - Return empty results instead of crashing on search errors
- [ ] **Logging**: Comprehensive logging with timestamps and levels
- [ ] **Connection Management**:
  - Connection pooling with min/max sizes
  - Inactive connection cleanup
  - Command timeouts
- [ ] **Content Size Limits**:
  - Truncation for oversized chunks (Graphiti 8192 token limit)
  - Sentence-boundary-aware truncation

---

### 10. TESTING & QUALITY

**Questions to evaluate your system:**

- [ ] **Unit Tests**: Coverage for:
  - Database utilities
  - Chunker
  - Models
- [ ] **Test Configuration**: pytest.ini with proper settings
- [ ] **Connection Testing**:
  - `test_connection()` for PostgreSQL
  - `test_graph_connection()` for Neo4j

---

## FEATURE COMPARISON MATRIX

| Feature | Reference System | Your System | Gap? |
|---------|-----------------|-------------|------|
| Vector DB (pgvector) | ✅ | | |
| Knowledge Graph (Neo4j) | ✅ | | |
| Temporal Facts (Graphiti) | ✅ | | |
| Semantic Chunking | ✅ | | |
| Entity Extraction | ✅ | | |
| Hybrid Search | ✅ | | |
| Graph Traversal | ✅ | | |
| Entity Timeline | ✅ | | |
| Streaming API | ✅ | | |
| Session Management | ✅ | | |
| Multi-provider LLM | ✅ | | |
| CLI Interface | ✅ | | |

---

## IMPLEMENTATION PRIORITIES

Based on your gap analysis, prioritize features by:

### High Impact (Core RAG capabilities)
1. Dual-storage (vector + graph) if missing
2. Hybrid search combining vector + keyword
3. Entity extraction during ingestion
4. Temporal fact tracking

### Medium Impact (User Experience)
5. Streaming API responses
6. Tool usage visibility
7. Session persistence
8. CLI interface

### Lower Impact (Operational)
9. Multi-provider support
10. Semantic chunking
11. Comprehensive testing
12. Ingestion CLI options

---

## QUESTIONS TO ASK YOUR SYSTEM

1. **Can your system answer "How is Company A related to Company B?"** (requires graph traversal)
2. **Can your system show a timeline of facts about an entity?** (requires temporal tracking)
3. **Can your system combine semantic similarity with keyword matching?** (requires hybrid search)
4. **Can your system extract and track entities from documents?** (requires entity extraction)
5. **Can your system stream responses in real-time?** (requires SSE support)
6. **Can your system explain which tools it used to answer a question?** (requires tool tracking)
7. **Can your system work with multiple LLM providers?** (requires provider abstraction)
8. **Can your system recover from partial ingestion failures?** (requires error handling)

---

## NEXT STEPS

After completing this analysis:

1. Document which features are present, missing, or partial in your system
2. Identify which gaps are most critical for your use case
3. Review the reference implementation files for detailed implementation patterns
4. Create a roadmap for implementing missing features

**Key files to reference:**
- `agent/agent.py` - Agent setup and tool registration
- `agent/tools.py` - Tool implementations
- `agent/graph_utils.py` - Graphiti/Neo4j integration
- `agent/db_utils.py` - PostgreSQL operations
- `ingestion/ingest.py` - Full ingestion pipeline
- `ingestion/chunker.py` - Semantic chunking
- `ingestion/graph_builder.py` - Knowledge graph building
- `sql/schema.sql` - Database schema and functions
