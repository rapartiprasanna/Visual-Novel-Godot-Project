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

---

## In Progress / Stub 🟡

| Item | Gap |
|------|-----|
| Combat arena | ✅ Planning HUD + grid + full resolve + 1v1/1v2 encounters |
| Combat resolution | ✅ Engine live; visual playback still deferred (Phase D) |
| NPC AoE policy | Fields set; beam AI not queued yet (no NPC beam in demo kit AI) |
| Settings / Shop | Placeholder text only |
| Chronicle Calendar | Static placeholder; design locked in `system-prompt.md` §6; no clock/data yet |
| **Save system** | ✅ File Select: 3 always-visible files on main menu; active-file autosave; no hub picker; delete deferred |
| Hub art | ✅ `assets/art/hub/sect_mountain.png` full-bleed; hotspots anchored to landmarks |
| Portraits | 🟡 Partial: `instructor1`, `sect_elder_wise`, `sect_elder_stern` via `PortraitCatalog`; missing expressions fall back |
| **Intro sequence** | ❌ Not started — designed in **Intro Sequence Spec** below; consumes C2 + C3; ships as step C3b |

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

#### C1. Consequence readers — make choices matter (small, do first)
Flags and profile stats are written but nothing reads them. Close the loop with three readers:
- [ ] **Profile → combat:** `EncounterCatalog.build_player_from_profile(profile)` replaces the hardcoded `build_player()`. Simple mapping (e.g. `strength = 10 + cultivation_level`, `max_points = 9 + cultivation_level`, karma untouched). Call site: `combat/scenes/combat_arena.gd` `_setup_encounter()`.
- [ ] **Flags → hub:** `HubLocationCatalog` honors `is_unlocked` from `GameStateManager.has_progression_flag` (e.g. `outer_slope` locked until `trained_morning_drills`). Touch: `core/data/hub_location_catalog.gd`, `scenes/home_hub/home_hub.gd`.
- [ ] **Flags → dialogue:** at least one branch in `elder_audience` that checks a prior flag (e.g. `chose_power_path` changes the opening line). Requires a small `required_flag` / `blocked_by_flag` field on `DialogueLine` or `DialogueChoice` + filter in the dialogue view. Touch: `core/data/dialogue_line.gd` or `dialogue_choice.gd`, `core/data/dialogue_catalog.gd`, `scenes/dialogue/dialogue_scene.gd`.

**Key files:** `combat/data/encounter_catalog.gd`, `core/data/hub_location_catalog.gd`, `core/data/dialogue_catalog.gd`.

#### C2. Dialogue playback extraction (prerequisite for C3)
Today `scenes/dialogue/dialogue_scene.gd` fuses playback logic with the full-screen scene and hard-calls `GameStateManager.finish_dialogue()`. Extract so the same textbox can be embedded anywhere (combat overlay, future cutscenes):
- [ ] `DialoguePlaybackController` (`core/dialogue/dialogue_playback_controller.gd`, RefCounted or Node): owns script/line cursor, typewriter counters, choice handling, consequence application. Emits signals only: `line_shown(line)`, `choices_shown(choices)`, `typing_finished`, `playback_finished(script_id)`. **Never** touches `GameStateManager` navigation.
- [ ] `DialogueTextboxView` (`ui/dialogue/dialogue_textbox_view.tscn` + `.gd`): reusable Control — portrait (via `PortraitCatalog` + color fallback), speaker label, RichTextLabel body, continue hint, choice container, optional backlog. Binds to a controller instance.
- [ ] Rewrite `scenes/dialogue/dialogue_scene.gd` as a thin host: instantiate view + controller, feed `GameStateManager.get_active_dialogue_script()`, on `playback_finished` call `GameStateManager.finish_dialogue()`. Full-scene behavior unchanged.
- [ ] Regression check: `training_grounds_intro` and `elder_audience` play identically (typewriter, choices, backlog, consequences, dialogue→combat handoff).

**Key files:** `scenes/dialogue/dialogue_scene.gd` (shrinks), new `core/dialogue/` + `ui/dialogue/`.

#### C3. Combat interludes + tutorial fight (uses C2)
Mid-combat dialogue system, designed to be reused for story beats / cutscenes inside any future fight. See **Combat Interlude & Tutorial Checklist** below for full detail.

> **Re-scoped (intro decision):** the tutorial encounter is now the **bandit ambush fight** inside the intro sequence (see **Intro Sequence Spec**), not an optional Training Grounds spar. Same system, same gating fields — different skin and entry point. `training_spar_tutorial` naming below is superseded by `intro_bandit_tutorial`.

