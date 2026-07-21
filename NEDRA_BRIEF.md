# Nedra Desktop — Complete Technical & Product Brief

**Version:** June 2026 audit  
**Audience:** Developer or agent building Nedra Desktop from scratch  
**Corporate line:** Powered by Rémanence  
**Tagline:** *Keep the soul. Change the brain.*

---

## What you are building

**Nedra Desktop** is a Windows-first personal AI workstation — one forest-green window on the user's desk. It is **not** a facility wellness app, not a browser tab, and not a single-vendor chat wrapper.

| Layer | Purpose |
|-------|---------|
| **Companion chat** | Named AI with personality, streaming replies, session sidebar |
| **Swappable brains** | Same companion; user picks Venice cloud, Groq backup, or local Kobold per message |
| **Local memory** | SQLite graph + Chroma vectors under `%LOCALAPPDATA%\Nedra\` |
| **Research (optional)** | Open Notebook tab — doc chat, sources; dev edition only in practice |
| **Sessions** | Multiple named chats persisted in JSON across restarts |

Default companion name: **Nedra** (user-renamable). Product window title: **Nedra Desktop**.

**v1 ship target:** Consumer edition for non-technical users — no Docker, no Neo4j, no terminal after install.

---

## Document map

| File | Contents |
|------|----------|
| [NEDRA_STACK.md](./NEDRA_STACK.md) | Full technology stack and rationale |
| [NEDRA_ARCHITECTURE.md](./NEDRA_ARCHITECTURE.md) | UI, bootstrap, threading, module map |
| [NEDRA_PERSONALITY.md](./NEDRA_PERSONALITY.md) | Companion layer, profiles, edition rules |
| [NEDRA_MEMORY.md](./NEDRA_MEMORY.md) | SQLite, Chroma, embeddings, thread safety |
| [NEDRA_MODELS.md](./NEDRA_MODELS.md) | Brain providers, switching, API keys |
| [NEDRA_PACKAGING.md](./NEDRA_PACKAGING.md) | PyInstaller, zip, install scripts |
| [NEDRA_QA.md](./NEDRA_QA.md) | Smoke, acceptance, diagnose, crash logging |
| [NEDRA_DECISIONS.md](./NEDRA_DECISIONS.md) | Decision log with tradeoffs |

This file is the **single consolidated summary** — read it first, then drill into the topic files.

---

## Soul vs brain (signature feature)

```
┌─────────────────────────────────────────────────────────┐
│  SOUL (persistent on the PC)                            │
│  · Personality profile + edition rules                  │
│  · Memory vault (SQLite + Chroma)                       │
│  · Session history (JSON)                               │
│  · User name + companion name                           │
├─────────────────────────────────────────────────────────┤
│  BRAIN (swappable via dropdown)                       │
│  · venice  → Venice.ai Llama 3.3 70B (default cloud)  │
│  · groq    → Groq free tier (backup)                    │
│  · kobold  → KoboldCpp localhost:5001 (optional local)  │
└─────────────────────────────────────────────────────────┘
```

Switching brain does **not** clear memory, sessions, or personality. Only the HTTP inference client changes.

---

## Technology stack (summary)

| Area | Choice | Why |
|------|--------|-----|
| Language | Python 3.10–3.11 | PyInstaller, LLM SDK ecosystem |
| GUI | Tkinter + custom forest theme | Native, ~280MB zip vs Electron |
| Config | YAML + dpath + `NEDRA_*` env | Human-readable; runtime absolute paths |
| Secrets | `.env` in LOCALAPPDATA | Keys never baked into exe |
| Consumer memory | SQLite + Chroma + HashEmbedder | No Docker, no GPU, offline embeddings |
| Default inference | Venice OpenAI-compatible API | Privacy cloud + strong model |
| Packaging | PyInstaller one-folder + zip | Fast startup; Chroma DLLs load cleanly |
| Excluded from consumer exe | torch, realtimestt, elevenlabs | Size; voice defaults off |

Full detail: [NEDRA_STACK.md](./NEDRA_STACK.md).

---

## Architecture (summary)

### Entry

```
main_gui_assistant.py
  ├── (no args)     → GUI chat
  ├── smoke-test    → headless bootstrap + memory
  ├── acceptance-test → 15-check ship gate
  └── diagnose-test → keys, Venice probe, exe paths
