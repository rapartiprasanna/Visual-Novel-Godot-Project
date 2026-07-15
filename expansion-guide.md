# Cultivation Game — Content Expansion How-To

Practical recipes for growing the demo. Content is authored in **GDScript catalogs** in the Godot project (`~/Documents/GodotGames/cultivation-game-1`), not in separate story JSON.

**Paths below are under the Godot project root** unless noted. Design intent lives in this manifest repo (`system-prompt.md`, `architecture.md`, `implementation-plan.md`).

> **Update this guide when unfinished systems land.** Sections marked *Not built yet* describe stubs or missing APIs. After those features ship, replace those notes with real authoring steps.

---

## Mental model

| You want… | You edit… |
|-----------|-----------|
| Dialogue / choices / rewards | `core/data/dialogue_catalog.gd` |
| Speakers / portraits | `core/data/portrait_catalog.gd` (+ art under `assets/art/portraits/`) |
| Hub hotspots | `core/data/hub_location_catalog.gd` |
| Combat kits / enemies / interludes | `combat/data/encounter_catalog.gd` |
| Technique definitions | `combat/data/action_library.gd` (runtime; keep `.tres` in sync if you use them) |
| Player stats shape | `core/data/player_profile.gd` + `core/consequence_engine.gd` + save payload |
| Scene routing | Almost always via consequences / hub types — not by calling `change_scene_to_file` yourself |

**Rule:** never call `SceneTree.change_scene_to_file` from dialogue/combat finish handlers. Always go through `GameStateManager.navigate_to` (deferred).

---

## 1. Add a new character (speaker + portrait)

### Minimum (color stub only)

1. Use a stable `speaker_name` string on dialogue lines (e.g. `"Senior Escort"`).
2. If `PortraitCatalog` has no match, the textbox uses a **color stub** — no code required.
3. Optional: add a stub color in `ui/dialogue/dialogue_textbox_view.gd` (`_apply_portrait`) so they look distinct.

### With art

1. Drop a texture under `assets/art/portraits/` (Godot will import it).
2. In `core/data/portrait_catalog.gd`:
   - `preload` the texture.
   - Add a `match` arm for your `speaker_name`.
   - Optionally branch on `expression_key` (`"neutral"`, `"stern"`, `"wise"`, …).
3. Use that exact `speaker_name` + `expression_key` on `DialogueLine`s in `DialogueCatalog`.

**Demo speakers with art:** `"Instructor"`, `"Sect Elder"` (wise/stern).  
**Demo stubs only:** `"Senior Escort"`, `"Junior Disciple"`, `"Narrator"`.

Dialogue lines never store texture paths — only speaker + expression.

---

## 2. Player stats

### What exists today (`PlayerProfile`)

| Field | Typical use |
|-------|-------------|
| `player_name` / `display_name` | Intro name entry; `[playerName]` substitution |
| `cultivation_level` | Combat power scaling (see below) |
| `current_qi` / `qi_capacity` | Dialogue rewards; **not** enforced as combat technique costs yet |
| `karma` | Dialogue + combat win reward |
| `escort_disciple_affection` | Stub counter from intro tutorial win |

### Manually (editor / debug)

- Change defaults in `PlayerProfile.create_default()`, or temporarily tweak after load in a script / debugger.
- Profile UI: hub overlay `ui/overlays/player_profile_panel.*` (shows main stats; affection not shown yet).

### Through gameplay (preferred)

Attach a `DialogueConsequence` on a **line shown** or **choice selected** (via catalog helpers):

| Field | Effect |
|-------|--------|
| `karma_delta` | Add to karma |
| `qi_delta` | Clamp into `[0, qi_capacity]` |
| `cultivation_level_delta` | Level floored at 1 |
| `escort_disciple_affection_delta` | Affection floored at 0 |
| `set_flags` / `clear_flags` | Progression flags |