- [ ] `CombatInterludeTrigger` resource + `InterludeTriggerType` enum
- [ ] `CombatInterludeController` node in combat arena + `CombatDialogueOverlay` (embeds `DialogueTextboxView`)
- [ ] Checkpoint signals on `CombatTurnManager` (no changes inside `CombatResolutionEngine`)
- [ ] Tutorial encounter `intro_bandit_tutorial` (wounded bandit, low HP) launched from intro dialogue via existing `start_combat_encounter_id`
- [ ] Lock-in gating for tutorial objectives (queue Walk → Punch → Block across scripted turns)
- [ ] Intro plumbing that lands with C3: player name entry (`LineEdit` step in dialogue), `player_name` on `PlayerProfile`, `[playerName]` text substitution, dialogue→dialogue chaining (`next_dialogue_script_id` consequence), intro→hub unlock flag

#### C3b. Intro content pass (uses C1 + C2 + C3)
Authoring pass for the opening sequence — mostly writing, not engineering. Full beat-by-beat detail in **Intro Sequence Spec** below.
- [ ] Scripts: `intro_departure`, `intro_bandit_ambush`, `intro_hide_ending`, `intro_bandit_victory`, `intro_sect_arrival` (+ entrance-exam dialogue)
- [ ] New saves route to `intro_departure` instead of `HOME_HUB` (one-line change in `GameStateManager.select_file` once chaining exists)
- [ ] Hub gated behind `arrived_at_sect` flag (C1 reader)
- [ ] Art: escort man portrait, bandit portrait, carriage/road background, (optional) sect gate background
- [ ] Affection stub: `escort_disciple_affection` counter or flag on `PlayerProfile` (no relationship system — Phase D)

#### C4. Calendar / Chronicle demo slice (was step 12 — **promoted**)
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
- [ ] `SaveService.migrate(data: Dictionary) -> Dictionary` keyed on `version`; identity for v1. All future payload additions (inventory, calendar state) go through it.

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

---

## Combat Interlude & Tutorial Checklist (Phase C, step C3)

Goal: pause combat at defined checkpoints, play dialogue over the arena, resume. The intro's bandit tutorial fight is the first consumer; the same system later drives story dialogue / cutscenes mid-fight.

### Design principles
- **Combat logic never knows about dialogue.** `CombatResolutionEngine` stays untouched. Pauses happen only at checkpoints that `CombatTurnManager` / the arena scene already control (phase boundaries, and later, playback step boundaries in C5).
- **Overlay, not scene change.** Combat state lives in the arena scene tree; navigating to the dialogue scene would destroy it. The interlude renders `DialogueTextboxView` (from C2) in a CanvasLayer above the arena.
- **Data-driven.** Interludes are resources attached to an encounter, not code in the arena script. A story fight and a tutorial differ only in trigger data + script content.

### Data layer
- [ ] `InterludeTriggerType` enum (`combat/enums/combat_enums.gd` or new file):
      `COMBAT_START`, `PLANNING_START` (turn N), `AFTER_RESOLVE` (turn N), `HP_BELOW` (combatant %, checked after resolve), `COMBATANT_DEFEATED`, `PLAYER_ACTION_QUEUED` (action id, for tutorial nudges), `COMBAT_END`
- [ ] `CombatInterludeTrigger` resource (`combat/data/combat_interlude_trigger.gd`):
      `trigger_type`, `turn_number`, `combatant_id`, `hp_ratio_threshold`, `action_id`, `dialogue_script_id`, `once: bool = true`, `consumed` (runtime), plus tutorial-gating fields below
- [ ] `EncounterCatalog` gains per-encounter interlude list: `get_interludes(encounter_id) -> Array[CombatInterludeTrigger]` (empty for existing encounters)
- [ ] Interlude dialogue scripts live in `DialogueCatalog` like any other script (same `DialogueScript` resources — reuse, no new format)

### Runtime layer
- [ ] `CombatTurnManager` additions (signals only, no behavior change):
      `turn_number: int`, signals `planning_started(turn_number)`, `resolution_finished(turn_number)`, `combatant_defeated(combatant_id)`, and `player_action_queued(action_id)` emitted from `queue_action_for_player`
- [ ] `CombatInterludeController` (node in `combat_arena.tscn`, script `combat/ui/combat_interlude_controller.gd`):
      - Loads triggers for the active encounter; subscribes to turn manager signals
      - On match: sets arena input to blocked (planning UI disabled, grid clicks ignored), shows overlay, plays script via `DialoguePlaybackController`
      - On `playback_finished`: hides overlay, re-enables input, marks trigger consumed
      - Consequences inside interlude scripts flow through `ConsequenceEngine` as usual (works today); combat-state mutations from dialogue are **out of scope** for demo
