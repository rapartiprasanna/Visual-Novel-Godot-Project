# ==============================================================================
# PROJECT MANIFEST & CONTEXT SOURCE OF TRUTH: "CULTIVATION GAME"
# ==============================================================================

# SECTION 1: PROJECT OVERVIEW & SCOPE CONSTRAINTS
- **Project Title**: Cultivation Game (Placeholder)
- **Target Platform**: PC / Desktop Windows
- **Project Scope**: A short, highly focused visual novel "mini-demo" showcase.
- **Timeline**: 3–4 weeks total development time, working a few hours a week.
- **Team**: 1 developer (Extensive Java/OOP experience, beginner to Godot 4).
- **Core Loop**: Main Menu -> Home Hub (Sect Mountain) -> Dialogue/Narrative Branching -> Turn-Based Combat -> Reward & Persistent Meta-Progression.
- **Game Opening**: New runs start with a scripted **intro sequence** before the hub unlocks: leaving home (player name entry) -> carriage -> bandit ambush (hide = short bad ending / fight = combat tutorial vs a wounded bandit) -> sect arrival + dialogue-driven entrance exam -> hub unlock (`arrived_at_sect` flag). Authoring detail: `implementation-plan.md` -> Intro Sequence Spec. Entrance-exam **minigames** and the dating/relationship system are explicitly **post-demo**; the demo ships only an `escort_disciple_affection` stub counter.

## CRITICAL AI SCOPE CONSTRAINTS
- **Strict Scope Control**: Do not suggest features outside this specification unless explicitly requested. No complex multiplayer, procedural generation, or realtime mechanics.
- **Asset Minimalism**: All visual assets are static 2D images or basic UI nodes. Character expressions change via instant texture swapping (no animation rigs).
- **Save System**: Hollow Knight / Super Mario Galaxy 2 style **file select**. The main menu always shows **3 save files**. Empty slot → start a new run on that file. Occupied slot → load that file. Each session is bound to one active file; mid-run saves overwrite that file only (autosave). **Do not** gate slots behind `has_completed_first_run`. **Do not** expose a hub multi-slot save picker. Slot **delete** (clear a file to start fresh on that slot) is deferred past the demo.

# ==============================================================================

# SECTION 2: TECHNICAL STACK & ARCHITECTURE
- **Engine**: Godot 4.x
- **Language**: GDScript 
- **Syntax Standard**: Enforce strictly static-typed GDScript. (e.g., Use `var health: int = 100` and explicit return types `func take_damage(amount: int) -> void:`).
- **Architectural Paradigm**: Merge traditional Object-Oriented Programming (OOP) concepts familiar to a Java developer with Godot's Scene-Node and Component patterns.
- **Data/View Separation**: Keep Game State and Core Data models (custom `Resource` files or data classes) strictly isolated from View/Control nodes.
- **State Management**: Use clean state-pattern patterns or explicit enums for game states rather than scattered boolean flags.
- **Godot 4 Syntax Alignment**: Never use deprecated Godot 3 syntax. Use `await` instead of `yield`, use updated signal syntax (`signal_name.connect(_on_callback)`), and proper UI layout rules.

# ==============================================================================

# SECTION 3: CORE GAME LOOP & SCENE MACHINE
The application architecture relies on a master State Manager that switches between four distinct operational scenes:

1. **Main Menu Scene (MainMenu / File Select / Settings / Shop)**
   - Canvas-based UI screens for game configuration.
   - **File Select** is the primary entry: 3 always-visible save files (empty = new run, occupied = continue that file). No separate New Game / Continue of “most recent.”
   - Shop system reads modular item data resources and modifies player inventory state.

2. **Default/Home Hub Scene (The Sect Mountain)**
   - Visual Base: Static background image representing the Sect Mountain.
   - Dynamic Overlays: Interactive scene buttons populate dynamically over the mountain based on global calendar or narrative triggers.
   - Persistent UI Overlay: Global access to Settings, Player Profile/Stats Panel, and the Calendar / Chronicle UI (see Section 6). Overlay chrome stays fixed when the map moves.
   - Time: Most activities advance the day clock via Flow A (activity → duration → resolve → return). See Section 6 for interrupt rules, nested places, and demo time costs (Training Grounds = 1 day; Elder = instant).
   - **Planned map feel (post-demo / world expansion):** omnidirectional scrolling/panning so the hub reads as a small map, not a single locked postcard. The mountain remains the centerpiece; art and hotspots for forest / road / village surroundings can sit off-frame until the player pans. Even a modest overscan-and-pan pass is desirable before large new zones land. Detail in `implementation-plan.md` → Phase D → Hub omnidirectional scroll.

