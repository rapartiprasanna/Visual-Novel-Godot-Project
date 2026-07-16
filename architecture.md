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
Main Menu ──File Select (3 slots)──► Intro sequence (until arrived_at_sect)
     │         empty → new run            │  departure → ambush → tutorial / hide end
     │         occupied → load file       │  → victory → sect arrival → hub unlock
     │                                    ▼
     │                              Home Hub (Sect Mountain)
     │                                    ├─► Training Grounds / Elder ──► Dialogue
     │                                    ├─► Mountain Gate ──► Combat + receipt
     │                                    ├─► Market Path ──► Shop overlay
     │                                    └─► Overlays: Profile / Chronicle / Settings
     └── Settings / Shop overlays (stubs)
```

**Intro routing (C3/C3b ✅):** new / mid-intro files route to `intro_departure`; hub unlocks only after `arrived_at_sect`. See `implementation-plan.md` → Intro Sequence Spec.

**Autoload:** `GameStateManager` persists across all scenes (like a Spring `@Service` singleton).

**Scene navigation:** all scene swaps go through `GameStateManager.navigate_to`, which **defers** `change_scene_to_file` to the next idle frame (coalesced). Never call `change_scene_to_file` synchronously from `finish_dialogue` / `finish_combat` / interlude `playback_finished` — doing so blanks the embedded game window and can break the debugger until Pause is pressed.

---

## Scene Registry (`GameStateManager`)

| `GameEnums.GameScene` | Scene path | Status |
|-----------------------|------------|--------|
| `MAIN_MENU` | `res://scenes/main_menu/main_menu.tscn` | ✅ Built |
| `HOME_HUB` | `res://scenes/home_hub/home_hub.tscn` | ✅ Built + routed |
| `DIALOGUE` | `res://scenes/dialogue/dialogue_scene.tscn` | ✅ Thin host (C2) |
| `COMBAT` | `res://combat/scenes/combat_arena.tscn` | ✅ Planning UI + full resolution |

**Entry point:** `run/main_scene` → Main Menu.

---

## Display / Window

Configured in `project.godot` / Project Settings → Display → Window:

| Setting | Value | Role |
|---------|-------|------|
| Viewport Width × Height | 1280 × 720 | Design resolution for layout |
| Window Width/Height Override | 1280 × 720 | Initial play window size |
| Stretch Mode | `canvas_items` | Scale 2D + Control UI when the window resizes |
| Stretch Aspect | `expand` | Fill the window (no letterbox; may show extra edges) |

HUD scenes use full-rect Control anchors (no `Camera2D`). Combat sidebar: scrollable body + Lock In / Retreat pinned so short windows don’t clip actions.

---

## Directory Structure (CultivationGame1)