- [ ] `CombatDialogueOverlay` (`combat/ui/combat_dialogue_overlay.tscn`): CanvasLayer + dim ColorRect + embedded `DialogueTextboxView`; no backlog needed in demo
- [ ] Pause semantics: demo combat is logic-instant, so "pause" = deferring the next phase transition until the overlay closes. `COMBAT_END` triggers defer the `finish_combat` receipt return. When C5 playback lands, the controller also gates the next playback step.

### Bandit tutorial fight (first consumer — re-skinned from training spar per intro decision)
- [ ] Encounter `intro_bandit_tutorial` in `EncounterCatalog`: 1v1 vs a **wounded bandit** (reduced HP, low damage), player from profile (C1). Player palette = Walk / Punch / Block only; bandit AI = Walk / Punch / Block (existing AI kit). Non-lethal for the player: on player HP ≤ threshold the escort man intervenes via interlude + scripted neutral receipt, never a defeat screen
- [ ] Tutorial gating fields on `CombatInterludeTrigger` (used only by tutorial):
      `required_action_ids: Array[String]` (lock-in disabled until these are queued this turn), `allowed_palette_action_ids` (sidebar shows only these)
- [ ] `CombatPlanningSidebar` support: palette filtering + lock-in disable with hint text (reads gating state from the interlude controller; sidebar stays dumb)
- [ ] Scripted flow (3 short turns), tutorial lines voiced by the **escort man** shouting from the sidelines:
      1. `COMBAT_START` interlude — explains points + Walk; palette = Walk only; lock-in requires a queued Walk
      2. Turn 2 `PLANNING_START` — explains windup + Punch; palette = Walk/Punch; requires Punch queued
      3. Turn 3 `PLANNING_START` — explains Block merging + defensive default; palette = Walk/Punch/Block; requires Block queued
      4. `AFTER_RESOLVE` turn 3 — gating lifts; fight runs to natural finish (bandit is wounded, so 1–2 more turns)
      5. `COMBAT_END` — wrap-up interlude, receipt (`player_won = true`, small reward, sets flags `completed_combat_tutorial` + `fought_the_bandits`, bumps `escort_disciple_affection`)
- [ ] Entry point: "Fight with the others" choice in `intro_bandit_ambush` using existing `DialogueConsequence.start_combat_encounter_id` (see Intro Sequence Spec)
- [ ] Beam/Blink deliberately **not** tutorialized — discovery in real encounters; tooltip text on palette buttons is enough

### Reuse path (post-demo, no rework expected)
- Story beat mid-boss-fight = `HP_BELOW` trigger + normal `DialogueScript` (works with zero new code)
- Cutscenes with camera/animation = new overlay variant behind the same controller checkpoints
- Mid-playback stingers = same triggers gated on C5 playback step boundaries

### Combat regression seed tests (land alongside C3, guards the signal additions)
- [ ] Headless test script(s) under `tests/combat/`: block-chain merge, windup interrupt cancel, walk collision retry, simultaneous damage group — pinned RNG seed

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
| Delta | Touch | Size |
|-------|-------|------|
| `player_name` on `PlayerProfile` (+ save payload; goes through `SaveService.migrate` when C-infra lands) | `core/data/player_profile.gd`, `core/save_service.gd` | XS |
| Name-entry dialogue line type (`input_line`) + `LineEdit` in textbox view | `core/data/dialogue_line.gd`, dialogue view (post-C2: `DialogueTextboxView`) | S |
| `[playerName]` substitution pass at line render | dialogue playback (post-C2: `DialoguePlaybackController`) | XS |
| `next_dialogue_script_id` on `DialogueConsequence` (dialogue→dialogue chaining) | `core/data/dialogue_consequence.gd`, `ConsequenceEngine`, `GameStateManager` | S |
| New saves route to `intro_departure` instead of `HOME_HUB` | `GameStateManager.select_file` | XS |
| Combat→dialogue return routing (receipt → `intro_bandit_victory` instead of hub) | `GameStateManager.finish_combat` (respect pending next-dialogue when set) | S |
| `escort_disciple_affection` counter on `PlayerProfile` | `core/data/player_profile.gd`, `ConsequenceEngine` delta support | XS |
| Hub gating on `arrived_at_sect` | C1 flag reader (already planned) | — |

---

## Calendar / Chronicle Checklist (Phase C, step C4)

Reference `system-prompt.md` §6.