Helpers in `DialogueCatalog`: `_consequence(...)`, `_consequence_affection(...)`, `_consequence_with_dialogue(...)`.

**Combat win karma:** `CombatReceipt.karma_reward` applied in `GameStateManager.finish_combat` (demo wins use a fixed reward in the arena).

### How combat reads the profile

`EncounterCatalog.build_player_from_profile(profile)`:

- `max_points = 9 + cultivation_level`
- `strength = 10 + cultivation_level`
- `defense = 10`, HP = 100 (fixed for now)

Karma / qi / affection do **not** affect combat stats today. To change that mapping, edit `build_player_from_profile` only.

### Adding a new persisted stat

1. Add `@export` on `PlayerProfile` (+ default in `create_default()`).
2. Apply it in `ConsequenceEngine.apply` (new delta field on `DialogueConsequence` if needed).
3. Persist via `SaveService` / `migrate` (bump save version when payload shape changes).
4. Show it in the profile panel if players should see it.

> **Not built yet — update this guide later:** inventory items, qi cost enforcement, technique mastery, full NPC relationship metrics beyond the affection stub.

---

## 3. Dialogue scripts, choices, and transitions

### Author a new script

1. Open `core/data/dialogue_catalog.gd`.
2. Add `_build_your_script() -> DialogueScript` using helpers (`_line`, `_line_with_choices`, `_choice`, …).
3. Register the trigger id in `get_script_by_trigger`’s `match`.
4. Wire entry (hub hotspot, `next_dialogue_script_id` chain, or post-combat pending dialogue).

**Trigger id ≈ script `id`** in demo content.

### Useful line/choice fields

| Field | Use |
|-------|-----|
| `next_line_id` | Linear continue; empty ends the script |
| `choices` | Branch; unavailable choices are **hidden** |
| `consequence` | Applied when the line is shown (lines) or when chosen |
| `required_flag` / `blocked_by_flag` | Gate availability |
| `unavailable_next_line_id` | Line fallback when a gated line is skipped (see `elder_audience`) |
| `is_name_input` | Shows name `LineEdit`; blocks advance until confirm |

### Chaining & combat handoff (no manual scene code)

| Consequence field | Result after dialogue finishes |
|-------------------|-------------------------------|
| `next_dialogue_script_id` | Start that script |
| `start_combat_encounter_id` | Start that encounter (then optional next script) |
| both via `_combat_then_dialogue` | Fight, then victory script on win |
| `return_to_main_menu` | Bad-end path (intro hide) |

Helpers: `_dialogue_chain`, `_combat_then_dialogue`, `_return_to_menu_consequence`.

### `[playerName]`

Use the literal token `[playerName]` in line text. Playback substitutes from `player_name` (fallback `display_name` / `"Disciple"`).

### Flag-gated opening (copy this pattern)

`elder_audience`: default opening uses `blocked_by_flag = "chose_power_path"` and `unavailable_next_line_id` pointing at a `required_flag = "chose_power_path"` variant.

### Mid-combat dialogue

Do **not** navigate to the dialogue scene (that destroys the arena). Add a `CombatInterludeTrigger` on the encounter and a script in `DialogueCatalog`. See §6.

---

## 4. Wire hub locations (scenes players can visit)

1. Add a row in `HubLocationCatalog.get_demo_locations()` via `_create(...)`.
2. Set:
   - `anchor_position` — UV (0–1) on `assets/art/hub/sect_mountain.png`
   - `content_type` — `DIALOGUE` | `COMBAT` | `SHOP`
   - `narrative_trigger_id` — dialogue trigger id **or** encounter id
   - optional `required_unlock_flag` — locked until that progression flag is set

| Type | Runtime |
|------|---------|
| `DIALOGUE` | `GameStateManager.start_dialogue(trigger_id)` |
| `COMBAT` | `GameStateManager.start_combat(encounter_id)` |
| `SHOP` | Opens shop overlay (**stub stock/buy flow**) |

