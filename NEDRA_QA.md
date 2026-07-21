# Nedra Desktop — QA and Automated Testing

**Audit source:** `nedra_acceptance.py`, `consumer_smoke.py`, `nedra_diagnose.py`, scripts/ (June 2026).

---

## Philosophy

Dave should not be the test harness. Every ship candidate must pass **headless** gates before a human double-clicks the exe.

Results persist to `%LOCALAPPDATA%\Nedra\`:

| File | Written by |
|------|------------|
| `smoke-last.txt` | smoke-test |
| `acceptance-last.txt` | acceptance-test |
| `diagnose-last.txt` | diagnose-test |
| `crash-last.txt` | uncaught exception hook |

---

## Entry points (frozen exe)

```powershell
Nedra.exe smoke-test
Nedra.exe acceptance-test
Nedra.exe diagnose-test
```

Exit code `0` = pass, `1` = fail (suitable for CI or agent loops).

Dev equivalent:

```powershell
$env:NEDRA_EDITION='consumer'
python main_gui_assistant.py acceptance-test
python main_gui_assistant.py diagnose-test
python main_gui_assistant.py smoke
```

---

## Smoke test (`consumer_smoke.py`)

**Purpose:** Fast post-build sanity (~10s without Groq brain test).

Checks:

1. `bootstrap_consumer()` succeeds
2. Runtime config: `edition == consumer`, `memory.backend == sqlite`, `embedding.backend == hash`
3. `MemoryManager.remember()` + `recall()` on SQLite (+ Chroma if available)
4. Optional: `NEDRA_SMOKE_BRAIN=1` + `GROQ_API_KEY` → live inference (skipped if key invalid)

Isolated mode: temp data dir via mock for CI.

---

## Acceptance suite (`nedra_acceptance.py`)

**Purpose:** Full pre-ship gate (~15 test groups, ~30–45s).

| # | Test | Validates |
|---|------|-----------|
| 1 | Bootstrap & config | Keys present, runtime yml |
| 2 | Session store | JSON CRUD |
| 3 | SQLite memory background | Thread-safe remember |
| 4 | Tk stream marks | `stream_mark` not invalid index |
| 5 | Venice inference + stream | Live API if key present |
| 6 | Groq key status | WARN if 401, not FAIL |
| 7 | Echo filter | Repeat "hello" not blocked after failed attempt |
| 8 | process_text full flow | Kobold→Venice switch + stream |
| 9 | Chat queue busy | done event clears `_busy` |
| 10 | Venice GUI send survives | Tk app + worker; window not destroyed |
| 11 | Brain dropdown | All three consumer brains in combobox |
| 12 | Input bar present | entry, send_btn exist |
| 13 | Companion rename | Settings path persists to yml + user.json |
| 14 | Kobold local | Port 5001 probe |
| 15 | Brain switch Venice / Kobold | `switch_brain()` true |

Failures collected in list; exit 1 if any FAIL (WARNs allowed).

---

## Diagnose (`nedra_diagnose.py`)

**Purpose:** Support script when user reports “it broke” — no GUI.

Reports:

- Timestamps of all known `Nedra.exe` locations
- Venice/Groq key presence (length only, not value)
- Direct Venice API probe (`Reply exactly: venice ok`)
- Kobold→Venice switch + `process_text` in PlainAssistant
- Tail of `crash-last.txt` if present
- Recommendation if desktop install older than test folder copy

---

## PowerShell ship scripts

| Script | Role |
|--------|------|
| `smoke_test_consumer.ps1` | Python smoke + exe smoke |
| `preflight_ship.ps1` | Unit tests + smoke + zip extract + exe smoke |
| `ship_queue.ps1` | Ordered gate sequence |
| `simulate_*_install.ps1` | Extract zip, run exe smoke |
| `build_nedra_consumer.ps1` | Build + implicit validation |

---

## Unit tests (dev, pytest)

Engine root and `tests/`:

- `test_sqlite_memory.py`
- `test_consumer_bootstrap.py`
- `test_brain_backends.py`
- `test_personality_engine.py`
- `test_memory_vault_smoke.py`
- `test_stability.py`
- `tests/test_integration.py`

Preflight runs subset before zip ships.

---

## Crash logging (`crash_log.py`)

Installed at GUI startup:

- `sys.excepthook` → write traceback to `crash-last.txt`
- `threading.excepthook` → background thread crashes

Essential for diagnosing “app closed after Venice” without user repro steps.

---

## Known bugs found via QA (fixed in audited codebase)

| Bug | Symptom | Fix |
|-----|---------|-----|
| Invalid Tk index `"stream"` | Stuck on “thinking” | Use `insert` mark for `stream_mark` |
| `speak_var.get()` on worker thread | Full app exit on Venice send | Capture speak flag on main thread |
| UI busy forever | Frozen after cloud hang | Watchdog timer + Escape cancel + request_id stale guard |
| Double bootstrap race | Rare corruption / hang | Bootstrap lock + worker uses reload_secrets only |
| Chroma + recall concurrent | Possible native crash | `_memory_lock` on remember/recall |
| Notebooks tab modal on open | User trapped in launcher error | Remove auto `ensure_notebooks_running` on tab show |

---

## Agent-driven ship loop (recommended)

```
build → acceptance-test (exe) → diagnose-test → copy to test folder → optional Dave 5-min GUI
```

Agent should not ask Dave to test until acceptance exits 0.

---

## Rebuild QA module names

| Current audited name | Nedra rename |
|---------------------|--------------|
| `arden_acceptance.py` | `nedra_acceptance.py` |
| `arden_diagnose.py` | `nedra_diagnose.py` |
| `consumer_smoke.py` | keep or `nedra_smoke.py` |

Wire in `main_gui_assistant.py` argv routing and PyInstaller `hiddenimports`.

---

## Minimal CI recipe

```yaml
- pip install -e . -r requirements-gui.txt pyinstaller
- pyinstaller packaging/nedra.spec --noconfirm
- ./dist/Nedra/Nedra.exe acceptance-test
```

Secrets: inject `VENICE_API_KEY` from CI vault for live Venice tests.