**Demo slice:**
- [ ] Run calendar state: current date (day), activity/event log, consumed event ids
- [ ] `MasterChronicle` catalog entry for ≥1 timed story event (place, trigger range, duration, type)
- [ ] `ActivityResolver`: look-ahead collapse; advance = `days_until_overlap + event.duration`
- [ ] Training Grounds costs 1 day; Elder remains instant
- [ ] No-event default: flavor + time passed
- [ ] Seclusion activity: short cutscene + stat bump; misses ordinary events
- [ ] `PlayerChronicle` global stub: record encounters; show faded future markers on Calendar UI
- [ ] Calendar UI: color by type; this-run past vs chronicle future; spoiler = date + place + color + short title

**Explicitly not in demo slice:** village hubs, travel corridors/random tables, recurring-until-triggered events, forced mains, nested multi-hub travel.

---

## Next Session Starter Tasks

Pick up from **Phase C, step C1 (consequence readers)** unless directed otherwise:

```
1. EncounterCatalog.build_player_from_profile — combat player stats from PlayerProfile
2. Hub location gating via progression flags (is_unlocked reader)
3. One flag-aware branch in elder_audience dialogue
```

**Key files to open:**
- `combat/data/encounter_catalog.gd`
- `core/data/hub_location_catalog.gd` / `scenes/home_hub/home_hub.gd`
- `core/data/dialogue_catalog.gd` / `core/data/dialogue_line.gd`

Then proceed C2 (dialogue playback extraction) → C3 (combat interludes + bandit tutorial + intro plumbing) → C3b (intro content pass, see Intro Sequence Spec) → C4 (calendar slice) → C5 (minimal playback) → C6 (shop) → C7 (settings).

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
- [ ] NPC beam AI that respects `friendly_fire_policy` (optional polish; NPCs currently Walk/Punch only)

---

## Narrative Implementation Checklist (Section 5)

- [x] `DialogueScript` Resource (lines, speaker, portrait expression key)
- [x] `DialogueChoice` Resource (text, consequence mutations)
- [x] `ConsequenceEngine` — apply deltas to `PlayerProfile`, global flags
- [x] Dialogue backlog/history panel
- [x] Character portrait controller — `PortraitCatalog` + TextureRect swap (partial art; color stub fallback)
- [ ] NPC affection / hostility metrics (deferred; demo ships only `escort_disciple_affection` stub counter via intro)
- [ ] Player name entry + `[playerName]` substitution (intro, lands with C3)
- [ ] Dialogue→dialogue chaining via `next_dialogue_script_id` (intro, lands with C3)

---

## Hub Location → Content Map (demo plan)

| Location ID | Intended content | Time cost | Status |
|-------------|------------------|-----------|--------|
| `training_grounds` | Tutorial dialogue | **1 day** (when calendar ships) | ✅ content; ⏳ day cost pending §6 |
| `elders_pavilion` | Branching narrative choice | **Instant** | ✅ |
| `mountain_gate` | 1v1 combat encounter + receipt | TBD with calendar | ✅ |
| `outer_slope` | 1v2 combat encounter + receipt | TBD with calendar | ✅ |
| `market_path` | Shop overlay | TBD with calendar | 🟡 Opens stub overlay |

**Note:** the combat tutorial is **no longer** a Training Grounds activity — it lives in the intro sequence (`intro_bandit_tutorial`, see Intro Sequence Spec). Training Grounds keeps its existing `training_grounds_intro` dialogue. The hub itself is gated behind the `arrived_at_sect` flag; new saves start at `intro_departure`.

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

> **Project:** CultivationGame1 (Godot 4.6) at `~/Documents/GodotGames/cultivation-game-1`. Manifest at `~/visual-novel-godot-project/system-prompt.md`. Phase A–B combat, File Select save (9b), hub mountain art, and partial portraits (Instructor + Elder wise/stern) are built. Phase C was **reordered** after an architecture review: next priority is **C1 — consequence readers** (profile→combat stats, flag→hub gating, flag→dialogue branch), then C2 dialogue playback extraction, C3 combat interludes + **bandit tutorial fight** + intro plumbing (name entry, `[playerName]` substitution, dialogue chaining), C3b **intro content pass** (departure → bandit ambush → tutorial → sect arrival; new saves start here, hub gated on `arrived_at_sect`), C4 calendar slice, C5 minimal resolution playback, C6 shop, C7 settings. Entrance-exam minigames and the dating system are Phase D. Read `architecture.md` and `implementation-plan.md` (Phase C section + **Intro Sequence Spec** + Combat Interlude & Tutorial Checklist).