```

Frozen: `Nedra.exe` with same argv routing.

### Startup sequence

1. Runtime hook sets `NEDRA_EDITION=consumer`, disables Chroma telemetry
2. `consumer_bootstrap()` — create `%LOCALAPPDATA%\Nedra\`, load `.env`, write `assistant_config.runtime.yml` with **absolute** DB paths
3. `crash_log.install_crash_logging()` → `crash-last.txt`
4. Splash → `ChatApp` → first-run wizard if `.first_run_done` missing
5. `startup_check.require_brain_key_or_exit()` — block until Venice **or** Groq key saved
6. `PlainAssistant` init — memory vault, personality, walk `brain.fallback_chain`

### GUI (`ChatApp`)

- **Sidebar:** sessions, + New chat, Notebooks, Settings
- **Chat:** read-only transcript; streaming via Tk `stream_mark`
- **Bottom bar:** Speak replies (default off), Brain combobox, Mic, Entry, Send
- **Threading:** worker daemon for inference; main thread polls queue every 80ms; **never** read Tk variables from worker

### Assistant (`PlainAssistant`)

`process_text()` pipeline: echo guard → memory recall → personality system prompt → stream/generate → trim history → background `remember()`.

`switch_brain(type)` hot-swaps backend without clearing soul state.

Full detail: [NEDRA_ARCHITECTURE.md](./NEDRA_ARCHITECTURE.md).

---

## Personality and companion layer

Two layers:

1. **`profiles/nedra.json`** — trait sliders (warmth, directness, humor, etc.), identity rules, boundaries
2. **`assets/persona.json`** — edition contract, voice strings, `edition_overrides.consumer` rules

`PersonalityEngine` builds system prompt from profile + edition rules + recalled memory block. Personality is **prompt-side only** — identical regardless of active brain.

Consumer tone: warm, plain-spoken friend; not a therapist; no jargon about Docker/APIs unless asked.

Full detail: [NEDRA_PERSONALITY.md](./NEDRA_PERSONALITY.md).

---

## Local memory

| Store | Path | Role |
|-------|------|------|
| Graph | `%LOCALAPPDATA%\Nedra\consumer_memory.db` | Memory nodes, entities |
| Vectors | `%LOCALAPPDATA%\Nedra\chroma\` | Semantic similarity |
| Sessions | `%LOCALAPPDATA%\Nedra\sessions.json` | UI chat list (separate from vault) |
| Tasks | `%LOCALAPPDATA%\Nedra\tasks.db` | Optional task manager |

Consumer uses **HashEmbedder** (SHA256 word hashing → 384-dim) — instant, offline, weak on paraphrase. Upgrade path: Ollama `nomic-embed-text` on dev edition.

`_memory_lock` wraps recall (worker) and remember (background thread).

Full detail: [NEDRA_MEMORY.md](./NEDRA_MEMORY.md).

---

## Model providers and runtime switching

Consumer combobox: `kobold`, `venice`, `groq`. Kobold promoted to top if port 5001 responds.

| Key | Env var | Notes |
|-----|---------|-------|
| Venice | `VENICE_API_KEY` | Default; 60s client timeout |
| Groq | `GROQ_API_KEY` | Backup; may 401 — warn, don't block ship |
| Kobold | (none) | User runs KoboldCpp separately |

Worker calls `reload_consumer_secrets()` then `switch_brain(choice)` before each `process_text`. Status line shows **active** `brain_backend.name`.

**Critical GUI rules:**
- Capture `speak_var.get()` on main thread before worker (off-thread read crashes app)
- `request_id` stale guard — Escape / brain change ignores late Venice replies
- Busy watchdog — auto-unlock after timeout + 10s
- Brain dropdown stays enabled during thinking (user escape hatch)

**Security:** Template YAML must not contain hardcoded API keys — env only.

Full detail: [NEDRA_MODELS.md](./NEDRA_MODELS.md).

---

## Onboarding and settings

### First-run wizard (`first_run.py`)

Modal forest-themed dialog when `.first_run_done` absent:

1. Welcome + tagline
2. **Your name** (optional)
3. **Companion name** (default "Nedra")
4. **Venice API key** (recommended) and/or Groq
5. Writes `user.json`, `.env`, runtime config; creates marker file

### Startup gate (`startup_check.py`)

If no Venice and no Groq key: modal loop until key pasted or user quits. Consumer cannot assume Kobold-only.

### Settings dialog

Same forest chrome; edit names, API keys, re-save secrets + profile. Triggers `bootstrap_consumer()` or brain re-prep on save.

### Defaults (consumer)

- `voice: none` — Speak replies unchecked
- `brain.default: venice`
- `memory.backend: sqlite`, `embedding.backend: hash`

---

## Packaging and install (non-technical user)

**Ship artifact:** `Nedra-Portable.zip` (~280–290 MB)

```
Nedra.exe + _internal/
assistant_config.consumer.yml (template)
assets/, profiles/
Install-Nedra.ps1    → copies to %LOCALAPPDATA%\Nedra\App\, desktop shortcut
Uninstall-Nedra.ps1  → removes shortcut + App; KEEPS user data
README-FOR-*.txt
```

**User path A (test):** Unzip anywhere → double-click `Nedra.exe`  
**User path B (installed):** Run `Install-Nedra.ps1` → use desktop shortcut **Nedra**

Data always lives in `%LOCALAPPDATA%\Nedra\` regardless of exe location.

**Support trap:** Multiple exe copies (portable folder vs installed App) share same data but may differ in **build date**. `diagnose-test` reports all known exe paths and timestamps.

Build: `scripts/build_nedra_consumer.ps1` → PyInstaller `packaging/nedra.spec` → `scripts/package_for_consumer.ps1`.

Full detail: [NEDRA_PACKAGING.md](./NEDRA_PACKAGING.md).

---

## QA harness (automated — not the user)

| Command | Purpose | Output |
|---------|---------|--------|
| `Nedra.exe smoke-test` | Fast bootstrap + memory | `smoke-last.txt` |
| `Nedra.exe acceptance-test` | 15-group pre-ship gate | `acceptance-last.txt` |
| `Nedra.exe diagnose-test` | Keys, Venice probe, exe dates | `diagnose-last.txt` |

Exit `0` = pass. Agent/build loop should run acceptance before asking a human to click the GUI.

**Acceptance highlights:** Venice inference + stream, brain switch Venice/Kobold, GUI Venice send survives (window not destroyed), companion rename persists, Tk stream marks valid, echo filter, queue busy clears.

**Crash logging:** `crash-last.txt` on uncaught main or thread exceptions.

Full detail: [NEDRA_QA.md](./NEDRA_QA.md).

---

## Known issues and fixes (from audited codebase)

| Issue | Cause | Fix / policy |
|-------|-------|--------------|
| App exits after Venice send | `speak_var.get()` on worker thread | Capture speak flag on main thread in `_on_send` |
| Stuck on "thinking" | Invalid Tk mark `"stream"` or `_busy` never cleared | Use `stream_mark`; watchdog + request_id cancel |
| User tests "fixed" build but sees old bug | Desktop shortcut → older `%LOCALAPPDATA%\Nedra\App\` exe | Sync install copy; diagnose reports dates |
| Notebooks tab traps user | Auto-modal launcher on tab open | Launch only on explicit button click |
| Memory corruption / crash | Concurrent Chroma access | `_memory_lock` on recall/remember |
| Groq 401 | Invalid/expired key | WARN in acceptance; Venice primary |
| Weak memory recall | Hash embeddings | Document upgrade path; acceptable v1 tradeoff |
| Hardcoded key in template YAML | Dev convenience leaked | **Remove** — env-only for Nedra rebuild |

Full decision rationale: [NEDRA_DECISIONS.md](./NEDRA_DECISIONS.md).

---

## Visual identity

Forest sanctuary palette (sampled from splash reference):

| Name | Hex | Use |
|------|-----|-----|
| Forest Deep | `#182d25` | Background |
| Forest Sidebar | `#1a3228` | Session rail |
| Forest Panel | `#243d32` | Headers, cards |
| Gilt | `#dbb874` | Titles, accents |
| Cream | `#d4d0bc` | Body text |