**Example gated combat:** Outer Slope unlocks after flag `trained_morning_drills` (set by Training Grounds dialogue).

**Hub vs intro:** hotspots only matter after flag `arrived_at_sect`. New saves start at `intro_departure`, not the hub.

> **Not built yet — update later:** calendar time costs per location (Training Grounds = 1 day designed; Elder = instant), hub map scroll, real Market Path shopping.

---

## 5. Techniques / actions and “learning” them

### Create or tweak a technique

1. Prefer editing **`combat/data/action_library.gd`** (`create_*` factories) — this is what combat loads at runtime.
2. Mirror changes in `combat/actions/*.tres` if you keep those as editor docs (runtime does **not** load `.tres` today).
3. Register in `ActionLibrary.get_starter_kit()` / constants if you add a new id.
4. Key fields: `id`, `windup_points`, `duration_points`, `technique_type`, `technique_power`, `range_tiles`, `block_skill_factor`, etc. (`combat/data/combat_action.gd`).

### Give the player access in a fight

Encounter-level kit (not ownership yet):

```gdscript
# EncounterCatalog.get_player_action_ids(encounter_id)
# intro_bandit_tutorial → walk, punch, block
# everything else → full starter kit (walk, punch, block, beam, blink)
```

Add a `match` arm for restricted or expanded kits per encounter.

### Temporary “teach this now” (tutorial gating)

Use interludes on `intro_bandit_tutorial`-style fights:

- `allowed_palette_action_ids` — sidebar only shows these while the gate is active
- `required_action_ids` — Lock In blocked until those actions are queued

This is **combat-session gating**, not permanent unlock.

### Learn from story choices — status

> **Not built yet.** There is no `DialogueConsequence` field that adds a technique to a lasting player kit. Flavor lines / flags can *claim* the player learned something, but the palette still comes from `get_player_action_ids`.

**When you implement learning, update this section with:** profile field or known-technique list → save `migrate` → consequence grant → how `get_player_action_ids` / arena reads the kit.

Until then, approximate “learning” by:

1. Setting a flag from a choice (`set_flags`).
2. In `get_player_action_ids`, including an action id only when `GameStateManager.has_progression_flag(...)` (manual check you add).

That pattern is a reasonable interim; formalize it once ownership exists.

---

## 6. Combat encounters and story-in-combat

### New encounter checklist

1. `EncounterCatalog.build_enemies(encounter_id)` — spawn `CombatantState`(s).
2. `get_player_action_ids` — player kit for that fight.
3. Optional `get_interludes` — mid-fight scripts + gating.
4. Launch via:
   - Hub `COMBAT` hotspot, or
   - Dialogue `_combat_then_dialogue(encounter_id, next_script_id)`.

Copy `_rival_disciple` / `_wounded_bandit` for enemy templates.

### Interlude story beats

1. Author dialogue scripts in `DialogueCatalog`.
2. Build triggers with `_interlude` / `_hp_below_interlude` helpers in `EncounterCatalog`.
3. Trigger types include: `COMBAT_START`, `PLANNING_START` (turn N), `AFTER_RESOLVE`, `HP_BELOW`, `COMBATANT_DEFEATED`, `PLAYER_ACTION_QUEUED`, `COMBAT_END`.

Interludes pause combat at checkpoints; resolution engine stays dialogue-agnostic.

---

## 7. Story events and progression flags

### Flags (the working “event” system today)

Flags are a string → bool map on `GameStateManager`.

| API | Use |
|-----|-----|
| `set_progression_flag(name, value=true)` | Via consequences (`set_flags`) |
| `has_progression_flag(name)` | Hub unlocks, dialogue gates, future readers |
| `clear_flags` on consequences | Remove flags |

**Demo flags (examples):** `trained_morning_drills`, `chose_power_path`, `arrived_at_sect`, `fought_the_bandits`, `completed_combat_tutorial`, …

**Readers already wired:**

