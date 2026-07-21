# Nedra Desktop — Architecture

**Audit source:** Engine modules under consumer desktop project (June 2026).

---

## High-level diagram

```
┌─────────────────────────────────────────────────────────────────┐
│  Nedra.exe (PyInstaller)  OR  python main_gui_assistant.py chat  │
└───────────────────────────────┬─────────────────────────────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        ▼                       ▼                       ▼
┌───────────────┐     ┌─────────────────┐     ┌──────────────────┐
│ consumer_     │     │ ChatApp (Tk)    │     │ PlainAssistant   │
│ bootstrap     │     │ sessions, UI,   │     │ brain, memory,   │
│ runtime yml,  │     │ queue, stream   │     │ personality, TTS │
│ .env, paths   │     └────────┬────────┘     └────────┬─────────┘
└───────────────┘              │                       │
                               │                       │
                    ┌──────────┴──────────┐   ┌────────┴────────┐
                    ▼                     ▼   ▼                 ▼
            SessionStore          Notebooks   BrainFactory   MemoryManager
            sessions.json         panel       backends       SQLite + Chroma
```

---

## Entry points

| Invocation | Behavior |
|------------|----------|
| Double-click `Nedra.exe` | GUI chat (consumer) |
| `Nedra.exe smoke-test` | Headless bootstrap + memory smoke |
| `Nedra.exe acceptance-test` | Full pre-ship suite (~15 checks) |
| `Nedra.exe diagnose-test` | API keys, brain switch, Venice probe, exe paths |
| `main_base_assistant.py` | Terminal chat (dev edition) |
| `launcher.py` | Docker Neo4j + Ollama orchestration (dev) |

---

## Bootstrap layer (`consumer_bootstrap`)

On startup (and after Settings save):

1. Ensure `%LOCALAPPDATA%\Nedra\` exists
2. Set `NEDRA_EDITION`, `NEDRA_DATA_DIR`, `NEDRA_OUTPUT_DIR`, `NEDRA_CONFIG`
3. Load `.env` for API keys
4. Copy template `assistant_config.consumer.yml` → `assistant_config.runtime.yml` with **absolute paths** for SQLite, Chroma, tasks DB
5. Merge `user.json` profile (human name, assistant name) into runtime config
6. Thread-safe lock prevents concurrent config rewrites

`reload_consumer_secrets()` reloads `.env` only — used on worker thread before brain switch without rewriting YAML.

---

## GUI architecture (`ChatApp`)

### Layout

- **Left sidebar:** branding, + New chat, Notebooks, Settings, session list
- **Main header:** companion title, status line (`Brain: X · Memory: sqlite`), Chat / Notebooks tabs
- **Chat panel:** read-only transcript (`ScrolledText`), streaming via `stream_mark` Tk mark
- **Bottom bar:** Speak replies checkbox, Brain combobox, Mic, Entry, Send

### Threading model

| Thread | Work |
|--------|------|
| Main (Tk) | UI events, `_poll_queue()` every 80ms, stream rendering, TTS via queue |
| Worker daemon | `switch_brain`, `process_text`, Venice/Kobold HTTP |
| Background | Memory `remember()` after each exchange; optional TTS thread avoided (main-thread TTS queue) |

### Request lifecycle (send message)

1. User sends → `_append("You", text)` → persist to `SessionStore`
2. Capture `speak` flag and brain choice on **main thread** (never read Tk vars from worker)
3. Bump `request_id`; set `_busy=True`
4. Worker: reload secrets → `switch_brain(choice)` → `process_text(text, chunk_callback)`
5. Chunks → queue → `_stream_chunk` on main thread
6. Done/err → unlock busy, append assistant message, optional speak queue item
7. Cancel (Escape) or brain change bumps `request_id` — stale worker results ignored

### Brain dropdown

- Consumer edition shows: `kobold`, `venice`, `groq` (Kobold promoted to top if port 5001 responds)
- Dropdown stays **readonly** during thinking (user can switch brain to bail out)
- Status line reads **active** `brain_backend.name`, not static YAML default

---

## Assistant core (`PlainAssistant`)

Responsibilities:

1. Initialize memory vault (graph + optional Chroma + embedder)
2. Initialize personality engine from profile + edition overrides
3. Walk `brain.fallback_chain` at init; first available backend wins
4. `switch_brain(type)` hot-swap without losing history
5. `process_text()` — echo guard, memory recall, prompt build, stream/generate, history trim, background remember

### Prompt assembly

```
system: PersonalityEngine.get_system_prompt(memory_context)
history: conversation_history (trimmed to last N exchanges)
user: current message
```

Memory context block prepended when `MemoryManager.recall()` returns relevant past exchanges.

---

## Brain factory

```
BrainFactory.BACKENDS = { kobold, lm_studio, ollama, groq, + optional claude, openai, venice, gemini }
BrainFactory.create_backend(type, config) → BaseBrainBackend instance
```

Each backend implements:

- `generate_response()` — blocking completion
- `generate_response_stream()` + `supports_streaming()` — token chunks to GUI
- `is_available()` — probe (HTTP / key presence)
- `get_model_info()` — status line

---

## Notebooks tab (optional)

- `NotebooksPanel` — toolbar: Launch stack, Embed, Open in browser, Refresh
- `notebooks_api_healthy()` — GET `http://127.0.0.1:5055/health`
- `NotebookEmbedHost` — `wryview` WebView2 when available
- Consumer portable: launcher script not bundled; **no modal on tab open** (user clicks Launch explicitly)

---

## Configuration resolution

```
NEDRA_CONFIG env → assistant_config.runtime.yml (consumer)
                 → assistant_config.yml (dev default)
get_config("dotted.path") via dpath
```

---

## Data flow: soul vs brain

| Soul (persistent) | Brain (swappable) |
|-------------------|-------------------|
| Personality profile + edition rules | Venice / Groq / Kobold instance |
| SQLite memories + Chroma vectors | Model name, API URL, timeout |
| Session transcript JSON | Streaming HTTP client |
| user.json names | No personality state |

Switching brain does not clear sessions or memory; only the inference client changes.

---

## Crash and logging

- `crash_log.install_crash_logging()` → `%LOCALAPPDATA%\Nedra\crash-last.txt`
- Session logs under `%LOCALAPPDATA%\Nedra\logs\<session_id>\session.log`
- Consumer ASCII log prefixes on Windows (no emoji in log files)

---

## Key module map (rebuild reference)

| Concern | Module path |
|---------|-------------|
| GUI shell | `modules/gui/chat_app.py` |
| Sessions | `modules/gui/session_store.py` |
| Onboarding | `modules/gui/first_run.py`, `startup_check.py` |
| Theme | `modules/gui/nedra_theme.py` (forest/gilt) |
| Assistant | `modules/base_assistant.py` |
| Brains | `modules/brain_backends/*` |
| Memory | `modules/memory_vault/*` |
| Personality | `modules/personality_engine/*` |
| Bootstrap | `modules/consumer_bootstrap.py` |
| Env | `modules/nedra_env.py` |
| QA | `modules/nedra_acceptance.py`, `nedra_diagnose.py`, `consumer_smoke.py` |
