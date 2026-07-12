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

## CRITICAL AI SCOPE CONSTRAINTS
- **Strict Scope Control**: Do not suggest features outside this specification unless explicitly requested. No complex multiplayer, procedural generation, or realtime mechanics.
- **Asset Minimalism**: All visual assets are static 2D images or basic UI nodes. Character expressions change via instant texture swapping (no animation rigs).
- **Save System**: Support multiple user save slots. These slots should remain locked until a global system flag (`has_completed_first_run == true`) is met.

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

1. **Main Menu Scene (MainMenu / Settings / Shop)**
   - Canvas-based UI screens for game configuration.
   - Shop system reads modular item data resources and modifies player inventory state.

2. **Default/Home Hub Scene (The Sect Mountain)**
   - Visual Base: Static background image representing the Sect Mountain.
   - Dynamic Overlays: Interactive scene buttons populate dynamically over the mountain based on global calendar or narrative triggers.
   - Persistent UI Overlay: Global access to Settings, Player Profile/Stats Panel, and the "Chronicle Calendar" (which tracks events across current and historical playthroughs).

3. **Dialogue Scene (Visual Novel Mode)**
   - Textbox UI for sequential dialogue processing with type-writer text progression.
   - Support for a basic dialogue history/backlog.
   - Branching narrative framework mapping player selections to concrete variable mutations.
   - Character Portrait Controller supporting fast switching between static facial expressions.

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
| Duration strike | Sword flurry (1w, 3d) | Effect over duration |
| Movement | Walk (0w, 1d per tile) | Position updates over duration |
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
| Beam | 2 | 0 | 4 pt | Line AoE; penetration; friendly fire |
| Blink | 2 | 0 | 5 pt | Teleport; Chebyshev range 4 |

## 4.10 Combat UI Requirements
**Planning phase**: action queue, skill palette, `allocated / max` points, floating points, selected-action detail panel, undo/clear queue, lock-in button, aggressive/defensive toggle, last-target indicator, range overlays (green valid / red invalid), HP bars.

**Resolution phase**: phase label (Planning → Resolving), combat log, optional damage floaters.

**Blink targeting**: Highlight cursor tile green; show path from current tile; red for out-of-range; click valid tile to queue destination.

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
  - **Global Progression Flags**: (Unlocking historical calendar timelines, branching alternative story events, or unlocking save slots).

# ==============================================================================

# SECTION 6: AI COMMUNICATION & TUTORING PROTOCOL
- **Deep Explanations**: Do not output raw code dumps. Contextualize explanations deeply, teaching the underlying architectural theory. When beneficial, draw analogies to Java structural patterns to bridge gaps in Godot paradigms.
- **Production-Ready Deliverables**: Write comprehensive code blocks. Avoid truncation shortcuts like `# TODO: implement this` or `# your logic here` within core script files.
- **Diagnostic Safety Check**: Before printing code snippets, mentally verify they strictly comply with Godot 4 API standards to minimize compiler friction.