3. **Dialogue Scene (Visual Novel Mode)**
   - Textbox UI for sequential dialogue processing with type-writer text progression.
   - Support for a basic dialogue history/backlog.
   - Branching narrative framework mapping player selections to concrete variable mutations.
   - **Implementation split (built):** playback logic lives in `DialoguePlaybackController`; reusable UI in `DialogueTextboxView`; full-screen scene is a thin host. Same textbox can embed over combat (interludes).
   - **Player name entry** (intro): a dialogue line type embeds a text input that writes `PlayerProfile.player_name`; all subsequent dialogue text supports `[playerName]` substitution at render time.
   - **Script chaining**: a dialogue consequence may name a next dialogue script, enabling multi-scene sequences (the intro) without returning to the hub.
   - Character Portrait Controller supporting fast switching between static facial expressions. **Demo note:** partial textures ship via `PortraitCatalog` (Instructor + Elder wise/stern); missing expressions fall back to closest available art or color stub.

4. **Combat Scene (Turn-Based Arena)**
   - Self-contained arena module triggered exclusively by targeted choices or specific events.
   - Returns a structured execution receipt (rewards, damage, health states) back to the global manager upon win/loss.

# ==============================================================================

# SECTION 4: UNIQUE COMBAT SYSTEM RULES

## 4.1 Combat Model Overview
- **Paradigm**: Simultaneous planning, sequential resolution on a **global timeline** (Model B).
- **Not traditional turns**: All combatants commit an ordered action queue during a planning phase; actions resolve together when the turn locks in.
- **Demo encounters**: One **1v1** instance and one **1v2** instance. No player-controlled allies in demo.
- **Intel stats**: Perception and Comprehension are **omitted** (demo and near-term future).

## 4.2 Turn Duration & Point-to-Time Mapping
- **Demo turn duration**: Fixed `T = 3.0` seconds per planning/resolution cycle.
- **Per-fighter mapping**: `time_per_point = T / max_points_this_turn`.
- **Example**: Player-A (100 max points) vs NPC-B (10 max points) share the same wall-clock turn. A 5-point punch executes every `0.15s` for A and every `1.5s` for B. A can land ~20 punches in the same window B lands ~2.
- **Future pacing** (`TurnPacingCalculator` stub): Replace fixed `T` with a dynamic algorithm. Candidate formulas:
  - `T = clamp(T_min, T_base + k * sqrt(total_action_count), T_max)`
  - `T = T_base * (max_points_in_encounter / reference_points)`
  - Hybrid: `T = clamp(T_min, T_base * budget_scale * log(1 + action_count), T_max)`

## 4.3 Action Anatomy
Every action has two components:
| Field | Meaning |
|-------|---------|
| `windup_points` | Prepare/channel phase; interruptible; if interrupted, effect may not fire |
| `duration_points` | Active window; partial results if interrupted mid-duration |

| Archetype | Example | Resolution |
|-----------|---------|------------|
| Instant strike | Punch (1w, 0d) | Effect at windup end |
| Duration strike / sustained AoE | Sword flurry / Qi Beam (2w, 2d) | Effect over duration — beam pulses once per duration point for anyone on the line |
| Movement | Walk (0w, 1d per tile) | Position updates over duration; blocked tiles wait/retry within budget |
| Stance | Block (0w, 1d per pt) | Mitigation active during duration |

- **Repeats**: Allowed in demo. Future: per-action `cooldown_turns`.
- **Simultaneous timestamp**: If two executions share the exact same timestamp, both resolve together (mutual damage, fist-clash semantics).

## 4.4 Grid & Space
| Rule | Value |
|------|-------|
| Topology | Square grid, **4-way movement**, **8-way melee adjacency** |
| Canonical unit | **Tiles** (1 tile = 1 Godot unit; ~2 ft is display-only flavor) |
| Demo grid size | **8×8** (tunable) |
| Walk cost | **1 point per tile** (Manhattan path) |
| Occupancy | **One living combatant per tile.** Walk resolves **tile-by-tile**. Cannot enter a tile another unit occupies unless that occupant leaves on the same timestamp (swaps/chains OK). If blocked, the walker **waits on that duration tick** and **retries the same step** on later ticks once the tile clears (within the remaining Walk budget). Unfinished tiles at Walk end refund as **floating**. Two movers claiming the same tile at once both wait and retry. Blink cannot land on an occupied tile. |
| Blink range | **Chebyshev** distance; no line-of-sight in demo |
| Beams / AoE | Measured in **tiles** (Bresenham line for beams) |
| Friendly fire | **On** for all AoE |
| NPC AoE policy | `minimize_friendly_fire` or `ignore_friendly_fire` |

