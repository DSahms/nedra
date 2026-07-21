# Nedra Desktop — Local Memory

**Audit source:** `memory_vault/`, `consumer_bootstrap`, `base_assistant` (June 2026).

---

## Design goals

1. **Local-first** — consumer data never requires cloud DB or Docker
2. **Survive restarts** — memories persist under `%LOCALAPPDATA%\Nedra\`
3. **Semantic recall** — past exchanges surface in new prompts when relevant
4. **Graceful degradation** — if Chroma fails, SQLite graph still stores; recall quality may drop
5. **Non-blocking GUI** — memory writes happen on background threads with a lock

---

## Storage layout (consumer)

| Artifact | Path | Role |
|----------|------|------|
| Graph DB | `%LOCALAPPDATA%\Nedra\consumer_memory.db` | Memory nodes, entities, conversations, relationships |
| Vector store | `%LOCALAPPDATA%\Nedra\chroma\` | Embeddings for semantic similarity |
| Tasks | `%LOCALAPPDATA%\Nedra\tasks.db` | TaskManager (optional feature) |
| Sessions | `%LOCALAPPDATA%\Nedra\sessions.json` | Chat UI sessions (separate from vault) |

Bootstrap rewrites relative paths in template YAML to these absolute paths in `assistant_config.runtime.yml`.

---

## Architecture

```
PlainAssistant.process_text()
        │
        ├─► _recall_memories(query) ──► MemoryManager.recall()
        │                                      │
        │                              ┌───────┴───────┐
        │                              ▼               ▼
        │                         ChromaStore    SQLiteGraphStore
        │                         (vectors)      (metadata graph)
        │
        └─► after response ──► _store_exchange_memory() [background thread]
                                      │
                                      └─► MemoryManager.remember()
```

---

## Graph store factory

`memory_factory.create_graph_store()` reads `memory.backend`:

| Value | Implementation |
|-------|----------------|
| `sqlite` | `SQLiteGraphStore` — consumer default |
| `neo4j` | `Neo4jClient` — dev/business (Docker bolt://localhost:7687) |

On connect: `initialize_schema()` creates tables for memories, entities, conversations.

---

## MemoryManager operations

### `remember(content, memory_type, importance, entity_names?)`

1. Create memory node in graph store (UUID id)
2. If Chroma available: embed text → `chroma.store_memory(neo4j_id, text, embedding, metadata)`
3. Optional entity linking in graph

Consumer exchanges stored as:

```
User: {user_text}
Assistant: {response}
```

`memory_type="event"`, `importance=0.6` for normal chat.

### `recall(query, limit)`

1. Embed query (hash or Ollama)
2. Chroma similarity search → ranked memory texts
3. Return list of `{content, ...}` dicts

Recalled text formatted as:

```
[Relevant past conversations]

User: ...
Assistant: ...

...
```

Injected into system prompt via personality engine.

---

## Embedding strategy

### Consumer: `HashEmbedder` (`utils/embedding_utils.py`)

- Splits text into words, SHA256 per word, accumulates into 384-dim vector, L2-normalizes
- **Pros:** No GPU, no Ollama, deterministic, instant, offline
- **Cons:** Not true semantic similarity — keyword-ish recall; acceptable for v1 consumer ship

Selected because Brittany-style install cannot assume Ollama or CUDA.

### Dev / business: `OllamaEmbedder`

- POST `http://localhost:11434/api/embeddings` with `nomic-embed-text`
- Used when `memory.embedding.backend: ollama` and edition ≠ consumer

Bootstrap forces `embedding.backend: hash` in runtime config for consumer.

---

## ChromaDB

- `PersistentClient(path=persist_directory)`
- Collection name: `memories`
- Document id = graph memory UUID (links graph ↔ vector)
- Upsert on remember; query on recall

Bundled inside PyInstaller consumer exe (large dependency — accepted for semantic path).

Telemetry disabled via runtime hook: `ANONYMIZED_TELEMETRY=False`.

---

## Thread safety

`PlainAssistant._memory_lock` wraps:

- `_recall_memories()` — worker thread during inference
- `_store_exchange_memory()` — background thread after reply

Prevents concurrent Chroma/SQLite access from corrupting state or crashing the process.

---

## Memory commands (in-chat)

`process_text()` detects `"remember this"` — stores last user+assistant exchange with high importance (0.9) synchronously.

Direct memory queries (`_check_memory_query`) can return recalled content without LLM call for simple lookup patterns.

---

## History vs vault

| Store | Scope | Trim policy |
|-------|-------|-------------|
| `conversation_history` | In-memory LLM context window | Trimmed to last 20 exchanges in `PlainAssistant` |
| Memory vault | Long-term persisted facts/events | Not auto-trimmed; recall pulls top-k by similarity |

Older chat turns leave the context window but remain in vault for recall.

---

## Uninstall policy

`Uninstall-Nedra.ps1` removes app binaries and desktop shortcut only.

**Memories remain** in `%LOCALAPPDATA%\Nedra\` unless user deletes that folder manually.

Document clearly in consumer README.

---

## Known limitations

1. Hash embeddings are weak for paraphrase recall — upgrade path is Ollama or hosted embed API
2. Chroma adds ~hundreds of MB to portable zip
3. Neo4j path unused in consumer — dead config keys remain for shared YAML schema
4. No memory export UI in v1 consumer

---

## Rebuild defaults (Nedra consumer YAML)

```yaml
memory:
  enabled: true
  backend: sqlite
  embedding:
    backend: hash
  sqlite_path: <rewritten at bootstrap>
  chroma:
    persist_directory: <rewritten at bootstrap>
```
