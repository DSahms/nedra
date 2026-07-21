# Nedra Desktop — Decisions Log

**Audit source:** Project docs, code comments, conversation history, ship notes (June 2026).  
**Format:** Decision → rationale → tradeoff.

---

## Product positioning

### D1 — Personal desk, not facility app

**Decision:** Nedra Desktop is separate from facility/mobile wellness products (NURA-class apps).

**Why:** Facility apps need public guardrails and guided onboarding; personal desk is private, full-capability, no red-team rules ported.

**Tradeoff:** Two codebases to maintain; no shared compliance layer.

---

### D2 — Soul vs brain split

**Decision:** Personality + memory are “soul”; LLM provider is “brain” — swappable per message via dropdown.

**Why:** User shouldn't reinstall or re-enter API keys to try local vs cloud. Signature product differentiator.

**Tradeoff:** Must keep brain switch reliable; status UI must reflect active backend not YAML default.

---

### D3 — Default companion name = product name

**Decision:** Default assistant name matches product (Nedra); user can rename.

**Why:** Brand clarity for consumer; power users personalize via Settings.

---

## Technology

### D4 — Tkinter over Electron

**Decision:** Native Tk/ttk GUI, custom forest theme.

**Why:** Small portable zip, no Chromium bundle, Windows-native feel, stdlib-adjacent.

**Tradeoff:** Ugly without custom theme work; not cross-platform polished (Windows-first ship).

---

### D5 — SQLite + Chroma for consumer memory (not Neo4j)

**Decision:** Consumer `memory.backend: sqlite`; Neo4j only for dev/business.

**Why:** Brittany cannot run Docker. Portable install is non-negotiable.

**Tradeoff:** Weaker graph queries than Neo4j; good enough for v1 personal memory.

---

### D6 — Hash embeddings for consumer

**Decision:** `HashEmbedder` instead of Ollama embeddings on consumer path.

**Why:** No GPU, no Ollama install, instant, offline.

**Tradeoff:** Recall quality is approximate; upgrade path documented for Ollama/API embeddings.

---

### D7 — Venice default, Groq backup, Kobold optional

**Decision:** Consumer YAML defaults to Venice; Groq as free backup; Kobold when :5001 up.

**Why:** Venice privacy positioning + Llama 3.3 70B; Groq for cost-sensitive; Kobold for Dave/power users with local GPU stack.

**Tradeoff:** Consumer needs *some* cloud key unless Kobold always running.

---

### D8 — OpenAI-compatible brain interface

**Decision:** All major backends speak chat-completions shape (Kobold, Venice, Groq, etc.).

**Why:** One streaming code path in GUI; factory pattern for new providers.

---

### D9 — PyInstaller one-folder bundle

**Decision:** `dist/Nedra/` folder not single-file exe.

**Why:** Faster startup; Chroma/rust DLLs easier to load; zip portable distribution.

**Tradeoff:** ~280MB; user sees `_internal` folder.

---

### D10 — Exclude heavy voice ML from consumer exe

**Decision:** Exclude torch, realtimestt, elevenlabs from spec.

**Why:** Size and CUDA dependency; consumer ships `voice: none`.

**Tradeoff:** Full voice stack requires dev install or future optional pack.

---

## Security and config

### D11 — API keys in `.env` not exe

**Decision:** `VENICE_API_KEY`, `GROQ_API_KEY` in `%LOCALAPPDATA%\Nedra\.env`.

**Why:** Keys rotatable without rebuild; user-owned machine.

**Mistake to fix on rebrand:** Template YAML must not contain hardcoded inference keys — env only.

---

### D12 — Runtime config generated at bootstrap

**Decision:** Ship template yml; write `assistant_config.runtime.yml` with absolute paths on first run.

**Why:** Relative `data/` paths break when cwd changes (exe vs shortcut vs portable folder).

---

### D13 — Canonical NEDRA_* env vars

**Decision:** `NEDRA_EDITION`, `NEDRA_CONFIG`, `NEDRA_DATA_DIR`, etc. with legacy alias read fallback during migration.

**Why:** Stale brand names in env confuse developers six months out.

---

## GUI and UX

### D14 — Chat transcript read-only

**Decision:** User types only in bottom entry bar; chat pane is display-only.

**Why:** Avoid Tk text widget edit confusion; streaming marks simpler.

---

### D15 — Brain dropdown stays enabled while thinking

**Decision:** Don't disable combobox on busy; allow switch + cancel.

**Why:** User escape hatch when cloud hangs; reduces “trapped UI” support burden.

---

### D16 — Never touch Tk variables from worker thread

**Decision:** Capture `speak_var.get()` on main thread before starting worker.

**Why:** `RuntimeError: main thread is not in main loop` — observed full app exit on Venice send.

**Critical:** Any Nedra rebuild must audit all worker→GUI boundaries.

---

### D17 — Request ID stale guard

**Decision:** Monotonic `request_id`; cancel/Escape increments; worker results tagged and ignored if stale.

**Why:** Venice hang + user switches brain — old reply must not re-lock UI.

---

### D18 — Notebooks tab does not auto-modal on open

**Decision:** Removed auto `ensure_notebooks_running` when switching to Notebooks tab.

**Why:** Portable install lacks launcher script; modal blocked entire window.

---

## Onboarding

### D19 — Require Venice OR Groq before chat

**Decision:** `startup_check` blocks until at least one cloud key present (consumer).

**Why:** Consumer cannot assume Kobold. Must have inference path.

**Tradeoff:** Kobold-only users still need to paste a key once or we add “local only” bypass (not shipped).

---

### D20 — First-run wizard + Settings dialog share forest theme

**Decision:** Modal dialogs match main window aesthetic.

**Why:** Product polish; one visual language.

---

## Packaging and ship

### D21 — Zip + Install script over NSIS for v1

**Decision:** Primary ship = `Nedra-Portable.zip` + `Install-Nedra.ps1`.

**Why:** Fewer moving parts; PowerShell acceptable for one-time install.

---

### D22 — Shared LOCALAPPDATA data across exe copies

**Decision:** All exe locations use same `%LOCALAPPDATA%\Nedra\`.

**Why:** User doesn't lose memories when switching test vs installed copy.

**Tradeoff:** Confusion when exe builds differ but data persists — diagnose-test shows exe dates.

---

### D23 — Automated acceptance before human test

**Decision:** 15-test acceptance suite is ship gate; agent runs without Dave.

**Why:** Dave reported being the error in the loop; automation is policy.

---

## Open / research layer

### D24 — Open Notebook homaged, not rebranded

**Decision:** MIT Open Notebook in Notebooks tab; legal notices; not on consumer box face.

**Why:** Respect upstream license and brand; research layer is optional.

---

### D25 — wryview WebView2 embed optional

**Decision:** Embed when stack healthy; browser fallback always available.

**Why:** WebView2 not guaranteed; stack often offline on consumer desk.

---

## Explicit non-goals (v1 consumer)

- Telegram bridge
- Morning briefing
- Neo4j / Docker required path
- NotebookLM-style podcast generation in-box
- Play Store / mobile
- Multi-user / auth

---

## Rebrand note (Nedra)

This decisions log is extracted from a mature audited codebase. Rebuilding as **Nedra Desktop** should preserve decisions D2, D4–D8, D11–D18, D21, D23 unless product intent explicitly changes.

Rename surface area: env vars, data dir, profiles, persona strings, install scripts, theme module names — not the architectural decisions themselves.