```
core/
  game_state_manager.gd      # Autoload: navigation, profile, flags, dialogue/combat, save/load, calendar
  save_service.gd            # user:// JSON global meta + slot files
  consequence_engine.gd      # Applies DialogueConsequence mutations
  activity_resolver.gd       # C4: look-ahead collapse algorithm (§6.6)
  enums/game_enums.gd        # GameScene + HubContentType + ChronicleEventType
  dialogue/
    dialogue_playback_controller.gd  # C2: cursor / typewriter / choices (no GSM nav)
  data/
    player_profile.gd
    player_chronicle.gd      # Global encountered-event stub (C4: fed by commit_activity)
    run_calendar_state.gd    # C4: per-slot day / activity log / consumed event ids
    master_chronicle_event.gd / master_chronicle.gd  # C4: timed story event catalog
    place_catalog.gd         # C4: nested place matching for event triggers
    portrait_catalog.gd      # Speaker + expression → portrait texture
    hub_location_data.gd     # content_type + required_unlock_flag + time_cost_days
    hub_location_catalog.gd
    dialogue_script.gd       # Script / Line / Choice / Consequence resources
    dialogue_line.gd
    dialogue_choice.gd
    dialogue_consequence.gd
    dialogue_catalog.gd      # Demo scripts by trigger ID
    combat_receipt.gd        # Win/loss DTO back to hub

scenes/
  main_menu/
  home_hub/                  # Hotspots route by HubContentType (full-bleed today; map scroll Phase D)
  dialogue/                  # Thin host → DialogueTextboxView + PlaybackController

ui/
  dialogue/
    dialogue_textbox_view.tscn / .gd   # Reusable textbox (embeddable in combat overlays)
  overlays/
    settings_overlay.tscn
    shop_overlay.tscn
    player_profile_panel.tscn
    chronicle_calendar_panel.tscn
    save_slots_overlay.tscn    # FileSelectPanel on main menu

combat/
  data/   # actions, encounters, CombatInterludeTrigger
  core/   # TurnManager (+ checkpoints), ResolutionEngine, timeline…
  ui/     # grid, sidebar, CombatInterludeController, CombatDialogueOverlay
  scenes/ # combat_arena
  enums/
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

**Navigation rule:** `GameStateManager.navigate_to` is the only scene-change entry point; it queues `change_scene_to_file` via `call_deferred` so transitions out of dialogue/combat finish handlers stay safe.

**Rule:** Views read/write state through `GameStateManager` or dedicated data resources — never hardcode narrative text in layout nodes (per manifest Section 5).

---

## Main Menu Architecture

- Canvas `Control` root, centered button panel.
- **File Select** (`ui/overlays/save_slots_overlay.tscn` as `FileSelectPanel`): **3 always-visible files**.
  - Empty → `GameStateManager.select_file` starts a new run on that slot.
  - Occupied → loads that slot into hub.
- **Settings / Shop** = instanced overlay scenes; toggled visible, not separate scenes.
- No New Game / Continue / “most recent” shortcut.

---

## Home Hub Architecture

- **Background layer:** `assets/art/hub/sect_mountain.png` via full-bleed `TextureRect` (`stretch_mode` keep-aspect-covered).
- **Hotspot layer:** `HubLocationCatalog.get_demo_locations()` spawns buttons; each has `content_type`. Unlock reads `required_unlock_flag` via `GameStateManager.has_progression_flag` (e.g. Outer Slope → `trained_morning_drills`).
- **Routing:**
  - `DIALOGUE` → `GameStateManager.start_dialogue(trigger_id)`
  - `COMBAT` → `GameStateManager.start_combat(encounter_id)`
  - `SHOP` → show `ShopOverlay`
- **Persistent overlay (top-right):** Settings, Profile, Chronicle, Main Menu (no Save / slot picker).
- **Persistence:** autosave overwrites the **active file only** (hub enter / return-to-hub / Main Menu exit).
- **Info panel:** Location blurb; after combat, shows one-shot receipt summary.

**Demo hotspots:** Training Grounds, Elder's Pavilion, Meditation Chamber (seclusion), Mountain Gate (1v1), Outer Slope (1v2, gated), Market Path.

**Time costs (C4 ✅):** Training Grounds = 1 day (may collapse into `instructor_night_visit` from day 2 onward); Elder = instant; Meditation Chamber = 3-day seclusion (no event scan). Mountain Gate / Outer Slope / Market Path stay 0 for now.

**Planned — omnidirectional map scroll (Phase D):** today the mountain is a locked full-bleed frame. Future: scrollable/pannable map root larger than the viewport so the hub feels like moving around a place; surroundings beyond the mountain become extra hotspots. Persistent chrome stays on a fixed CanvasLayer. See `implementation-plan.md` → Phase D → Hub omnidirectional scroll.

---

## Dialogue Architecture

| Piece | Role | Status |
|-------|------|--------|
| `DialogueScript` | Ordered/branching lines keyed by `line_id` | ✅ |
| `DialogueLine` | Speaker, expression key, text, optional choices / next; flag gates | ✅ |
| `DialogueChoice` | Button text, next line, consequence; flag gates | ✅ |
| `DialogueConsequence` | Karma/qi/stat deltas, flags, combat, dialogue chain, return-to-menu | ✅ |
| `DialogueCatalog` | Maps trigger ID → script (hub + intro + tutorial interludes) | ✅ |
| `ConsequenceEngine` | Mutates `PlayerProfile` + progression flags; queues combat/dialogue/menu | ✅ |
| `DialoguePlaybackController` | Script cursor, typewriter, choices, `[playerName]`, name input; signal-only | ✅ C2/C3 |
| `DialogueTextboxView` | Reusable textbox UI; portrait / backlog / name `LineEdit`; binds to controller | ✅ C2/C3 |
| Dialogue scene | Thin host: feeds active script, finishes on `playback_finished` | ✅ C2 |
| `PortraitCatalog` | Maps speaker + expression → texture; Instructor one art; Elder wise/stern | ✅ |

**Demo scripts:** hub `training_grounds_intro` / `elder_audience`; intro spine + bandit tutorial interludes (C3b).

**Dialogue additions (C3 ✅):**
- **Name entry** (`DialogueLine.is_name_input` + `LineEdit`) → `PlayerProfile.player_name`
- **`[playerName]` substitution** in playback
- **`next_dialogue_script_id`** dialogue→dialogue chaining
- **Intro scripts** authored (color-stub portraits for escort / junior disciple)

---

## Combat Module Architecture

| Component | Role | Status |
|-----------|------|--------|
| `CombatTurnManager` | Planning, queue, NPC AI, leftover defaults, delegates resolve | ✅ |
| `CombatResolutionEngine` | Timeline resolve: blocks, interrupts, simultaneous damage, substitutes | ✅ |
| `CombatTimeline` | Per-fighter event schedule, block merging | ✅ Built |
| `DamageCalculator` | Base damage + layered mitigation (passive ratio + active block) + small RNG | ✅ Refactored |
| `CultivationScaler` | Realm-based base stat / title scaling utility (`core/data/cultivation_scaler.gd`) | ✅ Built (not yet wired into `PlayerProfile`) |
| `GridUtils` | Manhattan/Chebyshev, paths, beam lines | ✅ Built |
| `TurnPacingCalculator` | Fixed 3s demo; dynamic stub | ✅ Stub |
| `CombatAction` / `.tres` | Data-driven techniques (`block_skill_factor` on Block) | ✅ Built |
| `CombatPlanningSidebar` | Queue, points, palette, lock-in, mode; scroll + pinned Lock In / Retreat | ✅ Built |
| `CombatGridView` | 8×8 tiles, unit markers, targeting overlays | ✅ Built |
| Combat arena HUD | Planning → lock-in → resolve → receipt | ✅ |
| `EncounterCatalog` | 1v1 / 1v2 / `intro_bandit_tutorial`; interludes; palette ids (C1/C3) | ✅ |
| `CombatInterludeTrigger` | Checkpoint + dialogue id + tutorial gating fields | ✅ C3 |
| `CombatInterludeController` | Checkpoint pauses → overlay dialogue → resume / end | ✅ C3 |
| `CombatDialogueOverlay` | CanvasLayer dim + embedded `DialogueTextboxView` | ✅ C3 |
| Movement collision | Soft-block Walk: wait/retry while tile occupied; unfinished tiles → float | ✅ |
| Sustained beam | Duration ticks pulse damage for anyone on the line | ✅ |
| Resolution playback | Tweened unit travel / timed timeline playback | ❌ Deferred polish |
| NPC beam + FF policy | Policy fields on combatants; AI still Walk/Punch only | 🟡 Partial |

---

## Global State (current)

`GameStateManager` holds:
- `player_profile: PlayerProfile`
- `player_chronicle: PlayerChronicle` — global meta stub (encountered event ids; survives across files)
- `run_calendar: RunCalendarState` — per-slot day / activity log / consumed event ids (C4)
- `active_save_slot: int` — file chosen at File Select; all mid-run writes go here
- `current_scene: GameEnums.GameScene`
- `active_dialogue_trigger_id` / `active_encounter_id`
- `pending_combat_encounter_id` (dialogue → combat handoff)
- `pending_dialogue_script_id` (dialogue→dialogue or post-combat dialogue)
- `pending_return_to_main_menu` (intro hide ending)
- `last_combat_receipt: CombatReceipt`
- `progression_flags: Dictionary`

**Scene routing:** `navigate_to(scene)` updates `current_scene`, emits `scene_changed`, then applies `change_scene_to_file` on the next frame (`_apply_queued_scene_change`). Multiple `navigate_to` calls in one frame coalesce to the last target.

**Not yet present:** inventory, full NPC relationship system (demo has `escort_disciple_affection` stub only). Calendar clock / chronicle UI now built (C4) — village hubs, travel corridors, forced mains, and year-scale UI chrome remain Phase D per `system-prompt.md` §6.8.

**Save / load (Phase C step 9b ✅):**

| Piece | Path / role |
|-------|-------------|
| `SaveService` | `user://global_meta.json` + `user://saves/slot_N.json` |
| Slot contents | `PlayerProfile` + `progression_flags` + timestamp + `run_calendar` (C4, `SAVE_VERSION` 3) |
| Global meta | `PlayerChronicle` only |
| Entry UX | Main menu File Select: 3 always-visible files |
| Empty file | New run bound to that slot (immediate write) |
| Occupied file | Load that slot |
| Autosave | Hub enter, return-to-hub, Main Menu exit → **active slot only** |
| Hub Save UI | Removed |
| Delete file | Deferred past demo |

