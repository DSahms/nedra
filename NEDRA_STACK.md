# Nedra Desktop — Technology Stack

**Audit source:** Consumer desktop engine on D drive (June 2026)  
**Purpose:** What runs Nedra, why each piece exists, and what is excluded from the shipped bundle.

---

## Product shape

Nedra Desktop is a **Windows-first personal AI workstation**: one Tkinter window, local persistence, swappable cloud/local LLM backends, no Docker required for the consumer edition.

Corporate line: **Powered by Rémanence**.

---

## Core runtime

| Layer | Technology | Why |
|-------|------------|-----|
| Language | Python 3.10–3.11 | Existing engine, PyInstaller support, rich LLM SDK ecosystem |
| GUI | Tkinter + ttk | Native Windows UI, no Electron weight, ships in stdlib |
| GUI styling | Custom theme module (forest palette, gilt accents) | Distinct product identity; no third-party UI framework |
| Entry / CLI | Typer + `main_gui_assistant.py` | Subcommands for chat, smoke, acceptance, diagnose; frozen exe runs with no args |
| Config | YAML + `dpath` dot-notation loader | Human-readable; runtime paths injected at bootstrap |
| Env vars | Canonical `NEDRA_*` names (with legacy alias fallbacks during migration) | Edition, config path, data dirs without hardcoding |
| Secrets | `python-dotenv` → `%LOCALAPPDATA%\Nedra\.env` | API keys never baked into exe; user-paste onboarding |

---

## Inference (brains)

| Provider | SDK / protocol | Role |
|----------|----------------|------|
| Venice AI | OpenAI-compatible HTTP (`openai` package) | Default consumer cloud; Llama 3.3 70B; privacy-oriented |
| Groq | OpenAI-compatible HTTP | Free-tier backup cloud |
| KoboldCpp | `requests` → `/v1/chat/completions` | Local inference on port 5001 (power-user / dev edition) |
| Ollama, LM Studio | Optional local backends | Dev/power builds |
| Claude, OpenAI, Gemini | Optional cloud backends | Dev/business builds; lazy-imported if deps present |

**Design choice:** OpenAI-compatible chat API as the lingua franca so backends swap without changing prompt assembly.

---

## Memory and data

| Component | Technology | Consumer edition |
|-----------|------------|------------------|
| Graph metadata | SQLite (`sqlite_graph_store.py`) | Primary store; no Neo4j on end-user desk |
| Vector recall | ChromaDB persistent client | Semantic search over memory text |
| Embeddings (consumer) | `HashEmbedder` — SHA256 word hashing to 384-dim vectors | Zero GPU, zero Ollama, works offline |
| Embeddings (dev) | Ollama `nomic-embed-text` | Higher-quality recall when Ollama is up |
| Tasks | SQLite `tasks.db` | Optional task manager integration |
| Chat sessions | JSON file `sessions.json` | Sidebar session list, simple and portable |

---

## Personality

| Piece | Format |
|-------|--------|
| Profile traits | JSON in `profiles/` (warmth, directness, humor, etc.) |
| Edition contract | `assets/persona.json` — edition overrides (consumer, business, wellness) |
| Engine | `PersonalityEngine` builds system prompt from profile + rules + memory context |

---

## Voice (optional, consumer defaults off)

| Piece | Technology | Shipped in consumer exe? |
|-------|------------|---------------------------|
| TTS | `pyttsx3` (local) | Optional; `voice: none` in consumer config |
| STT (mic button) | `speech_recognition` + Google Web Speech API | Optional dependency; mic needs network |
| Heavy voice stack | `realtimestt`, `realtimetts`, `elevenlabs` | **Excluded** from PyInstaller bundle (size + GPU) |

---

## Research tab (optional, dev edition)

| Piece | Technology |
|-------|------------|
| Open Notebook integration | Separate MIT stack (FastAPI + Next.js); health check on :5055 |
| In-app embed | `wryview` WebView2 on Windows |
| Launch | Native batch launcher or browser fallback |

Not bundled in consumer portable zip; status line only unless stack is installed.

---

## Packaging and build

| Tool | Role |
|------|------|
| PyInstaller (`packaging/nedra.spec` equivalent) | Single-folder `Nedra.exe` + `_internal` |
| `collect_all(chromadb)` + `wryview` | Bundle vector DB and optional WebView |
| Runtime hook | Set `NEDRA_EDITION=consumer`, disable Chroma telemetry |
| PowerShell | `build_*_consumer.ps1`, `package_for_*.ps1`, ship queue scripts |
| NSIS | Optional installer (zip + `Install-Nedra.ps1` is primary consumer path) |

**Excluded from consumer bundle:** torch, realtimestt, elevenlabs (explicit excludes in spec).

---

## Dependencies (engine `pyproject.toml`)

**Always relevant:**

- `pyyaml`, `dpath`, `python-dotenv`, `pydantic`, `typer`, `rich`
- `openai`, `anthropic`, `google-generativeai`, `ollama`, `groq` (via openai client)
- `chromadb`, `neo4j` (neo4j only for business/dev)
- `pyttsx3`, `python-telegram-bot` (telegram = business edition only)

**GUI extras (`requirements-gui.txt`):**

- `Pillow` — splash/icon
- `wryview` — WebView2 embed for Notebooks tab

**Test:** `pytest`

---

## What Nedra is not (stack-wise)

- Not a web app or Electron shell
- Not dependent on Docker for consumer install
- Not a single-vendor chat wrapper (soul/brain split is first-class)
- Not bundled with a local LLM weights file (Kobold/Ollama are external processes)

---

## Edition split

| Edition | Graph memory | Default brain | Target user |
|---------|--------------|---------------|-------------|
| **Consumer** | SQLite + hash embeddings | Venice cloud | Non-technical; Brittany-style install |
| **Business / dev** | Neo4j + Ollama embeddings | Kobold local | Dave desk; Docker; morning briefing, Telegram |

Nedra Desktop v1 ship target = **consumer edition** as documented here.