## 4.5 Block & Damage
- **Basic block**: 1 point → `windup: 0`, `duration: 1`. Instant stance, no windup delay.
- **Chained blocks**: Consecutive block actions in queue **merge** into one uninterrupted block window.
- **Mitigation**: Ratio-based formula. Near-perfect reduction (90%+) only when defender vastly outclasses attacker or uses a special block skill with meaningful tradeoffs (long windup, etc.).
- **Penetration**: Attacks may carry a penetration value that reduces block effectiveness. No bonus damage vs unblocked targets.
- **Block vs simultaneous punch**: Block applies if its duration covers the punch execution instant.

**Default substitution** when queued actions cannot execute (out of range):
| Mode | Behavior |
|------|----------|
| **Aggressive** | Replace failed action's point budget with **Walk** (1 pt/tile) to close gap; repeat until in range or out of points |
| **Defensive** | Replace failed action's point budget with **Block** (merged continuous window). Demo: always basic block |

**Floating points**: Refunded from interrupted duration, substitution leftovers, etc. Always **floor** fractional values (`0.9 → 0`). Expire at turn end; never carry over (future: bonus-point techniques).

## 4.6 Range, Targeting & Miss Rules
| Situation | Result |
|-----------|--------|
| Melee, target 8-way adjacent at execution | Auto-target, hit |
| Melee, not adjacent at execution | Action fires → **miss**, points consumed |
| Beam, target off line | Beam fires → **miss** |
| Single-target skill in range | Auto-target if conditions met |
| Target moves during attacker windup | Attacker still resolves at windup end → miss if out of range |

**Out-of-range queue fallback (multi-action)**:
1. Spend floating points on auto-walk (nearest enemy → last targeted → random).
2. If still out of range → apply Aggressive or Defensive default (player toggle; NPC preset).
3. Re-check each subsequent queue entry.

**Auto-target**: Always on for melee and single-target skills within range.

## 4.7 Interrupts
**During windup** (punch, beam windup):
- If `interrupt_value > stability_value` → **full cancel**.
- Attacker idle for remaining windup; action never fires.
- Windup points consumed; duration points → floating (floored).
- Next queued action starts after windup slice ends.

**During duration** (walk, flurry, block):
- **Partial effect** (e.g. half move → half tiles, floored).
- **No idle** for leftover duration — immediately advance queue.
- Unused duration points → floating (floored).

