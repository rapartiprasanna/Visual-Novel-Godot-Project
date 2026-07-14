# Cultivation Game — Codebase Architecture

Condensed session notes. **Game design rules** live in `system-prompt.md`. This file documents **how the Godot project is organized** and what is wired today.

---

## Two-Project Layout

| Project | Path | Role |
|---------|------|------|
| **Manifest / AI context** | `~/visual-novel-godot-project/` | `system-prompt.md`, planning docs, future GitHub repo |
| **Godot game** | `~/Documents/GodotGames/cultivation-game-1/` | Runnable CultivationGame1 (Godot 4.6) |

---

## High-Level Flow

```
Main Menu ──File Select (3 slots)──► Home Hub (Sect Mountain)
     │         empty → new run            │
     │         occupied → load file       ├─► Training Grounds / Elder ──► Dialogue
     │                                    ├─► Mountain Gate ──► Combat (scaffold + receipt)
     │                                    ├─► Market Path ──► Shop overlay
     │                                    └─► Overlays: Profile / Chronicle / Settings
     │                                        (autosave to active file; no hub slot picker)
     └── Settings / Shop overlays (stubs)
```

**Autoload:** `GameStateManager` persists across all scenes (like a Spring `@Service` singleton).

---

## Scene Registry (`GameStateManager`)

| `GameEnums.GameScene` | Scene path | Status |
|-----------------------|------------|--------|
| `MAIN_MENU` | `res://scenes/main_menu/main_menu.tscn` | ✅ Built |
| `HOME_HUB` | `res://scenes/home_hub/home_hub.tscn` | ✅ Built + routed |
| `DIALOGUE` | `res://scenes/dialogue/dialogue_scene.tscn` | ✅ MVP |
| `COMBAT` | `res://combat/scenes/combat_arena.tscn` | ✅ Planning UI + full resolution |

**Entry point:** `run/main_scene` → Main Menu.

---

## Directory Structure (CultivationGame1)

```
core/
  game_state_manager.gd      # Autoload: navigation, profile, flags, dialogue/combat, save/load
  save_service.gd            # user:// JSON global meta + slot files
  consequence_engine.gd      # Applies DialogueConsequence mutations
  enums/game_enums.gd        # GameScene + HubContentType
  data/
    player_profile.gd
    player_chronicle.gd      # Global encountered-event stub
    hub_location_data.gd     # Includes content_type
    hub_location_catalog.gd
    dialogue_script.gd       # Script / Line / Choice / Consequence resources
    dialogue_line.gd
    dialogue_choice.gd
    dialogue_consequence.gd
    dialogue_catalog.gd      # Demo scripts by trigger ID
    combat_receipt.gd        # Win/loss DTO back to hub

scenes/
  main_menu/
  home_hub/                  # Hotspots route by HubContentType
  dialogue/                  # Textbox, typewriter, choices, backlog, portrait stub

ui/overlays/
  settings_overlay.tscn
  shop_overlay.tscn          # Also opened from Market Path
  player_profile_panel.tscn
  chronicle_calendar_panel.tscn
  save_slots_overlay.tscn    # Exists; retarget to Main Menu File Select (remove from hub)

combat/
  data/ core/ actions/ scenes/ enums/
  ui/
    combat_grid_view.gd
    combat_planning_sidebar.gd
  core/
    combat_turn_manager.gd
    combat_resolution_engine.gd
    combat_timeline.gd
    ...
```

---

## Architecture Patterns (Java → Godot mapping)

| Concept | Godot implementation |
|---------|---------------------|
| **DTO / config beans** | `Resource` subclasses (`PlayerProfile`, `CombatAction`, `DialogueScript`, …) |
| **Application service** | `GameStateManager` autoload |
| **Command / mutation service** | `ConsequenceEngine` |
| **Enum state machine** | `GameEnums.GameScene`, `HubContentType`, `CombatEnums.*` |
| **View / controller** | Scene `.tscn` + thin `.gd` script; no game rules in UI |
| **Factory / catalog** | `HubLocationCatalog`, `DialogueCatalog`, `ActionLibrary` |
| **Module boundary** | `combat/` returns `CombatReceipt` via `GameStateManager.finish_combat` |