---

## Combat Interlude & Tutorial Architecture (Phase C3 ✅)

Mid-combat dialogue overlay system. The intro's bandit tutorial fight is the first consumer; same mechanism later serves story dialogue / cutscenes inside fights. Detailed checklist in `implementation-plan.md` (step C3).

**Prerequisite (C2):** ✅ dialogue playback extracted (`DialoguePlaybackController` + `DialogueTextboxView`).

**Interlude pieces (built):**
| Piece | Role |
|-------|------|
| `CombatInterludeTrigger` | Trigger type + `dialogue_script_id` + tutorial gating fields |
| `EncounterCatalog.get_interludes(id)` | Per-encounter trigger list; empty for normal fights |
| `CombatInterludeController` | Checkpoint pauses → overlay → resume / deferred combat end |
| `CombatDialogueOverlay` | CanvasLayer dim + embedded `DialogueTextboxView` |

**Boundary rules (honored):**
- `CombatResolutionEngine` untouched; `CombatTurnManager` exposes `turn_number` + checkpoint signals and `continue_after_resolution()` / `force_combat_finished()`.
- Interlude scripts are ordinary `DialogueScript`s; consequences via `ConsequenceEngine`.
- Sidebar stays dumb: palette filter + lock-in gate fed by the interlude controller.

