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
- [x] Main Menu scene (New Game, Continue disabled, Settings/Shop stubs, Quit)
- [x] Home Hub scene (mountain placeholder, 4 hotspots, persistent overlay)
- [x] Reusable UI overlays (Settings, Shop, Profile, Chronicle)

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
| Save system | `has_save_data()` hardcoded `false` |
| Portraits | Expression keys + color stub; no texture swap yet |

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

### Phase C — Persistence & polish
9. **Save / load** — Multiple slots locked until `has_completed_first_run` (slot state + global `PlayerChronicle` meta).
10. **Static art pass** — Mountain background, portraits, expression swapping.
11. **Shop + inventory** — Item resources, hub/market integration.
12. **Calendar / Chronicle (demo slice)** — See checklist below; full rules in `system-prompt.md` §6.
13. **Settings persistence** — Audio/display prefs.

### Phase D — Post-demo (explicitly deferred)
- Perception / Comprehension intel stats
- Qi cost enforcement, technique mastery, cooldowns
- Player-controlled allies, co-op, PvP
- Dynamic turn pacing algorithm
- Strictly defensive default mode
- Rich activity tables / sub-profession minigames / village hubs / travel random tables
- Forced main events (auto hub + break seclusion)
- Year-scale calendar chrome (large `D` seclusion still supported by engine)
- Combat resolution **visual playback** (tween markers along path / wall-clock timeline scrub)

---

## Calendar / Chronicle Checklist (Phase C, step 12)

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

Pick up from **Phase C, step 9** unless directed otherwise:

```
1. Save / load slots gated by has_completed_first_run
2. Persist PlayerProfile + progression_flags (+ stub PlayerChronicle meta)
3. Enable Main Menu Continue when a slot exists
```

**Key files to open:**
- `core/game_state_manager.gd`
- `scenes/main_menu/main_menu.gd`
- `system-prompt.md` (save-slot notes in Sections 1–3 / architecture)
- `architecture.md` — Pending Infrastructure

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
- [ ] Character portrait controller (texture swap; color stub in place)
- [ ] NPC affection / hostility metrics (deferred)

---

## Hub Location → Content Map (demo plan)

| Location ID | Intended content | Time cost | Status |
|-------------|------------------|-----------|--------|
| `training_grounds` | Tutorial dialogue | **1 day** (when calendar ships) | ✅ content; ⏳ day cost pending §6 |
| `elders_pavilion` | Branching narrative choice | **Instant** | ✅ |
| `mountain_gate` | 1v1 combat encounter + receipt | TBD with calendar | ✅ |
| `outer_slope` | 1v2 combat encounter + receipt | TBD with calendar | ✅ |
| `market_path` | Shop overlay | TBD with calendar | 🟡 Opens stub overlay |

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

> **Project:** CultivationGame1 (Godot 4.6) at `~/Documents/GodotGames/cultivation-game-1`. Manifest at `~/visual-novel-godot-project/system-prompt.md`. Phase A loop and Phase B combat (UI, grid, full resolution, 1v1 + 1v2) are built. Next priority: Phase C step 9 — save/load. Read `architecture.md` and `implementation-plan.md` in the manifest repo for codebase layout and task order.