**Rule:** Views read/write state through `GameStateManager` or dedicated data resources — never hardcode narrative text in layout nodes (per manifest Section 5).

---

## Main Menu Architecture

**Target (spec):** Hollow Knight / SMG2 file select — see Save / load below.

**As implemented today (incorrect; to rework):**
- Canvas `Control` root, centered button panel.
- **Continue** loads most recent slot; **New Game** ignores slot pick.
- Slots 2–3 gated by `has_completed_first_run`.

**Intended:**
- Main menu shows **3 always-visible save files** (inline or dedicated File Select overlay).
- Empty slot → `PlayerProfile.create_default()` → bind `active_save_slot` → hub.
- Occupied slot → `load_game(slot)` → hub.
- **Settings / Shop** = instanced overlay scenes; toggled visible, not separate scenes.
- No “Continue most recent” shortcut required for the demo.

---

## Home Hub Architecture

- **Background layer:** placeholder `ColorRect` bands (replace with static mountain image later).
- **Hotspot layer:** `HubLocationCatalog.get_demo_locations()` spawns buttons; each has `content_type`.
- **Routing:**
  - `DIALOGUE` → `GameStateManager.start_dialogue(trigger_id)`
  - `COMBAT` → `GameStateManager.start_combat(encounter_id)`
  - `SHOP` → show `ShopOverlay`
- **Persistent overlay (top-right):** Settings, Profile, Chronicle, Main Menu. **No Save / slot picker** (remove `SaveSlotsOverlay` from hub).
- **Persistence:** autosave overwrites the **active file only** (hub enter / return-to-hub / Main Menu exit).
- **Info panel:** Location blurb; after combat, shows one-shot receipt summary.

**Demo hotspots:** Training Grounds, Elder's Pavilion, Mountain Gate (1v1), Outer Slope (1v2), Market Path.

**Time costs (per §6, when calendar ships):** Training Grounds = 1 day; Elder = instant.

---

## Dialogue Architecture

| Piece | Role |
|-------|------|
| `DialogueScript` | Ordered/branching lines keyed by `line_id` |
| `DialogueLine` | Speaker, expression key, text, optional choices / next |
| `DialogueChoice` | Button text, next line, consequence |
| `DialogueConsequence` | Karma/qi/cultivation deltas, flags, optional queued combat |
| `DialogueCatalog` | Maps hub `narrative_trigger_id` → script |
| `ConsequenceEngine` | Mutates `PlayerProfile` + progression flags |
| Dialogue scene | Typewriter textbox, choice buttons, history panel, color portrait stub |

**Demo scripts:** `training_grounds_intro` (linear + 2 choices), `elder_audience` (3-way branch).

---

## Combat Module Architecture

| Component | Role | Status |
|-----------|------|--------|
| `CombatTurnManager` | Planning, queue, NPC AI, leftover defaults, delegates resolve | ✅ |
| `CombatResolutionEngine` | Timeline resolve: blocks, interrupts, simultaneous damage, substitutes | ✅ |
| `CombatTimeline` | Per-fighter event schedule, block merging | ✅ Built |
| `DamageCalculator` | Ratio damage + block mitigation + small RNG | ✅ Built |
| `GridUtils` | Manhattan/Chebyshev, paths, beam lines | ✅ Built |
| `TurnPacingCalculator` | Fixed 3s demo; dynamic stub | ✅ Stub |
| `CombatAction` / `.tres` | Data-driven techniques (`block_skill_factor` on Block) | ✅ Built |
| `CombatPlanningSidebar` | Queue, points, palette, lock-in, mode | ✅ Built |
| `CombatGridView` | 8×8 tiles, unit markers, targeting overlays | ✅ Built |
| Combat arena HUD | Planning → lock-in → resolve → receipt | ✅ |
| `EncounterCatalog` | 1v1 / 1v2 combatant presets by encounter id | ✅ |
| Movement collision | Soft-block Walk: wait/retry while tile occupied; unfinished tiles → float | ✅ |
| Sustained beam | Duration ticks pulse damage for anyone on the line | ✅ |
| Resolution playback | Tweened unit travel / timed timeline playback | ❌ Deferred polish |
| NPC beam + FF policy | Policy fields on combatants; AI still Walk/Punch only | 🟡 Partial |