**On interrupt**: No refunds to either side (attacker loses committed timeline; defender's interrupt action consumed normally).

## 4.8 Win / Loss & Resources
| Rule | Detail |
|------|--------|
| **Win** | All enemies HP ≤ 0 |
| **Loss** | Player HP ≤ 0 |
| **Economy (demo)** | Action points only |
| **Future Qi** | Reserve `qi_cost` on every `CombatAction` resource (0 in demo) |
| **Damage RNG** | Small variance in calculation |
| **Future mastery** | Per-technique mastery attribute tightens/improves RNG distribution |

**Damage formula (ratio, sketch)**:
```
base = technique_power × attack_points × strength_factor(attacker)
raw  = base × rng_multiplier(mastery)
mitigation = ratio(defense, strength, block_skill, penetration)
final = raw × (1 - mitigation)
```

## 4.9 Demo Technique Kit
| Action | Windup | Duration | Cost | Notes |
|--------|--------|----------|------|-------|
| Walk | 0 | 1/tile | 1 pt/tile | Default chase; Manhattan |
| Punch | 1 | 0 | 1 pt | Melee adjacent; auto-target |
| Block | 0 | 1/pt | 1 pt | Merge consecutive; default defensive |
| Beam | 2 | 2 | 4 pt | Sustained line AoE; damage **each duration tick** while a unit stays on the beam; penetration; friendly fire |
| Blink | 2 | 0 | 5 pt | Teleport; Chebyshev range **5** |

## 4.10 Combat UI Requirements
**Planning phase**: action queue, skill palette, `allocated / max` points, floating points, selected-action detail panel, undo/clear queue, lock-in button, aggressive/defensive toggle, last-target indicator, range overlays (green valid / red invalid), HP bars.

**Resolution phase**: phase label (Planning → Resolving), combat log, optional damage floaters.

**Blink targeting**: Highlight cursor tile green; show path from current tile; red for out-of-range; click valid tile to queue destination.

**Presentation note (demo)**: Resolution is **logic-instant** — positions snap and the combat log prints results with no tweened travel or wall-clock playback of the timeline. Timed/visual resolution playback (markers sliding tile-to-tile over the turn clock) is deferred polish, not required for demo correctness.

## 4.11 Resolution Algorithm (Implementation Target)
```
PLANNING:
  each fighter submits ActionQueue[] within point budget

BUILD_TIMELINE(fighter):
  t = 0
  merged_queue = collapse_consecutive_blocks(queue)
  for action in merged_queue:
    emit WindupStart, WindupEnd, DurationStart/End/ticks
    t += action.windup + action.duration

RESOLVE (global clock 0..T):
  for each event group at time t:
    apply active blocks to incoming damage events
    resolve simultaneous damage groups
    apply movement ticks / teleport at windup end
    check interrupts on active windups
    on range fail: float → walk substitute → block substitute
    on interrupt: cancel/partial + refund duration → floating (floor)

END:
  clear floating points
  check win/loss
```

## 4.12 Godot Data Schema (CombatAction Resource)
```
CombatAction (Resource)
  id, display_name
  windup_points, duration_points
  point_cost              # windup + duration
  qi_cost                 # 0 in demo; reserved for near-future
  technique_type          # MELEE | RANGED_LINE | AOE | MOVE | TELEPORT | BLOCK
  technique_power, stability, interrupt_value, penetration
  range_tiles, beam_length_tiles
  default_behavior        # CHASE | BLOCK | NONE
  cooldown_turns          # 0 in demo
  friendly_fire_policy    # NPC only: MINIMIZE | IGNORE

CombatantState
  max_points, strength, defense, hp, grid_position
  default_mode            # AGGRESSIVE | DEFENSIVE
  floating_points         # int; floor on refund; expires end of turn
  last_target_id
```

# ==============================================================================

# SECTION 5: DATA-DRIVEN NARRATIVE & CONSEQUENCE ENGINE
- **Storage Strategy**: Dialogue scripts, encounter tables, and character sheets must be handled as discrete data components (Godot `Resource` instances or cleanly structured JSON). Do not hardcode text lines directly inside layout views.
- **Consequence Vectors**: Choices made within the Dialogue Scene must actively mutate:
  - **Player Global Attributes**: (e.g., Cultivation Level, Qi Capacity, Karma).
  - **NPC Status Metrics**: (e.g., Affection Levels, Hostility Factors).
  - **Global Progression Flags**: (Unlocking historical calendar timelines, branching alternative story events).

# ==============================================================================

# SECTION 6: CALENDAR, TIME & CHRONICLE

## 6.1 Role
The calendar is the **time-budget and encounter-resolution layer**. Hub activities consume discrete time. Story content is keyed by **place ∩ time**. Being in the right place during an event’s trigger window fires that encounter instead of the activity’s default outcome. Missing many events is intentional; meta-progression fills future dates with previously discovered encounters on later runs.

## 6.2 Commit Flow (Flow A)
1. Player picks an **activity** from the hub (or a destination hub such as a village).
2. Player picks **duration** when applicable (some activities always default, e.g. explore gardens = 1 day).
3. Resolver runs (see 6.5); activity resolves as flavor beat, short cutscene, minigame, dialogue, and/or combat.
4. Player returns to the home hub; calendar shows the new date and updated log.

**Rule:** Every day must advance via an activity. There is no free “skip time” scrubber. Seclusion / closed-door cultivation is the intentional sink for jumping toward a known future date.

## 6.3 Time Unit & Scale
| Horizon | Rule |
|---------|------|
| **Demo** | Atomic unit = **1 day**. Durations are integer day counts (1 day, N days, 1 week as 7 days, etc.). |
| **Future** | Same day engine; long spans (months / years of cultivation) stay one committed block that plays as a short cutscene + results panel. Early story may still feel daily/weekly; later game stretches timescale without changing the commit model. |

## 6.4 Places (Nested)
Places form a hierarchy (e.g. `sect_mountain` → `west_slope`, `gardens`, `elders_pavilion`). Matching rules:
- An event placed on a **parent** fires when the player is at any **child**.
- An event placed on a **child** does **not** fire at a sibling.
- Travel corridors (e.g. `road_to_village`) are their own place nodes for matching.

**Demo hub time costs:**
- `elders_pavilion` — **instant** (no calendar advance).
- `training_grounds` — **costs 1 day**.

## 6.5 Event Timing Fields
| Field | Meaning |
|-------|---------|
| **Trigger range** | Inclusive date window during which being at a matching place can **start** the event. |
| **Event duration** | Calendar days consumed **once triggered**; fixed regardless of when in the trigger range the player hit it. |
| **Event type** | Used for Calendar color-coding (main, minor, side-story, NPC date, etc.). |

**Consumption:** Triggering an event **consumes it for this run** (no re-fire later in the same window).  
**Recurring until triggered:** Some events respawn on a schedule (e.g. weekly garden visit) until first successful trigger, then stop permanently for that run.  
**Missed non-forced events:** Missed completely for that run (except recurring-until-triggered patterns above).  
**Forced main events (post-demo):** Critical mains (e.g. sect assault) are hard to miss — auto-fire on hub load for that date and/or **break seclusion**. Demo need not implement forced mains.

**Conflict:** Accidental double match at the same place/window → **main** story wins. Content should be authored to avoid this.

## 6.6 Activity Resolution Algorithm
When the player commits activity `A` at place `P` for planned duration `D` starting on current date `T0`:

1. **Seclusion / closed-door cultivation**  
   - Ordinary story events do not share seclusion’s place; seclusion **misses** them by design.  
   - Post-demo: only special forced **main** events may break seclusion (`break if known/marked main overlaps`).  
   - Presentation: always a short (≈5–10s) cutscene + results (stats / technique gains), whether `D` is 1 day or 10 years.

2. **All other activities (look-ahead collapse)**  
   - Scan every day in `[T0, T0+D)`. If any day overlaps an unconsumed event’s **trigger range** at a place matching `P` (nested rules):  
     - **Do not** play generic empty days for the plan.  
     - Collapse immediately into that event (framed as occurring during the activity).  
     - Calendar advance = **`days_until_first_overlap + event.duration`**.  
     - Log lead-in days as the chosen activity (e.g. “Exploring west slope…”) with no separate generic beat, then log the event for its duration days.  
   - Example: explore 2 days from D1; event trigger opens D2; event duration 3 → land on **D5**.

3. **No overlapping event**  
   - **Demo:** small flavor text + time passed (`D` days).  
   - **Later:** per-activity tables (minigames, NPC beats, rewards, side-quest hooks, etc.).

4. **During a triggered event**  
   - Player is locked in that sequence until `event.duration` is applied, then returned to hub.

5. **Travel** (e.g. 3 days to a village)  
   - Short transition, then per segment / corridor resolve in order:  
     1. Planned story event (place + trigger overlap) — **always wins**  
     2. Else weighted random minor table (authored odds; enemies scaled to player)  
     3. Else arrive at destination hub (village scene with fewer activities, same hub pattern)  
   - Random examples: fight beast, find herb, nothing, bandits — then continue/arrive as designed.

## 6.7 Three Data Layers (one UI)
| Layer | Scope | Role |
|-------|-------|------|
| **MasterChronicle** | Authoring / data assets | Internal truth: event id, place, trigger range, duration, type, recurrence, content hooks. Not player-editable. |
| **PlayerChronicle** | **Global** across all runs & save slots | Starts empty; grows with events the player has encountered on any run. Relevant once multiple saves / NG+ style knowledge exists. |
| **Run calendar state** | Per save slot | Current date, activity log for this run, which events are consumed this run. |

**Calendar UI (single player-facing view):**
- **Past (this run):** events/activities already done.
- **Future markers:** from **PlayerChronicle** (encounters known from prior runs), shown on their dates.
- **Color-code** by event type (main, minor, side-story, NPC date, …).
- **PlayerChronicle entries:** same colors, **faded border** (or equivalent) to distinguish from this-run encounters.
- **Spoiler level:** date + location + color + short spoiler title (partial reveal, not full summary).

First run: PlayerChronicle empty → Calendar shows only this-run past. Later runs: future faded markers guide intentional re-encounters or deliberate avoidance.

## 6.8 Demo Slice (implement this; defer the rest)
- Visible current date + activity/event log.
- 1–2 day-costing activities (including Training Grounds = 1 day).
- Elder audience remains instant.
- At least one timed story event with trigger range + duration at one nested place.
- PlayerChronicle stub that records encountered events for a subsequent run’s future markers.
- Defer for now: sub-profession minigames, rich explore tables, village hubs, travel random tables, forced mains, year-scale UI chrome (engine should still accept large `D` on seclusion).

# ==============================================================================

# SECTION 7: AI COMMUNICATION & TUTORING PROTOCOL
- **Deep Explanations**: Do not output raw code dumps. Contextualize explanations deeply, teaching the underlying architectural theory. When beneficial, draw analogies to Java structural patterns to bridge gaps in Godot paradigms.
- **Production-Ready Deliverables**: Write comprehensive code blocks. Avoid truncation shortcuts like `# TODO: implement this` or `# your logic here` within core script files.
- **Diagnostic Safety Check**: Before printing code snippets, mentally verify they strictly comply with Godot 4 API standards to minimize compiler friction.