**Tutorial encounter:** `intro_bandit_tutorial` — Walk/Punch/Block palette; 3 gated turns; HP-below rescue; post-fight → `intro_bandit_victory`.

---

## Calendar / Chronicle Architecture (Phase C4 ✅)

Design authority: `system-prompt.md` Section 6.

| Piece | Role | Status |
|-------|------|--------|
| `MasterChronicleEvent` / `MasterChronicle` (`core/data/master_chronicle_event.gd`, `core/data/master_chronicle.gd`) | Authoring truth: place, trigger range, duration, type, content hook (`dialogue_script_id`) | ✅ 1 demo event (`instructor_night_visit`) |
| `PlaceCatalog` (`core/data/place_catalog.gd`) | Nested place hierarchy for event matching (parent → any child; siblings never match) | ✅ |
| `PlayerChronicle` (global meta save, unchanged from earlier phase) | Cross-run encountered events → future Calendar markers | ✅ now fed by `commit_activity` |
| `RunCalendarState` (`core/data/run_calendar_state.gd`, per save slot) | Current day, activity log, consumed event ids | ✅ persisted via `SaveService` (`SAVE_VERSION` 3) |
| `ActivityResolver` (`core/activity_resolver.gd`) | Look-ahead collapse, date advance | ✅ |
| `GameStateManager.commit_activity` / `commit_seclusion` | Hub entry points into the resolver / direct calendar advance | ✅ |
| `ChronicleCalendarPanel` (`ui/overlays/chronicle_calendar_panel.gd`) | Single UI merging run past + PlayerChronicle future (color + italic "faded" markers) | ✅ |

**Nested places:** parent match includes children; siblings do not. Travel roads (Phase D) are place nodes too; `PlaceCatalog.PARENTS` currently maps all demo hub locations under `sect_mountain`.

**Hub time costs (demo):** `elders_pavilion` = 0 (instant, skips the calendar entirely); `training_grounds` = 1 day. `mountain_gate` / `outer_slope` / `market_path` stay 0 for now (day costs TBD with future calendar content). `meditation_chamber` (new, `HubContentType.SECLUSION`) defaults to 3 days.

**Advance rule on overlap:** `ActivityResolver.resolve` scans `[T0, T0+D)` for an unconsumed `MasterChronicleEvent` matching the activity's place; on a hit, `advance = days_until_first_overlap + event.duration_days`, with lead-in days logged as the chosen activity (no generic empty beat) before the event's own log entry. No overlap → flavor text + `D`-day advance. Home hub prefers the triggered event's `dialogue_script_id` over the location's normal `narrative_trigger_id` when one fires.

**Seclusion (`GameStateManager.commit_seclusion`):** intentionally bypasses `ActivityResolver` — it never scans for events (misses ordinary story beats by design per §6.6 step 1), just advances `run_calendar.current_day` directly, then the hub plays the location's cutscene dialogue (`seclusion_meditation`) whose consequence carries the stat bump.

**Regression seeds:** `tests/calendar/activity_resolver_seed_test.gd` (instant/no-advance, flavor advance, event collapse + one-time consumption, place nesting) and `tests/calendar/save_migration_readonly_test.gd` (v2→v3 migration against real save slots, read-only — never writes back).

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

- **Consequence readers:** ✅ C1 — profile→combat; hub/dialogue flag gates; hub `arrived_at_sect` gate ✅ with intro.
- **Dialogue playback extraction:** ✅ C2 — embeddable textbox used by combat interludes.
- **Combat interludes + bandit tutorial + intro plumbing/content:** ✅ C3 / C3b (art still color stubs; optional escort portraits later).
- **Calendar / Chronicle UI wiring:** ✅ C4 — `RunCalendarState`, `MasterChronicle`, `PlaceCatalog`, `ActivityResolver`, `ChronicleCalendarPanel` all built; see Calendar / Chronicle Architecture above.
- **Shop + inventory:** Overlay is a stub; item resources / buy flow / save payload not built (Phase C step C6).
- **Minimal combat resolution playback:** deferred UI polish (**next: C5**).
- **Hub omnidirectional map scroll:** Phase D.
- **Qi cost / mastery / cooldown:** Spec only.
- **Portrait art (partial):** Instructor + Elders wired; intro cast uses color stubs.
- **Save file delete:** Deferred past demo.
- **Save migration scaffold:** ✅ `SaveService.migrate` identity for v1.