---

## Global State (current)

`GameStateManager` holds:
- `player_profile: PlayerProfile`
- `player_chronicle: PlayerChronicle` — global meta stub (encountered event ids; survives across files)
- `active_save_slot: int` — file chosen at File Select; all mid-run writes go here
- `current_scene: GameEnums.GameScene`
- `active_dialogue_trigger_id` / `active_encounter_id`
- `pending_combat_encounter_id` (dialogue → combat handoff)
- `last_combat_receipt: CombatReceipt`
- `progression_flags: Dictionary`

**Remove / stop using for slot UX:** `has_completed_first_run` as a gate that unlocks files 2–3. (Flag may remain in global meta briefly during rework, but must not lock slots.)

**Not yet present:** inventory, NPC relationship metrics, calendar clock / full chronicle UI (design locked in `system-prompt.md` §6).

**Save / load — target (Phase C rework):**

| Piece | Path / role |
|-------|-------------|
| `SaveService` | `user://global_meta.json` + `user://saves/slot_N.json` |
| Slot contents | `PlayerProfile` + `progression_flags` + timestamp (+ run calendar when §6 ships) |
| Global meta | `PlayerChronicle` only (no slot-lock flag) |
| Entry UX | Main menu File Select: 3 always-visible files |
| Empty file | New run bound to that slot |
| Occupied file | Load that slot |
| Autosave | Hub enter, return-to-hub, Main Menu exit → **active slot only** |
| Hub Save UI | **Removed** — no multi-slot picker on hub |
| Delete file | Deferred past demo (clear slot → empty again) |

**As implemented today (incorrect):** New Game / Continue + hub `SaveSlotsOverlay` + `has_completed_first_run` locking slots 1–2. Replace with File Select above.

---

## Calendar / Chronicle Architecture (designed; not built)

Design authority: `system-prompt.md` Section 6. Intended split:

| Piece | Role |
|-------|------|
| `MasterChronicle` (Resources / catalog) | Authoring truth: place, trigger range, duration, type, recurrence, content hook |
| `PlayerChronicle` (global meta save) | Cross-run encountered events → future Calendar markers |
| Run calendar state (per save slot) | Current date, consumed events, activity log |
| `ActivityResolver` (service) | Look-ahead overlap, travel priority, seclusion policy, date advance |
| `ChronicleCalendarPanel` | Single UI merging run past + PlayerChronicle future (color + faded borders) |

**Nested places:** parent match includes children; siblings do not. Travel roads are place nodes.

**Hub time costs (demo):** Elder instant; Training Grounds costs 1 day. Other day-cost activities added with calendar work.

**Advance rule on interrupt:** `days_until_first_overlap + event.duration`. Lead-in days logged as the chosen activity without playing generic empty beats.

---

## UI Overlay Pattern

All overlays extend `PanelContainer`, share:
- `show_overlay()` / `hide_overlay()` / `closed` signal
- Centered modal layout
- Close button

`PlayerProfilePanel` subscribes to `GameStateManager.player_profile_changed` for live refresh.

`ChronicleCalendarPanel` is currently a static placeholder; wire to run calendar + PlayerChronicle when Phase C calendar ships.

---

## Pending Infrastructure

- **File Select rework:** Replace New Game/Continue + hub slot picker + first-run locks with title-screen 3-file select (see Save / load target).
- **Calendar / Chronicle UI wiring:** Spec’d in §6; clock/resolver not built (PlayerChronicle stub persists now).
- **Qi cost:** Field reserved on `CombatAction`; not enforced.
- **Mastery / cooldown:** Designed in spec; not in code.
- **Portrait art:** Expression keys exist; textures not yet.