- Hub hotspot unlock (`required_unlock_flag`)
- Dialogue line/choice availability
- Intro → hub routing (`arrived_at_sect`)

**New “story event” recipe (flag-based):**

1. Set a flag from dialogue/combat consequence.
2. Gate later dialogue / hub / encounter kit on that flag.
3. Prefer unique, past-tense names (`trained_morning_drills`, not `drill`).

### Timed calendar / MasterChronicle events

> **Not built yet (Phase C4).** Chronicle UI is a placeholder; `PlayerChronicle` only records encountered ids for a future faded calendar. No day clock, no timed story unlocks, no ActivityResolver.

When C4 ships, update this section with: authoring a `MasterChronicle` entry, activity day costs, and how events appear on the Calendar UI.

### Forced mains / village travel / dating

> **Not built yet (Phase D).** Do not author content that depends on them.

---

## 8. Quick recipes

### “New hub conversation that bumps cultivation and unlocks a fight”

1. `_build_*` script in `DialogueCatalog` + register trigger.
2. Choice consequence: `cultivation_level_delta = 1`, `set_flags = ["my_unlock"]`.
3. Hub location: `COMBAT` encounter with `required_unlock_flag = "my_unlock"`.

### “Dialogue → fight → dialogue”

```gdscript
_combat_then_dialogue("my_encounter", "my_victory_script")
```

Ensure win path sets any flags; `finish_combat` chains to the pending victory script on win.

### “Same speaker, two openings by prior choice”

Mirror `elder_audience`: default line `blocked_by_flag` + `unavailable_next_line_id` → flagged alternate line `required_flag`.

### “Brand-new technique only after a story choice (interim)”

1. Add action in `ActionLibrary`.
2. Dialogue choice sets flag `learned_my_tech`.
3. In `get_player_action_ids`, append the action id when that flag is set.

Replace with real ownership when implemented.

---

## 9. File cheat sheet

| Area | Primary files |
|------|----------------|
| Dialogue content | `core/data/dialogue_catalog.gd`, `dialogue_line.gd`, `dialogue_choice.gd`, `dialogue_consequence.gd` |
| Playback (don’t put story here) | `core/dialogue/dialogue_playback_controller.gd`, `ui/dialogue/dialogue_textbox_view.*` |
| Portraits | `core/data/portrait_catalog.gd`, `assets/art/portraits/` |
| Hub | `core/data/hub_location_catalog.gd`, `scenes/home_hub/home_hub.gd` |
| Profile / saves | `core/data/player_profile.gd`, `core/consequence_engine.gd`, `core/save_service.gd` |
| Encounters | `combat/data/encounter_catalog.gd` |
| Actions | `combat/data/action_library.gd`, `combat/data/combat_action.gd` |
| Interludes | `combat/data/combat_interlude_trigger.gd`, `combat/ui/combat_interlude_controller.gd` |
| Navigation | `core/game_state_manager.gd` |

---

## 10. Checklist: after unfinished features land

Update this document when any of these ship:

| Feature | Plan step | What to document |
|---------|-----------|------------------|
| Calendar / timed story events | C4 | MasterChronicle authoring, day costs, Calendar markers |
| Combat resolution playback | C5 | Whether interludes pause during tween playback |
| Shop + inventory | C6 | Items, buy flow, Market Path stock |
| Settings persistence | C7 | — (rarely content-facing) |
| Technique learning / mastery / qi cost | Phase D | Grant/learn from choices, kit persistence |
| NPC relationships / dating | Phase D | Affection beyond stub → romance beats |
| Hub map scroll + surroundings | Phase D | Anchors on scrollable map, new zone hotspots |
| Catalogs as `.tres` / data files | Phase D | Migration from GDScript factories |

Also refresh `architecture.md` / `implementation-plan.md` when the “Completed” vs “stub” lists change — this guide should stay the **operator** doc, those stay the **roadmap** docs.
