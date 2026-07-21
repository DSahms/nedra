# Nedra Desktop — Model Providers and Runtime Switching

**Audit source:** `brain_backends/`, `base_assistant.py`, `chat_app.py`, consumer YAML (June 2026).

---

## Core concept

**Soul stays. Brain swaps.**

The companion personality, memory, and session history are unchanged when the user picks a different entry in the Brain dropdown. Only the HTTP client speaking to an LLM changes.

---

## Supported backends

| Key | Local? | Consumer UI? | Default model (consumer) |
|-----|--------|----------------|--------------------------|
| `venice` | No | Yes | `llama-3.3-70b` @ `api.venice.ai` |
| `groq` | No | Yes | `llama-3.3-70b-versatile` |
| `kobold` | Yes (:5001) | Yes (if reachable) | `local-model` via KoboldCpp |
| `ollama` | Yes | Dev only | configurable |
| `lm_studio` | Yes | Dev only | configurable |
| `claude` | No | Dev only | Claude 3.5 Sonnet |
| `openai` | No | Dev only | GPT family |
| `gemini` | No | Dev only | Gemini |

Consumer combobox: `CONSUMER_BRAINS = ("kobold", "venice", "groq")`.

If KoboldCpp responds on port 5001, dropdown reorders to put **kobold first** (auto-prefer local).

---

## Configuration (YAML sections)

Each backend reads `get_config("{backend}.{key}")`:

| Key | Typical value |
|-----|---------------|
| `base_url` | API root |
| `model` | Model id string |
| `timeout` | HTTP seconds (Kobold 90, Groq 30, Venice 60 client-side) |
| `api_key` | **Should be env-only in production** — see security note |

Consumer template defaults:

```yaml
brain:
  default: venice
  fallback_chain: [kobold, venice, groq]

base_assistant:
  brain: venice
```

---

## API keys and secrets

| Backend | Env var | Onboarding |
|---------|---------|------------|
| Venice | `VENICE_API_KEY` | First-run wizard or Settings (recommended primary) |
| Groq | `GROQ_API_KEY` | Alternate / backup in wizard |
| Claude | `ANTHROPIC_API_KEY` | Dev only |
| OpenAI | `OPENAI_API_KEY` | Dev only |
| Gemini | `GEMINI_API_KEY` or `GOOGLE_API_KEY` | Dev only |
| Kobold | None | User runs KoboldCpp locally |

Keys stored in `%LOCALAPPDATA%\Nedra\.env` via `save_user_secrets()`.

`reload_consumer_secrets()` reloads env before each send (worker thread).

**Security decision:** Template YAML must not ship hardcoded production keys — use empty string + env injection only.

---

## BrainFactory

```python
BrainFactory.create_backend(type, config_dict)
```

- Core backends always registered
- Cloud extras lazy-imported (missing `anthropic` / `openai` → skip gracefully)
- Sets `backend.name` and `backend.is_local` from `LOCAL_BACKENDS` frozenset

---

## Initialization (`PlainAssistant._init_brain`)

Walk `brain.fallback_chain` in order:

1. Load backend-specific config keys from YAML
2. `create_backend(type, config)`
3. If `is_available()` → use as active brain
4. Else try next in chain

Consumer with no keys: may init Kobold if local server up, else Venice if key present, else Groq.

---

## Runtime switch (`switch_brain(type)`)

Called from:

- GUI Brain combobox `<<ComboboxSelected>>`
- GUI worker before each `process_text` (ensures dropdown matches inference)
- `_ensure_brain_ready()` on startup (100ms after UI build)
- `_prepare_brain` path on Settings save

Steps:

1. Load config for target backend
2. `BrainFactory.create_backend`
3. `is_available()` probe
4. Replace `self.brain_backend` — log old → new

**Does not** clear `conversation_history` or memory.

---

## Inference paths

### Blocking

`brain_backend.generate_response(system_prompt, user_message, conversation_history)`

### Streaming (GUI default)

When `chunk_callback` provided and `supports_streaming()` true:

`brain_backend.generate_response_stream(..., on_chunk=callback)`

Venice/Groq/Kobold all implement streaming. GUI accumulates chunks → queue → `_stream_chunk`.

Stream timeout: Venice stream loop capped at ~55s wall clock.

---

## GUI integration details

| Concern | Implementation |
|---------|----------------|
| Status line | Shows `assistant.brain_backend.name` after switch |
| Busy state | Input disabled during inference; brain dropdown stays enabled |
| Cancel | Escape bumps `request_id`, clears busy |
| Watchdog | Auto-unlock after `timeout + 10s` (60s cloud, 130s kobold) |
| Stale replies | Worker tags messages with `request_id`; ignored if user cancelled |
| Thread safety | Never read `speak_var` from worker — capture on main thread before worker start |

---

## Provider-specific notes

### Venice

- OpenAI SDK with custom `base_url`
- Privacy-oriented cloud; Llama 3.3 70B
- Primary consumer recommendation

### Groq

- Fast free tier; key validation may fail (401) — show WARN in acceptance, user uses Venice

### KoboldCpp

- OpenAI-compatible `/v1/chat/completions` with `stream: true`
- Probe: GET `/models` on base URL
- No API key; requires user to start server separately

---

## Fallback behavior

If selected brain unavailable at send time:

- Worker posts err message to chat: connection hint (port 5001 for Kobold, Settings for cloud)
- Does not auto-fallback silently — user explicitly picks another brain

Init-time fallback chain differs from per-message strictness by design (get app running vs user intent).

---

## Rebuild checklist

1. Implement `BaseBrainBackend` interface
2. Register backends in factory
3. Add YAML sections per provider
4. Wire `NEDRA_*` env for config path
5. Consumer combobox filter + Kobold reachability probe
6. Acceptance tests: `brain_switch_venice`, `brain_switch_kobold`, `venice_inference`, `venice_gui_send_survives`