Font: Segoe UI (UI), Georgia italic (quotes). Sidebar tree glyph + gilt product wordmark.

---

## Data on disk (consumer)

```
%LOCALAPPDATA%\Nedra\
├── .env                          # VENICE_API_KEY, GROQ_API_KEY
├── user.json                     # human_name, assistant_name
├── assistant_config.runtime.yml  # generated; absolute paths
├── sessions.json
├── consumer_memory.db
├── chroma\
├── tasks.db
├── .first_run_done
├── logs\<session_id>\session.log
├── crash-last.txt
├── acceptance-last.txt
├── diagnose-last.txt
└── smoke-last.txt

%LOCALAPPDATA%\Nedra\App\         # after Install script
├── Nedra.exe
└── _internal\
```

---

## Rebuild-from-scratch checklist

### Phase 1 — Core engine

- [ ] `main_gui_assistant.py` with argv routes (chat, smoke, acceptance, diagnose)
- [ ] `modules/nedra_env.py` — `NEDRA_EDITION`, `NEDRA_CONFIG`, `NEDRA_DATA_DIR`, legacy alias fallback
- [ ] `modules/consumer_bootstrap.py` — bootstrap lock, runtime yml, `.env`, `user.json`
- [ ] `modules/assistant_config.py` — YAML load via dpath
- [ ] `assistant_config.consumer.yml` — consumer defaults (no hardcoded keys)
- [ ] `modules/base_assistant.py` — PlainAssistant, memory lock, switch_brain, process_text
- [ ] `modules/brain_backends/*` — BaseBrainBackend, factory, venice/groq/kobold minimum
- [ ] `modules/memory_vault/*` — SQLite graph, Chroma, HashEmbedder, MemoryManager
- [ ] `modules/personality_engine/*` — engine, profile, identity, renderer
- [ ] `profiles/nedra.json` + `assets/persona.json`

