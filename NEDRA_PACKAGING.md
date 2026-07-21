# Nedra Desktop — Packaging and Install

**Audit source:** `packaging/`, `scripts/`, `release/` (June 2026).

---

## Ship artifact

**Primary consumer deliverable:** `Nedra-Portable.zip`

Contents (flat or single folder):

- `Nedra.exe` — PyInstaller one-folder bundle
- `_internal/` — Python runtime, chromadb, dependencies
- `assistant_config.consumer.yml` — reference template
- `assets/`, `profiles/` — persona resources
- `Install-Nedra.ps1` — copies to `%LOCALAPPDATA%\Nedra\App`, creates desktop shortcut
- `Uninstall-Nedra.ps1` — removes shortcut + App folder; **keeps** user data
- `README-FOR-*.txt` — non-technical instructions
- `THIRD_PARTY_NOTICES.txt` — ChromaDB, Open Notebook homage, licenses

Target size: ~280–290 MB (Chroma + ML deps dominate).

---

## Build pipeline

```
scripts/build_nedra_consumer.ps1
    → pip install pyinstaller
    → pip install -r requirements-gui.txt  (Pillow, wryview)
    → pyinstaller packaging/nedra.spec --noconfirm
    → dist/Nedra/Nedra.exe

scripts/package_for_consumer.ps1
    → generate Install/Uninstall scripts into release/
    → copy README, notices
    → Compress-Archive → release/Nedra-Portable.zip
```

---

## PyInstaller spec (key decisions)

| Setting | Value | Why |
|---------|-------|-----|
| Entry | `main_gui_assistant.py` | Single exe entry; routes smoke/acceptance/diagnose |
| `datas` | consumer yml, assets, profiles | Runtime template + persona files |
| `hiddenimports` | brain backends, memory, GUI, chromadb, nedra_env, crash_log | Dynamic imports |
| `collect_all(chromadb)` | Yes | Rust bindings + telemetry modules |
| `collect_all(wryview)` | Yes | WebView2 optional embed |
| `runtime_hooks` | Set NEDRA_EDITION, disable telemetry | Consumer mode at launch |
| `excludes` | torch, realtimestt, elevenlabs | Size; consumer doesn't need heavy voice ML |
| `console` | False | Windowed app — no black terminal |
| Output | `dist/Nedra/Nedra.exe` | One-folder layout (faster than one-file) |

---

## Runtime hook

On exe start before main:

```python
NEDRA_EDITION=consumer
NEDRA_CONFIG=assistant_config.consumer.yml  # bundled template name
ANONYMIZED_TELEMETRY=False
```

Legacy env aliases mirrored one release cycle if migrating from older brand.

---

## Install flow (non-technical user)

### Path A — Portable (recommended for testing)

1. Unzip `Nedra-Portable.zip` to any folder (e.g. `D:\Test two\`)
2. Double-click `Nedra.exe`
3. Data still goes to `%LOCALAPPDATA%\Nedra\` — portable refers to exe location only

### Path B — Installed (Brittany-style)

1. Unzip to Downloads or Desktop
2. Right-click `Install-Nedra.ps1` → Run with PowerShell
3. Script copies binaries to `%LOCALAPPDATA%\Nedra\App\`
4. Creates desktop shortcut **Nedra.lnk** → `Nedra.exe`
5. User double-clicks shortcut for daily use

**No terminal after install.** No Docker. No Neo4j.

---

## Uninstall

`Uninstall-Nedra.ps1`:

- Removes desktop shortcut
- Deletes `%LOCALAPPDATA%\Nedra\App\`
- Does **not** delete memories, sessions, `.env`, or logs

README must state: delete `%LOCALAPPDATA%\Nedra\` to wipe all personal data.

---

## Bootstrap on first launch

Even without Install script:

1. `consumer_bootstrap()` creates `%LOCALAPPDATA%\Nedra\`
2. Writes `assistant_config.runtime.yml` with absolute DB paths
3. Loads or creates `.env` and `user.json`
4. `SessionStore` creates default session in `sessions.json`

---

## Multiple install copies problem (document for support)

Users may have:

| Location | Typical use |
|----------|-------------|
| `%LOCALAPPDATA%\Nedra\App\Nedra.exe` | Desktop shortcut target |
| `D:\Test two\Nedra.exe` | Dev/test portable extract |
| `dist\Nedra\Nedra.exe` | Build output |

**All share the same `%LOCALAPPDATA%\Nedra\` data.**

Different **build dates** cause “I fixed it but nothing changed” if shortcut points to old exe.

`diagnose-test` reports exe paths and timestamps — use in support.

---

## Optional NSIS installer

`packaging/installer.nsi` exists for `Nedra_Setup.exe` — optional; zip + Install script is primary consumer path.

---

## Dev edition packaging (not consumer zip)

- `docker-compose.yml` for Neo4j
- `launcher.py` for full stack
- `main_base_assistant.py` terminal mode
- Open Notebook native launcher batch at repo root
- Not shipped to non-technical consumer

---

## Third-party notices (required)

- Open Notebook — MIT, homaged in docs/legal, not consumer-branded on box face
- ChromaDB — bundled in exe
- SurrealDB / other research deps — only if Notebooks stack installed separately

---

## Rebuild file checklist

| File | Purpose |
|------|---------|
| `packaging/nedra.spec` | PyInstaller |
| `packaging/runtime_hook_nedra.py` | Edition env |
| `scripts/build_nedra_consumer.ps1` | Build |
| `scripts/package_for_consumer.ps1` | Zip + install scripts |
| `release/Install-Nedra.ps1` | Generated installer |
| `Nedra.bat` | Dev launcher `NEDRA_EDITION=consumer` |

---

## Consumer README bullets

- Sign up at venice.ai for API key (or Groq backup)
- First run: name companion, your name, paste key
- Brain dropdown: Venice recommended; Kobold if local server running
- Notebooks tab: optional research — not required for chat
