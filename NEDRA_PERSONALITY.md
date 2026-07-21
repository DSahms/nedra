# Nedra Desktop — Personality and Companion Layer

**Audit source:** `personality_engine/`, `profiles/`, `assets/persona.json`, consumer config (June 2026).

---

## Design intent

Nedra is not a raw LLM wrapper. The **soul** is a persistent companion identity: name, trait sliders, edition-specific voice lines, boundaries, and rules — independent of which model generates tokens.

Tagline for the product: **Keep the soul. Change the brain.**

Default companion name: **Nedra** (user-renamable in onboarding and Settings).

---

## Two-layer personality model

### Layer 1 — Profile JSON (`profiles/nedra.json`)

Structured trait object:

| Field | Type | Purpose |
|-------|------|---------|
| `name` | string | Display name in UI and prompts |
| `warmth`, `directness`, `humor`, `formality`, `empathy`, `patience` | 0.0–1.0 floats | Trait sliders fed into prompt renderer |
| `identity_rules` | string[] | Hard identity constraints |
| `boundaries` | string[] | Safety / scope limits (not a therapist, etc.) |

Consumer profile skews: high warmth (0.8), high directness (0.85), moderate humor, low formality.

### Layer 2 — Persona contract (`assets/persona.json`)

Cross-edition contract with:

- `identity.description` — who the companion claims to be
- `identity.never_claims_to_be` — blocklist (GPT, Claude, chatbot, etc.)
- `identity.always_identifies_as` — canonical self-description
- `traits` — base defaults (can be overridden per edition)
- `voice` — greeting, farewell, acknowledgment, encouragement, concern strings
- `edition_overrides` — per-edition trait tweaks, voice lines, `additional_rules`

---

## Edition overrides

| Edition | Personality shift |
|---------|-------------------|
| **consumer** | Warmer, less formal, plain-spoken; rules forbid jargon about Docker/APIs unless asked; “helpful friend who remembers” |
| **business** | Higher directness, lower humor; chief-of-staff tone; morning briefing integration |
| **wellness** | Higher empathy/patience; recovery-aware rules (separate product policy — not ported to consumer guardrails) |

Consumer edition loads `edition: "consumer"` from YAML → `PersonalityEngine.load_edition_overrides("consumer")`.

---

## PersonalityEngine (`engine.py`)

**Init:** Load `PersonalityProfile` from JSON + load `persona.json` contract.

**`load_edition_overrides(edition)`:**

1. Read `persona.edition_overrides[edition].traits` → mutate profile floats
2. Load `voice` dict → greetings, farewells used by GUI and `get_greeting()`
3. Append `additional_rules` to system prompt

**`get_system_prompt(memory_context)`:**

Combines:

- Identity rules from profile
- Edition additional rules
- Trait-derived behavioral guidance (via renderer)
- Injected `[Relevant past conversations]` block when memory recall returns context

**`get_greeting()`:** Returns edition-appropriate opener; GUI shows on new/empty sessions.

---

## Identity application at runtime

`identity.apply_user_identity(profile, assistant_name, human_name)`:

- Updates profile name and companion naming in prompts
- Sets personalized greeting: `"Hey {human}. What's on your mind?"` when human name saved
- Triggered after Settings save and bootstrap profile load

---

## Where personality touches the UI

| Surface | Behavior |
|---------|----------|
| Window title | "Nedra Desktop" |
| Header | User-chosen assistant name (large gilt title) |
| Sidebar | "NEDRA" + product tagline |
| Chat labels | Speaker name tag on transcript (`nedra_name` / `user_name` tags) |
| Status pulse | "{Name} is thinking…" during inference |
| First-run wizard | Default assistant name field pre-filled "Nedra" |

---

## Voice config (personality, not TTS)

`voice_config` from edition overrides is **copy** for greetings/farewells — distinct from `base_assistant.voice` (none/local/elevenlabs) which controls audio output.

Consumer ships with `voice: none` — personality voice strings appear in text only.

---

## Boundaries (consumer)

From profile + persona rules:

- Not a therapist or medical professional
- No diagnosis or prescriptions
- Encourage professional help when appropriate
- No cross-user information sharing
- Plain language; no exposing implementation details to end user

---

## Rebuild checklist for Nedra

1. Create `profiles/nedra.json` with Nedra identity rules
2. Update `assets/persona.json` — `always_identifies_as` → Nedra Desktop companion
3. Set `personality.default_profile: "nedra"` in consumer YAML
4. Rename `get_nedra_profile()` in profile loader
5. Wire `edition_overrides.consumer` rules unchanged in spirit
6. Update GUI theme strings (tagline, quotes) in theme module
7. First-run default `DEFAULT_ASSISTANT_NAME = "Nedra"`

---

## What personality does not do

- Does not select or configure LLM backends (Brain dropdown)
- Does not store long-term memories (Memory vault does)
- Does not manage API keys (bootstrap + Settings)

Personality is **prompt-side only** — same engine regardless of Venice, Groq, or Kobold.