### Phase 2 — GUI

- [ ] `modules/gui/nedra_theme.py` — forest palette
- [ ] `modules/gui/chat_app.py` — sessions, queue, stream, brain dropdown, thread rules
- [ ] `modules/gui/session_store.py`
- [ ] `modules/gui/first_run.py` + `startup_check.py`
- [ ] `modules/gui/splash.py`
- [ ] `modules/crash_log.py`

### Phase 3 — QA

- [ ] `modules/consumer_smoke.py`
- [ ] `modules/nedra_acceptance.py` (~15 tests)
- [ ] `modules/nedra_diagnose.py`
- [ ] pytest unit tests for memory, bootstrap, brains, personality

### Phase 4 — Package

- [ ] `packaging/nedra.spec` + `runtime_hook_nedra.py`
- [ ] `scripts/build_nedra_consumer.ps1`
- [ ] `scripts/package_for_consumer.ps1` → `Install-Nedra.ps1`, zip
- [ ] Consumer README (Venice signup, brain dropdown, data location)

### Phase 5 — Ship gate

```powershell
pyinstaller packaging/nedra.spec --noconfirm
.\dist\Nedra\Nedra.exe acceptance-test
.\dist\Nedra\Nedra.exe diagnose-test
```

Exit 0 on both before human GUI validation.

---

## Key module map (source tree)

```
engine/
├── main_gui_assistant.py          # entry
├── assistant_config.consumer.yml
├── assets/persona.json
├── profiles/nedra.json
├── packaging/nedra.spec
├── scripts/build_nedra_consumer.ps1
├── scripts/package_for_consumer.ps1
└── modules/
    ├── nedra_env.py
    ├── consumer_bootstrap.py
    ├── assistant_config.py
    ├── base_assistant.py
    ├── crash_log.py
    ├── consumer_smoke.py
    ├── nedra_acceptance.py
    ├── nedra_diagnose.py
    ├── brain_backends/
    │   ├── brain_factory.py
    │   ├── base_backend.py
    │   ├── venice_backend.py
    │   ├── groq_backend.py
    │   └── kobold_backend.py
    ├── memory_vault/
    │   ├── memory_manager.py
    │   ├── sqlite_graph_store.py
    │   ├── chroma_store.py
    │   └── memory_factory.py
    ├── personality_engine/
    │   ├── engine.py
    │   ├── profile.py
    │   └── identity.py
    └── gui/
        ├── chat_app.py
        ├── session_store.py
        ├── first_run.py
        ├── startup_check.py
        ├── nedra_theme.py
        ├── splash.py
        └── notebooks_panel.py      # optional research tab
```

---

## Explicit non-goals (v1 consumer)

- Facility/mobile wellness guardrails (separate product family)
- Docker or Neo4j required for install
- Bundled LLM weights
- Telegram bridge, morning briefing (business/dev only)
- Memory export UI
- Cross-platform polish (Windows-first)

---

## Optional research layer (dev)

**Open Notebook** (MIT) — homaged in legal notices, not rebranded on consumer box. Health check `http://127.0.0.1:5055/health`. Embed via `wryview` WebView2 when stack running; browser fallback always available. Not bundled in consumer portable zip.

---

## Product family context (do not merge codebases)

Nedra Desktop = personal desk, private, full capability. Facility wellness apps (NURA-class) = separate repos, public guardrails, guided mobile onboarding. Policy: **do not port facility red-team rules into Nedra consumer.**

---

## One-paragraph elevator pitch

Nedra Desktop is a forest-themed Windows companion that remembers you on your own machine. You name it, paste a Venice API key once, and talk in a single window with live streaming replies. The same personality and memories stay with you when you switch brains — Venice cloud, Groq backup, or your own Kobold server on port 5001. No Docker, no terminal, no reinstall to try a different model. Install from a zip or a one-click PowerShell script; your data lives in one familiar folder under AppData. Ship only after automated acceptance passes — the user should never be the debug loop.

---

*End of consolidated brief. Topic depth lives in the eight companion markdown files in this directory.*
