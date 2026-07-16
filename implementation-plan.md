# Cultivation Game — Implementation Plan

Condensed roadmap from design sessions. **Combat mechanics detail** is in `system-prompt.md` Section 4. This file tracks **what is done, what is next, and recommended order**.

---

## Completed ✅

### Project setup
- [x] Manifest repo at `~/visual-novel-godot-project` with `system-prompt.md` (Sections 1–6, full combat spec)
- [x] Godot project `CultivationGame1` at `~/Documents/GodotGames/cultivation-game-1`
- [x] Local git commit on manifest (GitHub push pending `gh auth login`)

### Core app shell
- [x] `GameStateManager` autoload + `GameEnums`
- [x] `PlayerProfile` resource
- [x] Main Menu scene (File Select ×3, Settings/Shop stubs, Quit)
- [x] Home Hub scene (`sect_mountain.png` background, hotspots, persistent overlay)
- [x] Reusable UI overlays (Settings, Shop, Profile, Chronicle; File Select panel on main menu)
- [x] Save / load — `SaveService`, 3 file slots, global `PlayerChronicle` meta, File Select entry

### Phase A — Core loop connected
- [x] `DialogueScript` / `DialogueLine` / `DialogueChoice` / `DialogueConsequence` resources
- [x] `DialogueCatalog` demo scripts (`training_grounds_intro`, `elder_audience`)
- [x] `ConsequenceEngine` mutates profile + progression flags
- [x] Hub hotspot routing by `HubContentType` (dialogue / combat / shop)
- [x] Dialogue scene MVP: typewriter, choices, backlog, portrait color stub
- [x] Mountain Gate → combat; stub HUD returns `CombatReceipt` to hub
- [x] Market Path opens shop overlay on hub

### Combat scaffold (logic only)
- [x] `CombatAction`, `CombatantState`, `QueuedCombatAction` resources
- [x] Starter techniques: Walk, Punch, Block, Beam, Blink (code + `.tres`)
- [x] `CombatTimeline` with block-chain merging
- [x] `DamageCalculator`, `GridUtils`, `TurnPacingCalculator`
- [x] `CombatTurnManager` — queue API, basic NPC AI, stub resolution
- [x] `combat_arena.tscn` demo 1v1 bootstrap + Phase A receipt HUD

### Phase B — Combat UI / resolution
- [x] Planning sidebar — points, HP, skill palette, queue list, undo/clear, lock-in, default mode, retreat
- [x] 8×8 `CombatGridView` — tile clicks, unit markers (P/E), Walk/Blink/Beam highlight overlays
- [x] Wire palette → `CombatTurnManager` queue; remove Simulate Win/Loss stub
- [x] Full resolution engine (step 7) — blocks, interrupts, simultaneous damage, float + default substitutes
- [x] 1v2 encounter (step 8) — Outer Slope hub + EncounterCatalog

### Phase C groundwork (partial)
- [x] **C1 consequence readers** — profile→combat stats; flag→hub unlock (`outer_slope` / `trained_morning_drills`); flag→dialogue (`chose_power_path` opening)
- [x] **C2 dialogue playback extraction** — `DialoguePlaybackController` + `DialogueTextboxView`; `dialogue_scene` thin host
- [x] **Display / combat HUD polish** — 1280×720 base + `canvas_items`/`expand` stretch; combat sidebar scrolls with Lock In / Retreat pinned visible
- [x] **C3 combat interludes + bandit tutorial + intro plumbing** — see checklist below
- [x] **C3b intro content pass** — departure → ambush → tutorial/hide → victory → sect arrival; hub gated on `arrived_at_sect`
- [x] **C-infra save migration** — `SaveService.migrate` v1→v2 maps legacy `cultivation_level` into might/swiftness defaults
- [x] **Deferred scene navigation** — `GameStateManager.navigate_to` coalesces `change_scene_to_file` via `call_deferred` (fixes blank game window after combat / intro hide ending)
- [x] **Damage mitigation refactor** — `DamageCalculator` replaced the "0 if not blocking" single-ratio mitigation with two additive layers per `cultivation_progression_system.md` §3: always-on **Passive Defense Ratio** (`calculate_passive_mitigation`) + supplementary **Active Block** (`calculate_active_block_mitigation`, penetration-counterable), summed/capped in `calculate_final_mitigation`. `calculate_final_damage` now floors at 1, never 0. No call-site signature changes (`combat/core/combat_resolution_engine.gd` unchanged).
- [x] **`CultivationScaler` utility** (`core/data/cultivation_scaler.gd`) — realm curve (`MajorRealm` enum, `calculate_base_stat`, `get_cultivation_title`) per `cultivation_progression_system.md` §1–2. Not yet wired into `PlayerProfile`/`CombatantState`; demo stats remain hand-authored flat ints. Wiring it into character sheets (and/or NPC templates for realms above Qi Gathering) is future work.

### Phase C4 — Calendar / Chronicle demo slice
- [x] **`RunCalendarState`** (`core/data/run_calendar_state.gd`) — per-slot `current_day`, `activity_log`, `consumed_event_ids`; persisted via `SaveService` (`SAVE_VERSION` bumped 2→3, `migrate` v2→v3 defaults old saves to day 1 / empty log).
- [x] **`MasterChronicleEvent` + `MasterChronicle`** (`core/data/master_chronicle_event.gd`, `core/data/master_chronicle.gd`) — one demo event: `instructor_night_visit` at `training_grounds`, trigger window day 2–6, duration 1, type `SIDE_STORY`, hooks `training_grounds_night_visit` dialogue.
- [x] **`PlaceCatalog`** (`core/data/place_catalog.gd`) — nested place matching (parent event fires at any child place; siblings never match) per §6.4.
- [x] **`ActivityResolver`** (`core/activity_resolver.gd`) — look-ahead collapse algorithm (§6.6): scans `[T0, T0+D)` for an unconsumed matching event; collapses into it (`advance = days_until_overlap + event.duration`, lead-in days logged as the activity, no generic empty beat) or falls back to flavor text + `D`-day advance. Instant activities (`time_cost_days == 0`) skip the calendar entirely.
- [x] **`GameStateManager.commit_activity` / `commit_seclusion`** — hub entry points; `commit_activity` runs the resolver and returns the triggered event (if any) so the hub can play its dialogue instead of the location's usual trigger; `commit_seclusion` advances the calendar directly with no event scan (§6.6 step 1) then plays a cutscene dialogue with a stat-bump consequence.
- [x] **Hub wiring**: `HubLocationData.time_cost_days` (Training Grounds = 1 day, Elder's Pavilion = 0/instant); new `meditation_chamber` hub location (`HubContentType.SECLUSION`, 3-day default) routes through `commit_seclusion`.
- [x] **Dialogue content**: `training_grounds_night_visit` (event payload, small karma + spirit reward) and `seclusion_meditation` (cutscene, spirit + willpower reward) added to `DialogueCatalog`.
- [x] **`PlayerChronicle` future markers**: `commit_activity` records triggered event ids into the existing global `PlayerChronicle` stub for next-run faded markers.
- [x] **`ChronicleCalendarPanel` rewired** — real `RichTextLabel` body: current day, this-run activity log (color-coded by `GameEnums.ChronicleEventType`), and a "Foreseen (from a prior run)" section reading `PlayerChronicle` ∩ unconsumed `MasterChronicle` events (italicized, partial-reveal spoiler title only).
- [x] Headless regression seeds: `tests/calendar/activity_resolver_seed_test.gd` (instant no-advance, flavor advance, event collapse + one-time consumption, place nesting) + `tests/calendar/save_migration_readonly_test.gd` (validates v2→v3 migration against real save slots, read-only).
- [ ] Deferred within C4 scope: multi-day look-ahead lead-in is implemented but not yet exercised by any >1-day activity (all costed activities are 1 day); second day-costing activity beyond Training Grounds; richer Calendar UI chrome (dedicated per-entry coloring widgets vs. BBCode).

---

## In Progress / Stub 🟡

| Item | Gap |
|------|-----|
| Combat arena | ✅ Planning HUD + grid + full resolve + 1v1/1v2; sidebar scroll + pinned Retreat |
| Combat resolution | ✅ Engine live; visual playback still deferred (C5 minimal / Phase D full) |
| Display / window | ✅ 1280×720 + stretch (`canvas_items` / `expand`); tweak in Project Settings → Display → Window |
| NPC AoE policy | Fields set; beam AI not queued yet (no NPC beam in demo kit AI) |
| Settings / Shop | Placeholder text only |
| Chronicle Calendar | ✅ C4: real `RunCalendarState` clock, `ActivityResolver` look-ahead collapse, `MasterChronicle`/`PlayerChronicle`-backed UI; village hubs/travel/forced mains remain Phase D |
| **Save system** | ✅ File Select: 3 always-visible files on main menu; active-file autosave; no hub picker; delete deferred |
| Hub art | ✅ `assets/art/hub/sect_mountain.png` full-bleed; hotspots anchored to landmarks |
| **Hub map scroll** | ❌ Planned — omnidirectional pan so the hub feels like a map (world around the mountain); see Phase D / Hub scroll note |
| Portraits | 🟡 Partial: `instructor1`, `sect_elder_wise`, `sect_elder_stern` via `PortraitCatalog`; intro cast uses color stubs |
| **Intro sequence** | ✅ Spine playable (name entry, ambush choice, tutorial fight, hide ending, sect arrival → hub) |
| **Scene transitions** | ✅ Deferred `navigate_to` — no blank embedded window after combat / hide ending |

---

## Recommended Build Order

### Phase A — Connect the core loop ✅
1. ~~Hub → Dialogue routing~~
2. ~~Dialogue scene MVP~~
3. ~~Hub → Combat routing~~
4. ~~Combat receipt → hub~~

### Phase B — Combat playable ✅
5. ~~**Combat UI** — Sidebar (queue, point budget, skill palette, lock-in, undo, aggressive/defensive toggle).~~
6. ~~**Grid view** — 8×8 tile rendering, unit markers, Walk/Blink/Beam targeting overlays.~~
7. ~~**Full resolution pass** — Floating points, default behaviors, interrupts, block mitigation.~~
8. ~~**1v2 encounter** — Outer Slope hub + `EncounterCatalog` presets.~~

### Phase C — Persistence & polish (REORDERED)

**Rationale for reorder (architecture review):** the demo's unique value is (1) calendar-driven loop and (2) timeline combat that players can actually learn. Old order shipped shop/settings before either. New order closes the consequence loop first, builds the tutorial + interlude system on an extracted dialogue component, then calendar, then economy/settings.

Completed groundwork:
9. 🟡 ~~**Save / load plumbing**~~ → corrected by 9b.
9b. ✅ **File Select rework**:
    - [x] Main menu: replace New Game / Continue with **3 always-visible save files**
    - [x] Empty file → new run bound to that slot; occupied → load that slot
    - [x] Remove hub Save button + `SaveSlotsOverlay` from hub (panel retargeted to main menu File Select)
    - [x] Drop `has_completed_first_run` slot gating; all 3 files available from first launch
    - [x] Autosave / mid-run write → **active file only**
    - [x] Slot **delete** deferred past demo
10. 🟡 **Static art pass**
    - [x] Mountain hub background (`assets/art/hub/sect_mountain.png`) + hotspot re-anchors
    - [x] Partial portraits wired via `PortraitCatalog` + dialogue texture swap
      - Instructor → `instructor1.jpeg` (all expressions until variants exist)
      - Sect Elder → `sect_elder_wise` / `sect_elder_stern` (`neutral` falls back to wise)
    - [ ] Optional later: Instructor expression variants + Elder `neutral` dedicated art

Remaining Phase C, in order:

#### C1. Consequence readers — make choices matter ✅
Flags and profile stats are written but nothing reads them. Close the loop with three readers:
- [x] **Profile → combat:** `EncounterCatalog.build_player_from_profile(profile)` copies the full stat block via `CombatantState.from_player_profile` (swiftness → action budget; might/physique/spirit/warding for damage channels). Call site: `combat/scenes/combat_arena.gd` `_setup_encounter()`.
- [x] **Flags → hub:** `HubLocationCatalog` honors `is_unlocked` from `GameStateManager.has_progression_flag` (e.g. `outer_slope` locked until `trained_morning_drills`). Touch: `core/data/hub_location_catalog.gd`, `scenes/home_hub/home_hub.gd`.
- [x] **Flags → dialogue:** at least one branch in `elder_audience` that checks a prior flag (e.g. `chose_power_path` changes the opening line). Requires a small `required_flag` / `blocked_by_flag` field on `DialogueLine` or `DialogueChoice` + filter in the dialogue view. Touch: `core/data/dialogue_line.gd` or `dialogue_choice.gd`, `core/data/dialogue_catalog.gd`, `scenes/dialogue/dialogue_scene.gd`.

**Key files:** `combat/data/encounter_catalog.gd`, `core/data/hub_location_catalog.gd`, `core/data/dialogue_catalog.gd`.

#### C2. Dialogue playback extraction (prerequisite for C3) ✅
Today `scenes/dialogue/dialogue_scene.gd` fuses playback logic with the full-screen scene and hard-calls `GameStateManager.finish_dialogue()`. Extract so the same textbox can be embedded anywhere (combat overlay, future cutscenes):
- [x] `DialoguePlaybackController` (`core/dialogue/dialogue_playback_controller.gd`, RefCounted or Node): owns script/line cursor, typewriter counters, choice handling, consequence application. Emits signals only: `line_shown(line)`, `choices_shown(choices)`, `typing_finished`, `playback_finished(script_id)`. **Never** touches `GameStateManager` navigation.
- [x] `DialogueTextboxView` (`ui/dialogue/dialogue_textbox_view.tscn` + `.gd`): reusable Control — portrait (via `PortraitCatalog` + color fallback), speaker label, RichTextLabel body, continue hint, choice container, optional backlog. Binds to a controller instance.
- [x] Rewrite `scenes/dialogue/dialogue_scene.gd` as a thin host: instantiate view + controller, feed `GameStateManager.get_active_dialogue_script()`, on `playback_finished` call `GameStateManager.finish_dialogue()`. Full-scene behavior unchanged.
- [ ] Regression check: `training_grounds_intro` and `elder_audience` play identically (typewriter, choices, backlog, consequences, dialogue→combat handoff). *(manual in editor)*

**Key files:** `scenes/dialogue/dialogue_scene.gd` (shrinks), new `core/dialogue/` + `ui/dialogue/`.

#### C3. Combat interludes + tutorial fight (uses C2) ✅
Mid-combat dialogue system, designed to be reused for story beats / cutscenes inside any future fight. See **Combat Interlude & Tutorial Checklist** below for full detail.

> **Re-scoped (intro decision):** the tutorial encounter is the **bandit ambush fight** inside the intro sequence (see **Intro Sequence Spec**). `training_spar_tutorial` naming is superseded by `intro_bandit_tutorial`.

- [x] `CombatInterludeTrigger` resource + `InterludeTriggerType` enum
- [x] `CombatInterludeController` node in combat arena + `CombatDialogueOverlay` (embeds `DialogueTextboxView`)
- [x] Checkpoint signals on `CombatTurnManager` (no changes inside `CombatResolutionEngine`)
- [x] Tutorial encounter `intro_bandit_tutorial` (wounded bandit, low HP) launched from intro dialogue via existing `start_combat_encounter_id`
- [x] Lock-in gating for tutorial objectives (queue Walk → Punch → Block across scripted turns)
- [x] Intro plumbing: player name entry (`LineEdit`), `player_name` on `PlayerProfile`, `[playerName]` substitution, `next_dialogue_script_id` chaining, intro→hub unlock flag

#### C3b. Intro content pass (uses C1 + C2 + C3) ✅
Authoring pass for the opening sequence. Full beat-by-beat detail in **Intro Sequence Spec** below.
- [x] Scripts: `intro_departure`, `intro_bandit_ambush`, `intro_hide_ending`, `intro_bandit_victory`, `intro_sect_arrival` (+ dialogue-driven entrance exam)
- [x] New saves route to `intro_departure` instead of `HOME_HUB`
- [x] Hub gated behind `arrived_at_sect` flag
- [ ] Art: escort man portrait, bandit portrait, carriage/road background, (optional) sect gate background *(color stubs for now)*
- [x] Affection stub: `escort_disciple_affection` on `PlayerProfile`

#### C4. Calendar / Chronicle demo slice (was step 12 — **promoted**) ✅
The retention hook; must ship in demo. See Calendar / Chronicle Checklist below; full rules in `system-prompt.md` §6. Includes one missable timed event that records to `PlayerChronicle` and shows as a faded future marker on a second run.

#### C5. Minimal combat resolution playback (pulled forward from Phase D, minimal scope)
Not the full wall-clock scrub — just enough motion to sell the timeline model:
- [ ] Tween unit markers tile-by-tile on Walk/Blink (fixed short per-step duration, not wall-clock accurate)
- [ ] Brief visual flash on block windows and interrupt cancels
- [ ] Combat log lines highlighted as their step plays
- Interlude checkpoints (C3) must be respected between playback steps so tutorials/cutscenes can pause playback later.

**Key files:** `combat/ui/combat_grid_view.gd`, `combat/scenes/combat_arena.gd`; resolution engine stays logic-instant (playback is a replay of the log/timeline, keeping data/view split).

#### C6. Shop + inventory (was step 11 — demoted behind loop-critical work)
- [ ] Item resources + inventory on `PlayerProfile` (or dedicated inventory state)
- [ ] Wire Market Path / Shop overlay to real stock + buy flow
- [ ] Persist inventory in save slot payload (bump `SAVE_VERSION`, see C7-infra)

#### C7. Settings persistence — audio/display prefs.

#### C-infra (anytime, cheap): save migration scaffold
- [x] `SaveService.migrate(data: Dictionary) -> Dictionary` keyed on `version`; identity for v1. All future payload additions (inventory, calendar state) go through it.

### Phase D — Post-demo (explicitly deferred)
- **Entrance-exam minigames** (1–2 bespoke minigames at the sect exam; demo uses dialogue-driven exam instead — see Intro Sequence Spec)
- **NPC relationship / dating system** (junior disciple becomes dateable; demo ships only the `escort_disciple_affection` stub counter)
- Additional portrait expression variants (Instructor stern/neutral split, Elder neutral, etc.)
- Save file **delete** (clear a slot to start a brand-new run on that file)
- Perception / Comprehension intel stats
- Qi cost enforcement, technique mastery, cooldowns
- Player-controlled allies, co-op, PvP
- Dynamic turn pacing algorithm
- Strictly defensive default mode
- Rich activity tables / sub-profession minigames / village hubs / travel random tables
- Forced main events (auto hub + break seclusion)
- Year-scale calendar chrome (large `D` seclusion still supported by engine)
- **Full** wall-clock resolution playback / timeline scrub (minimal tween pass moved into demo as C5)
- Story cutscenes mid-combat with camera moves / scripted enemy actions (interlude system from C3 is the foundation)
- Combat regression test suite expansion (seed tests land with C3; broaden before combat features grow)
- `GameStateManager` service split (`RunSessionState` / `SceneRouter` / `PersistenceService` / `ChronicleService`) — do **before** post-demo content expansion
- Catalog migration to `.tres` / data files (`DialogueCatalog`, `HubLocationCatalog`, `EncounterCatalog`) — do **before** calendar content volume grows
- **Hub omnidirectional map scroll** (see note below) — lands when surrounding-world art/locations expand; can ship a light pan even before new zones exist

#### Hub omnidirectional scroll (planned — Phase D / world-expansion polish)

**Goal:** the Home Hub should feel like a **map you can move around**, not a single full-bleed postcard. Pan in all directions (drag / edge pan / WASD or arrows) within a larger canvas so players sense space around the sect mountain — even a modest overscan feels better than a locked frame. Later, forest / village / road landmarks around the mountain become reachable hotspots on the same scrollable layer.

**Design notes (not built):**
- Keep hotspots as anchored `%` positions on a **map root** larger than the viewport (e.g. `Control` or `Node2D` map with camera / scroll offsets).
- Persistent overlay (Settings / Profile / Chronicle / Main Menu) stays **fixed on a CanvasLayer** — it must not scroll with the map.
- Clamp scroll to map bounds; optional gentle inertia; optional “center on mountain” reset.
- Current `sect_mountain.png` can overscan slightly first; expand art + `HubLocationCatalog` anchors when new surroundings are authored.
- **Not blocking C3–C7.** Schedule when adding world around the mountain, or as a small feel-pass after the intro unlocks the hub.

---

## Combat Interlude & Tutorial Checklist (Phase C, step C3)

Goal: pause combat at defined checkpoints, play dialogue over the arena, resume. The intro's bandit tutorial fight is the first consumer; the same system later drives story dialogue / cutscenes mid-fight.

### Design principles
- **Combat logic never knows about dialogue.** `CombatResolutionEngine` stays untouched. Pauses happen only at checkpoints that `CombatTurnManager` / the arena scene already control (phase boundaries, and later, playback step boundaries in C5).
- **Overlay, not scene change.** Combat state lives in the arena scene tree; navigating to the dialogue scene would destroy it. The interlude renders `DialogueTextboxView` (from C2) in a CanvasLayer above the arena.
- **Data-driven.** Interludes are resources attached to an encounter, not code in the arena script. A story fight and a tutorial differ only in trigger data + script content.

### Data layer
- [x] `InterludeTriggerType` enum (`combat/enums/combat_enums.gd`):
      `COMBAT_START`, `PLANNING_START` (turn N), `AFTER_RESOLVE` (turn N), `HP_BELOW` (combatant %, checked after resolve), `COMBATANT_DEFEATED`, `PLAYER_ACTION_QUEUED` (action id, for tutorial nudges), `COMBAT_END`
- [x] `CombatInterludeTrigger` resource (`combat/data/combat_interlude_trigger.gd`)
- [x] `EncounterCatalog.get_interludes(encounter_id)` (empty for existing encounters)
- [x] Interlude dialogue scripts in `DialogueCatalog`

### Runtime layer
- [x] `CombatTurnManager`: `turn_number` + checkpoint signals + `continue_after_resolution()` / `force_combat_finished()`
- [x] `CombatInterludeController` (`combat/ui/combat_interlude_controller.gd`)
- [x] `CombatDialogueOverlay` (`combat/ui/combat_dialogue_overlay.tscn`)
- [x] Pause semantics: defer next phase / `finish_combat` until overlay closes

### Bandit tutorial fight
- [x] Encounter `intro_bandit_tutorial` + wounded bandit + Walk/Punch/Block palette + non-lethal HP rescue
- [x] Tutorial gating fields + sidebar palette filter / lock-in hint
- [x] Scripted 3-turn Walk → Punch → Block flow + free play + `COMBAT_END` wrap-up
- [x] Entry from `intro_bandit_ambush` "Fight with the others"; return → `intro_bandit_victory`
- [x] Beam/Blink not tutorialized

### Combat regression seed tests
- [x] Headless seed under `tests/combat/combat_resolution_seed_test.gd` (block-chain merge + walk path); expand later

---

## Intro Sequence Spec (Phase C, step C3b) — DO NOT LOSE

The demo opens with a scripted linear sequence **before** the hub unlocks: leaving home → bandit ambush (choice) → tutorial fight or bad ending → sect arrival + entrance exam → hub. New saves start here, not at the hub. This section is the authoring source of truth for the intro; quoted lines are locked unless deliberately rewritten.

### Scope decisions (made 2026-07-14)
- **In demo:** the full departure → bandit → tutorial → arrival spine. It reuses existing systems (dialogue, choices, consequences, combat handoff) plus the C3 interlude/tutorial work, so it is cheap and makes the demo feel like a game instead of a systems sampler.
- **Cut from demo:** the 1–2 entrance-exam **minigames**. Brand-new single-use mechanics are the scope trap. The exam is **dialogue-driven** in the demo (branching interview with the elders — both portraits already exist), with an **optional** second combat encounter as a "combat trial" if an interactive beat is wanted (reuses the arena for free). Real minigames → Phase D.
- **Affection is a stub:** winning the tutorial bumps `escort_disciple_affection` (simple counter or flag on `PlayerProfile`). No relationship system in demo — NPC affection metrics remain Phase D. The dateable NPC is **not** in the demo; the counter just preserves the consequence for later.

### Cast (intro)
| Character | Role | Portrait |
|-----------|------|----------|
| **Escort man** ("Senior Escort", name TBD) | Gruff sect representative collecting recruits; delivers tutorial lines from the sidelines; delivers the hide-path rejection | ❌ New art needed |
| **Junior disciple** (name TBD) | Young recruit traveling in the same carriage; the future dateable NPC; affection target of the tutorial win | ❌ New art needed (can defer to color stub; never shown prominently in demo) |
| **Wounded bandit** | Tutorial opponent | ❌ New art needed (combat marker only if needed; portrait optional) |
| **Sect elders** | Entrance exam interviewers | ✅ `sect_elder_wise` / `sect_elder_stern` exist |

### New backgrounds
- Village road / carriage exterior (scenes 1–2; one image can serve both)
- Optional: sect gate exterior for arrival (can reuse `sect_mountain.png` at first)

### Scene 1 — `intro_departure` (leaving home)
1. Escort man: **"So you are the kid coming with us. What's your name?"**
2. **Name entry step:** text input box on screen (`LineEdit` embedded in the dialogue textbox). Writes `PlayerProfile.player_name`. Validate non-empty; trim; sensible max length. This is a new `DialogueLine` type (`input_line`), not a choice.
3. Escort man: **"C'mon [playerName], it's time to go."** — first use of `[playerName]` substitution (token replaced at render time from `PlayerProfile.player_name`; substitution pass applies to all dialogue text from here on).
4. Single choice (only one option, functions as a themed continue button): **"Get into the carriage"** → consequence `next_dialogue_script_id = intro_bandit_ambush`.

### Scene 2 — `intro_bandit_ambush` (the carriage is stopped)
1. Short scene-setting lines: the carriage halts; shouting outside; bandits on the road. Escort man and the other disciples move to fight.
2. **Branch choice:**
   - **"Hide"** → consequence sets flag `hid_from_bandits`, `next_dialogue_script_id = intro_hide_ending`.
   - **"Fight with the others"** → consequence `start_combat_encounter_id = intro_bandit_tutorial` (existing dialogue→combat handoff). Combat returns → `intro_bandit_victory`.

### Scene 2a — `intro_hide_ending` (bad ending, short)
1. The bandits are defeated by the others while the player hides.
2. Escort man: **"Go home. The path to immortality is not for the faint of heart."**
3. Brief epilogue line (player returns to the village), fade out, **return to Main Menu**.
4. **Save rule:** the active file stays at intro start (`intro_departure` not yet complete), so retrying is fast — the intro up to the choice is ~2 minutes. Do **not** dead-end the save file in an unwinnable state.

### Scene 2b — combat: `intro_bandit_tutorial`
Full spec in **Bandit tutorial fight** checklist above (C3). Summary: 1v1 vs a wounded (low-HP) bandit; player palette locked to Walk / Punch / Block; 3 scripted gated turns teaching Walk → Punch → Block via escort-man interludes; then gating lifts and the fight finishes naturally. Non-lethal — if player HP drops below threshold, the escort man intervenes (interlude + neutral receipt), no defeat screen. Win sets `completed_combat_tutorial` + `fought_the_bandits` and bumps `escort_disciple_affection`.

### Scene 2c — `intro_bandit_victory` (post-fight)
1. Escort man acknowledges the player (tone varies: clean win vs. rescued-by-intervention can share one script with a flag-gated opening line — nice C1 showcase, optional).
2. Junior disciple has a short admiring line (plants the affection thread; portrait can be a color stub).
3. Carriage moves on → `next_dialogue_script_id = intro_sect_arrival`.

### Scene 3 — `intro_sect_arrival` (sect mountain + entrance exam)
1. Arrival lines at the sect gate (background: sect gate or `sect_mountain.png`).
2. **Entrance exam, dialogue-driven (demo):** branching interview with the elders (`sect_elder_wise` / `sect_elder_stern`). Player answers mutate karma / flags (e.g. an honest vs. ambitious answer sets `chose_power_path`-style flags) — reuses the `elder_audience` branching pattern.
3. **Optional interactive beat:** a second 1v1 "combat trial" encounter reusing the arena, full palette. Include only if the pacing needs it; no new mechanics.
4. **Minigames explicitly deferred to Phase D** (see scope decisions above).
5. Closing consequence: set flag `arrived_at_sect`, route to `HOME_HUB`, autosave. Hub hotspots are gated on this flag (C1 reader), so loading a mid-intro file can never skip to the hub.

### Engineering deltas required (land with C3 unless noted)
| Delta | Touch | Size | Status |
|-------|-------|------|--------|
| `player_name` on `PlayerProfile` (+ save payload; `SaveService.migrate`) | `core/data/player_profile.gd`, `core/save_service.gd` | XS | ✅ |
| Name-entry (`is_name_input`) + `LineEdit` in textbox view | `dialogue_line.gd`, `DialogueTextboxView` | S | ✅ |
| `[playerName]` substitution | `DialoguePlaybackController` | XS | ✅ |
| `next_dialogue_script_id` chaining | `dialogue_consequence.gd`, `ConsequenceEngine`, `GameStateManager` | S | ✅ |
| New saves → `intro_departure` | `GameStateManager.select_file` | XS | ✅ |
| Combat→dialogue return | `GameStateManager.finish_combat` | S | ✅ |
| `escort_disciple_affection` | `PlayerProfile`, `ConsequenceEngine` | XS | ✅ |
| Hub gating on `arrived_at_sect` | `home_hub.gd` + file-select routing | — | ✅ |
| Deferred scene navigation | `GameStateManager.navigate_to` | XS | ✅ |

---

## Calendar / Chronicle Checklist (Phase C, step C4)

Reference `system-prompt.md` §6.

**Demo slice:**
- [x] Run calendar state: current date (day), activity/event log, consumed event ids — `core/data/run_calendar_state.gd`
- [x] `MasterChronicle` catalog entry for ≥1 timed story event (place, trigger range, duration, type) — `instructor_night_visit`
- [x] `ActivityResolver`: look-ahead collapse; advance = `days_until_overlap + event.duration` — `core/activity_resolver.gd`
- [x] Training Grounds costs 1 day; Elder remains instant — `HubLocationData.time_cost_days`
- [x] No-event default: flavor + time passed
- [x] Seclusion activity: short cutscene + stat bump; misses ordinary events — `meditation_chamber` hub location, `GameStateManager.commit_seclusion` (skips event scan entirely)
- [x] `PlayerChronicle` global stub: record encounters; show faded future markers on Calendar UI
- [x] Calendar UI: color by type; this-run past vs chronicle future; spoiler = date + place + color + short title — `ui/overlays/chronicle_calendar_panel.gd`

**Explicitly not in demo slice:** village hubs, travel corridors/random tables, recurring-until-triggered events, forced mains, nested multi-hub travel.

---

## Next Session Starter Tasks

Pick up from **Phase C, step C5 (minimal combat resolution playback)** unless directed otherwise:

```
1. Tween unit markers tile-by-tile on Walk/Blink (fixed short per-step duration)
2. Brief visual flash on block windows and interrupt cancels
3. Combat log lines highlighted as their step plays
4. Respect C3 interlude checkpoints between playback steps
```

**Key files to open:**
- `combat/ui/combat_grid_view.gd`, `combat/scenes/combat_arena.gd`
- `combat/core/combat_resolution_engine.gd` (read-only reference — stays logic-instant; playback replays its log/timeline)
- `combat/ui/combat_interlude_controller.gd` (checkpoint pause contract playback must respect)

**Calendar / Chronicle (C4) is now built** — see `core/activity_resolver.gd`, `core/data/master_chronicle*.gd`, `core/data/run_calendar_state.gd`, `core/data/place_catalog.gd`, `ui/overlays/chronicle_calendar_panel.*`. Remaining polish (optional, non-blocking): a second day-costing hub activity to exercise multi-day look-ahead lead-in logging; richer per-entry Calendar UI widgets instead of BBCode color tags.

Then proceed C5 (minimal playback) → C6 (shop) → C7 (settings).

---

## Combat Implementation Checklist (when resuming combat)

Reference `system-prompt.md` §4.11 for algorithm.

- [x] `CombatResolutionEngine` — global event loop from sorted timeline
- [x] Active block windows tracked per combatant at timestamp
- [x] Simultaneous damage groups at same timestamp
- [x] Out-of-range: float walk → aggressive chase / defensive block substitute
- [x] Windup interrupt → cancel + float duration; duration interrupt → partial + float refund (floor)
- [x] Floating points expire end of turn
- [x] Leftover point fill from aggressive/defensive default mode
- [x] Combat UI scene separate from `CombatTurnManager` (data/view split)
- [x] Blink targeting mode with Chebyshev range overlay
- [x] 1v2 encounter preset (`EncounterCatalog` + Outer Slope hub)
- [x] Combat sidebar scroll + Lock In / Retreat pinned (no more clipped Retreat on short windows)
- [ ] NPC beam AI that respects `friendly_fire_policy` (optional polish; NPCs currently Walk/Punch only)

---

## Narrative Implementation Checklist (Section 5)

- [x] `DialogueScript` Resource (lines, speaker, portrait expression key)
- [x] `DialogueChoice` Resource (text, consequence mutations)
- [x] `ConsequenceEngine` — apply deltas to `PlayerProfile`, global flags
- [x] Dialogue backlog/history panel
- [x] Character portrait controller — `PortraitCatalog` + TextureRect swap (partial art; color stub fallback)
- [ ] NPC affection / hostility metrics (deferred; demo ships only `escort_disciple_affection` stub counter via intro)
- [x] Player name entry + `[playerName]` substitution
- [x] Dialogue→dialogue chaining via `next_dialogue_script_id`

---

## Hub Location → Content Map (demo plan)

| Location ID | Intended content | Time cost | Status |
|-------------|------------------|-----------|--------|
| `training_grounds` | Drill dialogue (or `instructor_night_visit` event, day 2–6) | **1 day** | ✅ C4 |
| `elders_pavilion` | Branching narrative choice | **Instant** | ✅ |
| `meditation_chamber` | Seclusion cutscene (`seclusion_meditation`) + stat bump | **3 days** (no event scan) | ✅ C4 |
| `mountain_gate` | 1v1 combat encounter + receipt | 0 (TBD with calendar) | ✅ |
| `outer_slope` | 1v2 combat encounter + receipt | 0 (TBD with calendar) | ✅ |
| `market_path` | Shop overlay | 0 (TBD with calendar) | 🟡 Opens stub overlay |

**Note:** combat tutorial lives in the intro (`intro_bandit_tutorial`). Training Grounds keeps `training_grounds_intro`. Hub unlocks after `arrived_at_sect`; new saves start at `intro_departure`.

**Planned UX:** omnidirectional hub map scroll (pan around / beyond the mountain) when surrounding world content lands — see Phase D → Hub omnidirectional scroll.

---

## GitHub Setup (one-time)

```bash
gh auth login
cd ~/visual-novel-godot-project
gh repo create "Visual-Novel-Godot-Project" --public --source=. --remote=origin --push \
  --description "GDScript manifest and AI context for Cultivation Game (Godot 4)"
```

---

## New Session Context Blurb

Copy into next chat:

> **Project:** CultivationGame1 (Godot 4.6) at `~/Documents/GodotGames/cultivation-game-1`. Manifest at `~/visual-novel-godot-project/system-prompt.md`. Phase A–B combat, File Select, C1–C3b intro spine, deferred scene navigation (`navigate_to`), and **C4 calendar/chronicle demo slice** are built. Next: **C5 minimal combat playback**, then C6 shop, C7 settings. Phase D: hub map scroll, entrance-exam minigames, dating. Read `architecture.md` and `implementation-plan.md` (Phase C).
