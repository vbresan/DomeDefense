# Phase 9c — Missile Heads and Comet Tails

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Depends on** | Phase 9b.1 (accepted; launcher colours updated by hand) |
| **Phase goal** | **Visual only. No gameplay change.** Give **enemy missiles** (including MIRV warheads, smart bombs and the missiles dropped by bombers and satellites) and **ABMs** a proper look: a small bright **head** with a soft additive **halo**, and a **fading comet tail** that streams behind the head, grows out of the point where the missile started, and is brightest at the head. Enemy missiles are red, ABMs cyan-white, smart bombs magenta. Tails and halos are additive planes built from engine primitives and two new procedural materials; every colour is a material instance parameter. Positions, speeds, kills, scoring, waves and flow stay as they are. Flyer bodies (bombers, satellites) are **not** touched in this phase. |

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **No gameplay change.** Missile positions, speeds, targets, kills (an explosion kills a missile when the missile's **point** is inside it; the head size is irrelevant), impacts, scoring, waves, flyers, MIRV splits, smart-bomb dodging and game flow are unchanged. Only components, materials and visual calls are added.
3. **Existing logic must survive** unless a step says to change it. Read each graph (`read_graph_dsl`) before editing it.
4. **Compile after every Blueprint change and recompile every material.** Every Blueprint ends with no errors and no warnings; every material compiles without errors.
5. **Run each step's "Check"** before moving on. **Save All after each step's Check passes.**
6. **You cannot press keys, click, touch, move the mouse, or resize in PIE, and you cannot start a game.** No missile exists on the title screen, so **you cannot see any head or tail in PIE.** You can verify: compiles, static read-backs of every component, variable, function, graph and material parameter, and (Step 4) a calibration capture of a temporary actor in the **editor** world. Everything seen in play is group D (**NEEDS-HUMAN**); device checks are **NEEDS-DEVICE**. Never mark a D or T item PASS.
7. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
8. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after the run. **Save before loading a level.** **Do not touch Project Settings, input settings or any `Config/*.ini` file.** **Do not use, edit or delete anything in `/Game/DomeDefense/Spike/`.**
9. **NEVER REWRITE AN EXISTING FUNCTION OR EVENT GRAPH** with `write_graph_dsl`. A **brand-new** function is written **once**, in one small write. **Existing graphs are changed only by small node or pin edits** (`create_node`, `connect_pins`, `set_pin_value`).
10. **READ EVERY NEW OR CHANGED GRAPH BACK** right after writing it. If a body reads back empty or incomplete: write it once more; if it is still wrong, **stop and report** under *Tool problems*.
11. **VISIBILITY AUDIT (the lesson of the invisible smart bomb).** Every new visual component must have `bHiddenInGame` **FALSE**. Components that start switched off (`HaloSmart`, `TailSmart`) start with **`bVisible` false and `bHiddenInGame` false**; `ArmSmart` turns them on with `SetVisibility`, which never clears `bHiddenInGame`. Read both flags back for **every** new component and list them in the report.
12. **MOBILITY.** Every new component is **Movable** (the tail pivot is rotated and the tail planes are moved and scaled at run time). Read `Mobility` back for each.
13. **CHECK EVERY CALL YOU BUILD for a boolean or number pin left at its default.** Read the input pins back and list them in the report (the list is in A9).
14. **After any tool error, do not retry the same large operation.** Stop that item, try one smaller piece at most, and if it still fails **stop and report**. **If the editor stops responding or restarts, stop and report.** Do not continue with a half-edited asset.
15. **Order is part of the spec:** read and measure first, then materials, then `BP_EnemyMissile`, then `BP_ABM`, then the calibration, then the run.
16. **FROZEN: do not edit** every asset except `BP_EnemyMissile` and `BP_ABM` and the new assets of Step 1. In particular: `BP_DDGameMode` (all functions), `BP_DDPlayerController`, `BP_DDHUD`, `BP_Flyer`, `BP_Explosion`, `BP_Crosshair`, `BP_TapMarker`, `BP_MissileBase`, `BP_City`, `BP_Backdrop`, `BP_PlayfieldCamera`, `BP_PlayfieldDebug`, `PDA_DDConfig`, `DA_DDConfig`, `L_Game` (except the fully reverted proof of Step 4), and **every existing material and material instance** (including `MI_DD_Enemy`, `MI_DD_Abm`, `MI_DD_SmartBomb`, `M_DD_Unlit`). **The only edits to existing assets are listed in section 4.**
17. **No new keys, no new log lines, no new config values.** No new Tick on any actor: the existing `Event Tick` of the two Blueprints gets one added call each.
18. **Node IDs with parentheses cannot be written in the graph DSL** (`Clamp`, `Min`, `Max`, `Atan2 (Degrees)`, `Abs`). Create those nodes with `create_node` and connect them with `connect_pins`. **Guard every division by a variable with a Branch** (this phase divides only by constants; use multiplication by 0.01 instead of dividing by 100).
19. **Materials:** Additive blend, Unlit, two-sided, **opacity constant 1** (the pattern of `M_DD_Glow`), no Noise node, no texture sample, no Time node, **at most 40 expressions** each. Report the count.
20. **Do not clean up anything this document does not list.** (The old `MI_DD_Enemy` and `MI_DD_Abm` stay in the project, unused by the heads; they are removed in a later phase.)

---

## 2. Scope

### In scope
- Two new materials `M_DD_Tail`, `M_DD_Halo`; eight new material instances (3.6).
- `BP_EnemyMissile`: smaller head, halo, comet tail (and a smart-bomb variant of each), two variables, two new functions, three small edits.
- `BP_ABM`: smaller head, halo, comet tail, three variables, two new functions, three small edits.

### Out of scope (do NOT do)
- `BP_Flyer` bodies (bomber, satellite), explosions, the tap marker, the crosshair, launchers, cities, backdrop, HUD.
- A tail that stays behind after the missile dies, fades out or leaves a trail on the ground: **the tail disappears with its missile**.
- Persistent origin-to-head lines, particle systems, Niagara, textures, sound, post-processing, per-wave colours.
- Any key, config value, log line, Project or Android setting.

---

## 3. Design (decided; do not change)

### 3.1 What the player sees

| Missile | Head | Halo | Tail |
|---|---|---|---|
| Enemy missile, MIRV warhead, missile dropped by a flyer | small hot core `MI_DD_EnemyCore`, 18 units | soft red glow, 44 units | red comet tail, 240 long, 5 wide |
| Smart bomb (after `ArmSmart`) | the existing magenta `DotSmart`, now 35 units | soft magenta glow, 62 units | magenta comet tail, 240 long, 5 wide |
| ABM | small white-cyan core `MI_DD_AbmCore`, 14 units | soft cyan glow, 36 units | cyan comet tail, 420 long, 6 wide |

The tail is a flat additive strip that is **brightest at the head and fades to nothing at its far end**. It **grows out of the point where the missile started** (3.4), so an ABM leaves the launcher cap without a streak inside the launcher, and a MIRV warhead's tail begins at the split point.

### 3.2 Layers (relative Y of the visual components inside each actor; the actor itself sits at the play-plane Y)

| Part | Enemy missile | ABM |
|---|---|---|
| `TailPivot` / `Tail` / `TailSmart` | 68.5 | 78.5 |
| `Halo` / `HaloSmart` | 69.3 | 79.3 |
| `Dot` / `DotSmart` | 70 (existing) | 80 (existing) |

So the order front to back is head, halo, tail; all enemy parts are behind the ABM layer (78.5 and above) and all ABM parts are behind the explosion layer (90). The enemy tail (68.5) is in front of every city and launcher part (at most 66.5).

### 3.3 Orientation (decided)

- The tail is oriented **once**, in `Init`, along the straight line from the start to the target. It is **not** re-oriented per frame (a dodging smart bomb slides sideways and its tail stays parallel to the original line, attached to the head).
- `TailPivot` is a SceneComponent at relative (0, Y, 0) whose **only** rotation is a **pitch**: `TailPivot.SetRelativeRotation(Make Rotator(Roll 0, Pitch = phi, Yaw 0))`, where `phi = Atan2Degrees(dz, dx)` for the flight direction `d = Target - Start` (`dx = d.X`, `dz = d.Z`; the first argument of the node is **dz**, the second **dx**). A positive pitch turns the pivot's +X axis toward +Z, so the pivot's +X axis points along the flight direction. `Atan2Degrees(0, 0)` is 0.
- The tail plane is a child of `TailPivot`. Its **centre** is at pivot-local `(-len/2, 0, 0)`, so it extends **backward** from the head along -X, and its head end is at pivot-local X = 0.
- The planes are rolled 90 (relative rotation Roll 90, Pitch 0, Yaw 0, as in 9b) so that they face the camera; the plane's local X is the length and its local Y is the width. Relative scale of a tail plane is `(len/100, width/100, 1)`.

### 3.4 Tail length (decided)

Every frame: `len = Clamp(Distance(GetActorLocation, start), 0.1, TrailLen)`, where `start` is `StartPos` (enemy) or `OriginPos` (ABM). The tail therefore grows from 0 to `TrailLen` while the missile travels its first `TrailLen` units, then keeps that length. The per-frame update sets, on each tail plane, the relative location `(len * -0.5, 0, 0)` and the relative scale `(len * 0.01, TrailWidth * 0.01, 1)`.

### 3.5 Materials (all Additive, Unlit, two-sided, opacity constant 1; every colour goes to Emissive)

**`M_DD_Tail`**

| Parameter (exact name) | Type | Default |
|---|---|---|
| `Color` | vector | (1, 1, 1) |
| `Intensity` | scalar | 1.0 |
| `Power` | scalar | 1.5 |
| `Axis` | scalar | 0 |
| `FlipU` | scalar | 0 |

```
uv        = TextureCoordinate(0)
alongRaw  = Lerp(uv.x, uv.y, Axis)                 // coordinate along the strip (Axis 0: U, Axis 1: V)
crossRaw  = Lerp(uv.y, uv.x, Axis)                 // coordinate across the strip
t         = Lerp(alongRaw, 1 - alongRaw, FlipU)    // 1 at the head end
alongFade = Power(Saturate(t), Power)              // 0 at the far end, 1 at the head end
crossFade = Saturate(1 - Abs(2 * crossRaw - 1))    // 1 on the centre line, 0 at both edges
Emissive  = Color * Intensity * alongFade * crossFade
Opacity   = 1 (constant)
```
`Axis` and `FlipU` are **calibration parameters** for the engine plane's UV orientation (Step 4). Set on the **parent material defaults only**, never on the instances, so one change flips every tail. The assumed layout (`Axis` 0, `FlipU` 0) is: U runs along the plane's local X and increases toward +X, which is toward the head.

**`M_DD_Halo`**

| Parameter | Type | Default |
|---|---|---|
| `Color` | vector | (1, 1, 1) |
| `Intensity` | scalar | 1.0 |
| `Power` | scalar | 2.0 |

```
r        = Distance(TextureCoordinate(0), (0.5, 0.5)) * 2
g        = Power(Saturate(1 - r), Power)
Emissive = Color * Intensity * g
Opacity  = 1 (constant)
```

### 3.6 Material instances (parent in brackets; create in `/Game/DomeDefense/Materials/`)

| Instance | Parent | Parameters |
|---|---|---|
| `MI_DD_EnemyCore` | `M_DD_Unlit` | `Color` (1.0, 0.45, 0.35) |
| `MI_DD_AbmCore` | `M_DD_Unlit` | `Color` (0.85, 1.0, 1.0) |
| `MI_DD_TailEnemy` | `M_DD_Tail` | `Color` (1.0, 0.16, 0.05), `Intensity` 1.0 |
| `MI_DD_TailAbm` | `M_DD_Tail` | `Color` (0.20, 0.85, 1.0), `Intensity` 1.0 |
| `MI_DD_TailSmart` | `M_DD_Tail` | `Color` (1.0, 0.15, 0.90), `Intensity` 1.0 |
| `MI_DD_HaloEnemy` | `M_DD_Halo` | `Color` (1.0, 0.12, 0.06), `Intensity` 0.7 |
| `MI_DD_HaloAbm` | `M_DD_Halo` | `Color` (0.20, 0.85, 1.0), `Intensity` 0.7 |
| `MI_DD_HaloSmart` | `M_DD_Halo` | `Color` (1.0, 0.15, 0.90), `Intensity` 0.8 |

### 3.7 Components

All new components: collision disabled (`bodyInstance.collisionEnabled` = `NoCollision`), `CastShadow` false, **Movable**, and the flags of rule 11. Planes are `/Engine/BasicShapes/Plane` (Step 0 re-measures it: 9b measured 100 × 100, thickness 0, pivot at the centre); scale = target size / measured size. Positions are relative to the actor root.

**`BP_EnemyMissile`** (existing: `Root`, `Dot`, `DotSmart`)

| Component | Parent | Type | Relative location | Rotation | Relative scale | Material | Initial flags |
|---|---|---|---|---|---|---|---|
| `Dot` (**edit**) | Root | Sphere (existing) | (0, 70, 0) unchanged | none | **(0.18, 0.18, 0.18)** | **`MI_DD_EnemyCore`** | unchanged |
| `DotSmart` (**edit**) | Root | Sphere (existing) | (0, 70, 0) unchanged | none | **(0.35, 0.35, 0.35)** | `MI_DD_SmartBomb` unchanged | unchanged (`bVisible` false) |
| `Halo` | Root | Plane | (0, 69.3, 0) | Roll 90 | (0.44, 0.44, 1) | `MI_DD_HaloEnemy` | visible |
| `HaloSmart` | Root | Plane | (0, 69.3, 0) | Roll 90 | (0.62, 0.62, 1) | `MI_DD_HaloSmart` | `bVisible` false |
| `TailPivot` | Root | SceneComponent | (0, 68.5, 0) | none | 1 | none | visible |
| `Tail` | `TailPivot` | Plane | (0, 0, 0) | Roll 90 | **(0.001, 0.05, 1)** | `MI_DD_TailEnemy` | visible |
| `TailSmart` | `TailPivot` | Plane | (0, 0, 0) | Roll 90 | **(0.001, 0.05, 1)** | `MI_DD_TailSmart` | `bVisible` false |

(The tiny X scale keeps a tail invisible until the first update.)

**`BP_ABM`** (existing: `Root`, `Dot`)

| Component | Parent | Type | Relative location | Rotation | Relative scale | Material | Initial flags |
|---|---|---|---|---|---|---|---|
| `Dot` (**edit**) | Root | Sphere (existing) | (0, 80, 0) unchanged | none | **(0.14, 0.14, 0.14)** | **`MI_DD_AbmCore`** | unchanged |
| `Halo` | Root | Plane | (0, 79.3, 0) | Roll 90 | (0.36, 0.36, 1) | `MI_DD_HaloAbm` | visible |
| `TailPivot` | Root | SceneComponent | (0, 78.5, 0) | none | 1 | none | visible |
| `Tail` | `TailPivot` | Plane | (0, 0, 0) | Roll 90 | **(0.001, 0.06, 1)** | `MI_DD_TailAbm` | visible |

### 3.8 Variables (Instance Editable, category `Trail`; these are the tuning knobs)

| Blueprint | Variable | Type | Default |
|---|---|---|---|
| `BP_EnemyMissile` | `TrailLen` | float | 240.0 |
| `BP_EnemyMissile` | `TrailWidth` | float | 5.0 |
| `BP_ABM` | `TrailLen` | float | 420.0 |
| `BP_ABM` | `TrailWidth` | float | 6.0 |
| `BP_ABM` | `OriginPos` | vector (not editable) | (0, 0, 0) |

### 3.9 Budget
New components: enemy +5 (`Halo`, `HaloSmart`, `TailPivot`, `Tail`, `TailSmart`), ABM +3 (`Halo`, `TailPivot`, `Tail`). New assets: 2 materials and 8 instances (each at most 40 expressions). Shared instances only. The only per-frame logic is `UpdateTrail` (two or four component calls per missile). With a busy wave of about 25 missiles and a few ABMs the cost is negligible.

### 3.10 `ArmSmart` edit
After the existing calls (`Dot.SetVisibility(false)`, `DotSmart.SetVisibility(true)`), append, each with **New Visibility** as stated and **Propagate To Children false**:
`Halo.SetVisibility(false)`, `Tail.SetVisibility(false)`, `HaloSmart.SetVisibility(true)`, `TailSmart.SetVisibility(true)`.

---

## 4. What this phase changes in earlier specs

| Earlier text | Now |
|---|---|
| Phase 4 `BP_EnemyMissile.Dot`: scale 0.4, material `MI_DD_Enemy`, "No trail" | Scale 0.18, material `MI_DD_EnemyCore`; plus `Halo`, `TailPivot`, `Tail`. |
| Phase 8c `DotSmart`: scale 0.7 | Scale 0.35; plus `HaloSmart`, `TailSmart`. Still `bVisible` false by default. |
| Phase 8c `ArmSmart` | Four `SetVisibility` calls appended (3.10). Nothing else changes. |
| Phase 4 `BP_EnemyMissile.Init` | One call `AimTrail()` appended at the end. |
| Phase 4 `BP_EnemyMissile.Event Tick` | One call `UpdateTrail()` after `Advance(DeltaSeconds)`. |
| Phase 2 `BP_ABM.Dot`: scale 0.25, material `MI_DD_Abm` | Scale 0.14, material `MI_DD_AbmCore`; plus `Halo`, `TailPivot`, `Tail`. |
| Phase 2 `BP_ABM.Init` | `OriginPos = StartPos` and one call `AimTrail()` appended at the end. |
| Phase 2 `BP_ABM.Event Tick` | One call `UpdateTrail()` in the branch that moves the ABM. |
| 9b.1 frozen list | `BP_EnemyMissile` and `BP_ABM` are unfrozen **for the edits of this document only**. |

---

## 5. Tooling notes (from the 9a, 9b, 9b.1 and 8c reports)

1. Components on a Blueprint: `ActorTools.add_component` (engine meshes by path), then **`set_parent_component`**; after each re-parent **read back the parent and the relative transform** and correct the location if it changed. Edit an existing component's scale and material with the object-property tools; read back.
2. `StaticMeshTools.get_bounds` measures an engine mesh. `CastShadow` (not `bCastShadow`) is the shadow property.
3. Material work: `MaterialTools.create_material`, `add_expression`, `connect_expressions`, `connect_to_output`, `get_expressions`, `recompile`; the class for interpolation is `MaterialExpressionLinearInterpolate`; the blend mode is set on the material; instances with `MaterialInstanceTools.create`, `set_parent`, scalar and vector setters, `list_parameters`. Use `list_expression_classes` once for exact class names (TextureCoordinate, ComponentMask, Abs, Saturate, Power, OneMinus, Multiply, Subtract, Distance, Constant2Vector, ScalarParameter, VectorParameter, LinearInterpolate). Delete unused nodes (`delete_unused_expressions`).
4. New functions: `BlueprintTools.create` and one small `write_graph_dsl` per **new** function; read it back. Parenthesised node IDs come from `create_node` (rule 18). The degrees version of Atan2 is found by a node search for "Atan2"; if only a radians version exists, multiply the result by 57.29578 and record it.
5. `Make Rotator` has the pins Roll, Pitch, Yaw. Wire **Pitch** only; leave Roll and Yaw at 0 and read them back.
6. **PIE cannot spawn actors** and no missile exists on the title screen. `CaptureEditorImage` captures the PIE title screen; `CaptureViewport` needs a camera transform and shows the **editor** world (use `(0, 3000, 900)` as in 9b).
7. Store every value in a local before something it depends on changes. A non-Pure function needs exec wires.
8. If a tool reports that a property cannot be read or set, say so; do not guess.

---

## 6. Implementation steps

### Step 0 — Read and measure (no changes)
1. **`BP_EnemyMissile`:** every component (parent, transform, mesh, material, `bVisible`, `bHiddenInGame`, `Mobility`, collision flag); variables; the graphs of `Init`, `Advance`, `ArmSmart`, `Die`, `Impact`, `Event Tick`. For `Event Tick` name the node that calls `Advance`. For `ArmSmart` list its existing calls.
2. **`BP_ABM`:** the same; the graphs of `Init`, `OnArrived` and `Event Tick`. For `Event Tick` name the node that moves the ABM (the `SetActorLocation` in the branch that is not the arrival branch).
3. Read `M_DD_Unlit`, `M_DD_Glow` (the additive pattern: blend mode, two-sided, opacity), `MI_DD_Enemy`, `MI_DD_Abm`, `MI_DD_SmartBomb` (Color values only).
4. **Measure** `/Engine/BasicShapes/Plane` and `/Engine/BasicShapes/Sphere` (size, pivot). 9b measured the plane as 100 × 100 with thickness 0 and a centred pivot and the sphere as 100 × 100 × 100; confirm.
5. Run `list_expression_classes` and record the class names for 3.5.

**Check:** the report section for Step 0 is complete (components with parents and flags, the `Event Tick` and `Init` node names, expression class names, mesh sizes).

### Step 1 — Materials
Create `M_DD_Tail` and `M_DD_Halo` (3.5) in `/Game/DomeDefense/Materials/`: blend **Additive**, shading Unlit, two-sided, opacity constant 1. Recompile and read back `get_expressions` (each at most 40). Then create the eight instances of 3.6 and set every parameter. Read all parameters back (`list_parameters`), including `Axis` 0 and `FlipU` 0 on `M_DD_Tail`.

**Check:** both materials recompile without errors; every parameter reads back; instance parents are as in 3.6; expression counts listed. **Save All.**

### Step 2 — `BP_EnemyMissile`
Read the Blueprint again first.
1. **Edit** `Dot` (scale (0.18, 0.18, 0.18), material `MI_DD_EnemyCore`) and `DotSmart` (scale (0.35, 0.35, 0.35)); read back; do not touch their visibility.
2. **Add** `Halo`, `HaloSmart`, `TailPivot` (all under `Root`), `Tail` and `TailSmart` (under `TailPivot`) as in 3.7. Set mesh, location, rotation, scale, material, collision, `CastShadow` false, Movable, and the flags of rule 11 (`HaloSmart` and `TailSmart`: `bVisible` false, **`bHiddenInGame` false**). Read parent, transform, material and flags back for every new component.
3. **Variables:** `TrailLen` (240.0) and `TrailWidth` (5.0): float, Instance Editable, category `Trail`.
4. **New function `AimTrail`** (no inputs, non-Pure). Write it once; read it back:
```
d   = Target - StartPos                                   // store (vector)
phi = Atan2Degrees(d.Z, d.X)                              // store; first argument d.Z, second d.X
TailPivot.SetRelativeRotation(Make Rotator(Roll 0, Pitch phi, Yaw 0), bSweep false)
```
5. **New function `UpdateTrail`** (no inputs, non-Pure). Write it once; read it back:
```
If NOT bActive → return
loc  = GetActorLocation                                   // store
trav = Distance(loc, StartPos)                            // store
len  = Clamp(trav, 0.1, TrailLen)                         // create_node for Clamp; store
loc2 = Make Vector(len * -0.5, 0.0, 0.0)                  // store
sc   = Make Vector(len * 0.01, TrailWidth * 0.01, 1.0)    // store
Tail.SetRelativeLocation(loc2)      ; Tail.SetRelativeScale3D(sc)
TailSmart.SetRelativeLocation(loc2) ; TailSmart.SetRelativeScale3D(sc)
```
6. **`Init`:** append one call `AimTrail()` at the end (after the last existing node). Read `Init` back; nothing else changes.
7. **`Event Tick`:** insert one call `UpdateTrail()` **after** `Advance(DeltaSeconds)`. Read back.
8. **`ArmSmart`:** append the four `SetVisibility` calls of 3.10. Read back every pin (rule 13).
9. Compile.

**Check:** `BP_EnemyMissile` compiles clean; the five new components and the two edited ones read back as in 3.7 with the flags of rules 11 and 12; `AimTrail`, `UpdateTrail` read back non-empty; `AimTrail` is called only from `Init`, `UpdateTrail` only from `Event Tick`; `Advance`, `Die`, `Impact`, `Split`, `CheckSplit`, `ApplyDodge`, `ComputePush` unchanged; both tick flags still true. **Save All.**

### Step 3 — `BP_ABM`
Read the Blueprint again first.
1. **Edit** `Dot` (scale (0.14, 0.14, 0.14), material `MI_DD_AbmCore`); read back.
2. **Add** `Halo` and `TailPivot` (under `Root`) and `Tail` (under `TailPivot`) as in 3.7, with the same property and flag requirements. Read back every new component.
3. **Variables:** `TrailLen` (420.0) and `TrailWidth` (6.0): float, Instance Editable, category `Trail`; `OriginPos`: vector, not Instance Editable.
4. **New function `AimTrail`** (no inputs, non-Pure). Write once; read back:
```
d   = Target - OriginPos                                  // store
phi = Atan2Degrees(d.Z, d.X)                              // store; first argument d.Z, second d.X
TailPivot.SetRelativeRotation(Make Rotator(Roll 0, Pitch phi, Yaw 0), bSweep false)
```
5. **New function `UpdateTrail`** (no inputs, non-Pure). Write once; read back:
```
If NOT bActive → return
loc  = GetActorLocation                                   // store
trav = Distance(loc, OriginPos)                           // store
len  = Clamp(trav, 0.1, TrailLen)                         // create_node for Clamp; store
loc2 = Make Vector(len * -0.5, 0.0, 0.0)                  // store
sc   = Make Vector(len * 0.01, TrailWidth * 0.01, 1.0)    // store
Tail.SetRelativeLocation(loc2) ; Tail.SetRelativeScale3D(sc)
```
6. **`Init`** (input `StartPos` is the function's input; the variable is `OriginPos`): after the existing `SetActorLocation(StartPos)` set `OriginPos = StartPos`, and at the end of the function call `AimTrail()`. Read back; `Target` must already be set before `AimTrail` runs (it is set at the top of `Init`); nothing else changes.
7. **`Event Tick`:** insert one call `UpdateTrail()` in the branch that **moves** the ABM, after its `SetActorLocation`. Do **not** put it in the arrival branch. Read back.
8. Compile.

**Check:** `BP_ABM` compiles clean; components and flags as required; `OriginPos` is written only in `Init`; `AimTrail` is called only from `Init`, `UpdateTrail` only from `Event Tick`; `OnArrived` unchanged; the tick flag is still true. **Save All.**

### Step 4 — Calibration of the tail gradient (recommended; a few calls)
Goal: make the tail **bright at the head end and fading toward the far end**, whatever the plane's UV layout is.
1. List the actors in `L_Game` (record the count). Place **one** temporary `BP_EnemyMissile` at (0, 0, 900).
2. On that instance set the `Tail` component to relative location `(-120, 0, 0)` and relative scale `(2.4, 0.5, 1)` (a 240 × 50 strip; the instance is only for this test). The `Dot` (the head) is at the right end of the strip.
3. `CaptureViewport` from `(0, 3000, 900)`. Describe the strip: where it is brightest, and whether it fades **along** its length.
   - Bright at the right end (next to the head) and fading to the left: **correct**, change nothing.
   - Bright at the left end: set `FlipU` to **1** on the **`M_DD_Tail` defaults** (not on the instances).
   - The strip fades **across its width** (a bright band along the middle, with the 240-long direction uniform): set `Axis` to **1** on the `M_DD_Tail` defaults, then capture again and apply the `FlipU` rule.
4. Recompile `M_DD_Tail`, capture again to confirm, and record the final `Axis` and `FlipU`.
5. **Revert:** delete the temporary actor, save the level, list the actors again: the count and the list **must equal** the one recorded in step 1 and `L_Game` must not be dirty. If anything cannot be reverted, **stop and report**.

**Fallback:** if the capture does not show the strip clearly or the actor cannot be placed, skip the step, leave `Axis` 0 and `FlipU` 0 and record why; the human flips the parameters on `M_DD_Tail` if needed (D2).

**Check:** the final `Axis` and `FlipU` are recorded; the level is exactly as before. **Save All.**

### Step 5 — Run
1. **Save All.**
2. `StartPIE`. The title screen appears at once. **Wait about 20 seconds**; nobody taps, so no game starts and no missile exists.
3. `CaptureEditorImage` at about 5 s and again at about 15 s. Describe each image: the title screen as in 9b.1 (backdrop, cities, three launchers at rest, ammo numbers, title text); nothing else changed.
4. `GetLogEntries` (filter `[DD]`), then `StopPIE`.
5. Fill in the report (section 8).

---

## 7. Acceptance tests

### A. Static checks
- [ ] **A1** New assets: `M_DD_Tail`, `M_DD_Halo`, `MI_DD_EnemyCore`, `MI_DD_AbmCore`, `MI_DD_TailEnemy`, `MI_DD_TailAbm`, `MI_DD_TailSmart`, `MI_DD_HaloEnemy`, `MI_DD_HaloAbm`, `MI_DD_HaloSmart`. Changed: `BP_EnemyMissile`, `BP_ABM`. Nothing else, nothing under `Spike`, no Project Settings, input settings or `.ini` change, no config change. `L_Game` unchanged.
- [ ] **A2** `BP_EnemyMissile` and `BP_ABM` compile with 0 errors and 0 warnings; every new material recompiles without errors; each has at most 40 expressions; no Noise, texture or Time node; blend mode Additive, two-sided, opacity constant 1.
- [ ] **A3** All material parameters read back as in 3.5 and 3.6 (with the calibrated `Axis` and `FlipU` recorded).
- [ ] **A4** `BP_EnemyMissile`: components, parents, locations, rotations, scales, materials as in 3.7; `Dot` scale 0.18 and `MI_DD_EnemyCore`; `DotSmart` scale 0.35 and `bVisible` false; `HaloSmart` and `TailSmart` `bVisible` false; `TrailLen` 240 and `TrailWidth` 5 (Instance Editable, category `Trail`).
- [ ] **A5** `BP_ABM`: components as in 3.7; `Dot` scale 0.14 and `MI_DD_AbmCore`; `TrailLen` 420, `TrailWidth` 6, `OriginPos`.
- [ ] **A6** `AimTrail` and `UpdateTrail` (both Blueprints) read back as in Steps 2 and 3; `Make Rotator` reads Roll 0, Pitch `phi`, Yaw 0; the `Atan2` arguments are (`Z`, `X`) in that order; `Clamp` reads (`trav`, 0.1, `TrailLen`).
- [ ] **A7** Hooks: `Init` ends with `AimTrail()` in both Blueprints (and the ABM's `Init` sets `OriginPos`); `Event Tick` calls `UpdateTrail()` after `Advance(DeltaSeconds)` (enemy) and in the moving branch only (ABM); `ArmSmart` ends with the four `SetVisibility` calls of 3.10.
- [ ] **A8** **Visibility audit:** a table of every new component with `bVisible`, `bHiddenInGame` and `Mobility`: `bHiddenInGame` false and `Mobility` Movable everywhere; `bVisible` true except `HaloSmart` and `TailSmart`.
- [ ] **A9** **Call audit (rule 13):** a table of every pin: `Make Rotator` (Roll, Pitch, Yaw) and `SetRelativeRotation` (`bSweep`) in both `AimTrail`; `Atan2` arguments; `Clamp` (value, Min, Max); the two `Make Vector` nodes of `UpdateTrail` (X, Y, Z, with the 0.01 and -0.5 factors); `SetRelativeLocation` and `SetRelativeScale3D` targets in both `UpdateTrail`; the four `SetVisibility` calls in `ArmSmart` (target, New Visibility, Propagate To Children); the `UpdateTrail` call positions.
- [ ] **A10** Unchanged: every frozen asset of rule 16; `Advance`, `Die`, `Impact`, `Split`, `CheckSplit`, `ApplyDodge`, `ComputePush`, `OnArrived` read as before; no new log line, key or config value.
- [ ] **A11** Calibration (Step 4): the final `Axis` and `FlipU` and a description of the capture, or the recorded fallback; if the step ran, `L_Game` has the same actors as before.

### B. Log checks (agent; the one 20 s run, nobody taps)
- [ ] **B1** The startup lines as in the 9b.1 run: camera, GameMode, config (`Width=2250.0 Height=1800.0`), bases 3, cities 6, PlayerController, view target, crosshair, HUD, `[DD][Area] Viewport=...`.
- [ ] **B2** Exactly one of `[DD][Save] Loaded high=<n>` or `[DD][Save] No file high=<n>`, followed by exactly one `[DD][Flow] Title high=<n>`.
- [ ] **B3** During the 20 s wait: no `Spawner started`, no `[DD][Wave]`, no `[DD][Enemy]`, no `[DD][Flyer]`, no `[DD][Score]`, no `[DD][Damage]`, no `[DD][Input] Fire request`, no `[DD][Aim]`, no `[DD][Game]`, and no `[DD][Flow]` line except the one `Title` line.
- [ ] **B4** No line containing `ERROR`, `Accessed None`, `Divide by zero`, `Blueprint Runtime Error`, or a material or shader compile error.

### C. Visual checks (captures of the title screen; describe in words)
- [ ] **C1** The title screen is unchanged from 9b.1: sky, glow, mountains, six cities, three launchers at rest, ammo numbers, title text. No new object appears (no missile exists).
- [ ] **C2** Step 4 capture (if run): the test strip is bright at the head end and fades along its length.

### D. Interactive checks (human, PIE) — all NEEDS-HUMAN
List every item as NEEDS-HUMAN in the report; do not run or claim them.
- [ ] **D1** **Enemy missile:** a small hot head with a soft red halo and a red tail behind it. The tail grows out of the spawn point during the first moments of the flight and fades toward its far end.
- [ ] **D2** **Gradient direction:** the tail is brightest next to the head. If it is brightest at the far end, set `FlipU` to 1 on the `M_DD_Tail` defaults; if it fades across the width instead of along the length, set `Axis` to 1 (Step 4 covers this when it could run).
- [ ] **D3** **Direction:** the tail points straight back along the line to where the missile started, for missiles flying left, right and straight down.
- [ ] **D4** **MIRV:** at the split, each warhead's tail starts at the split point; the parent keeps its own tail.
- [ ] **D5** **ABM:** a white-cyan head with a cyan tail that grows out of the launcher cap; no streak inside the launcher; the center base's very fast ABM still shows a short bright streak.
- [ ] **D6** **Smart bomb:** a larger magenta head with a magenta halo and tail; during a dodge the head slides sideways and the tail stays attached to it, parallel to the original line.
- [ ] **D7** **Flyer missiles:** missiles dropped by bombers and satellites also have red tails.
- [ ] **D8** **Layering:** heads, halos and tails are drawn in front of cities, launchers, mountains and glow; explosions are drawn in front of them; ABMs are in front of enemy missiles.
- [ ] **D9** **Readability:** red enemies, cyan ABMs and magenta smart bombs stand out against the sky, mountains, launchers and cities at a small and at a very wide window.
- [ ] **D10** **No gameplay change:** kills (the smaller head does not change what an explosion kills), scoring, waves, flyers, pause, quit behave as before; a tail disappears the instant its missile dies, impacts or arrives.
- [ ] **D11** A busy late wave runs without a hitch.

### D-DEVICE. Android (NEEDS-DEVICE)
- [ ] **T-1** Heads, halos and tails appear as soft additive glows (no solid or black boxes around them).
- [ ] **T-2** The frame rate in a busy wave is not noticeably lower than before this phase.
- [ ] **T-3** The tails are readable at phone size (about 5 units is a few pixels).

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped; the editor is on `L_Game`; nothing under `Spike` was touched; if Step 4 ran, the level is as before.

---

## 8. Report format (the agent's final message)

```
## Phase 9c report
**Engine version:** x.y.z
**MCP tools used:** (list)

### Step 0 findings
(components of BP_EnemyMissile and BP_ABM with parents and flags; the Event Tick and Init node names;
 expression class names; Plane and Sphere sizes and pivots)

### Assets created or changed
(table: path | type | change | compiled clean)

### Materials
(table: material | blend mode | expression count | parameters read back | notes; the instances' parameters)

### Component inventory
(table: Blueprint | component | parent | mesh | scale | location | material | bVisible | bHiddenInGame | Mobility | collision flag)

### Function and graph read-backs
(AimTrail and UpdateTrail in both Blueprints; Init, Event Tick, ArmSmart before and after: which nodes changed)

### Call audit (rule 13)
(table: function | call | pin | wired from / value)

### Calibration (Step 4)
(what the capture showed; final Axis and FlipU; or why it was skipped; level actor count before and after)

### Acceptance results
A1 PASS/FAIL ... through E1 PASS/FAIL/NEEDS-HUMAN/NEEDS-DEVICE

### Log excerpt
(all [DD] lines of the run, with timestamps)

### Screenshot notes
(paths of the captures and a description of each)

### Tool problems (errors, retries, editor restarts, writes that read back empty)
### Deviations
(fallbacks used, the Atan2 variant, anything else)

### Open questions / risks
```

---

## 9. What this phase hands to later phases

| Provided | How a later phase uses it |
|---|---|
| `M_DD_Tail`, `M_DD_Halo` and their instances | Per-wave or per-theme colours by changing the instance parameters; the same materials for flyer exhaust or explosion debris. |
| `TrailLen`, `TrailWidth`, `Intensity`, `Power` | Tuning without a patch. |
| `Halo` planes | A glow behind any head; a muzzle flash or an explosion core can use `M_DD_Halo` too (9d). |
| `ArmSmart` visual switch | A different kind of missile can switch to its own head, halo and tail the same way. |

**Known leftovers that stay:** `StopSpawner`, `bDirectorSilent`, the two disconnected spawn chains in `SpawnRandomEnemy`, the disconnected older fire chain in `LaunchAt`, HUD debug lines 2 to 4, the overlay `BP_PlayfieldDebug`, the debug keys, the unused `MI_DD_Enemy` and `MI_DD_Abm`, the `Spike` folder (removed in 9g).

---

## 10. Common pitfalls

1. **The tail is oriented the wrong way.** `phi = Atan2Degrees(dz, dx)` with the first argument **dz**; the pivot's +X axis must point along the flight direction and the plane extends backward (-X). Swapped arguments make the tail point sideways.
2. **`Atan2` in radians** used as degrees. Use the degrees node or multiply by 57.29578.
3. **Roll or Yaw wired instead of Pitch** on `Make Rotator`. Wire Pitch only.
4. **The plane's scale axes swapped.** The planes are rolled 90 like the 9b planes: local X is the length, local Y is the width, thickness scale 1. A tail scaled `(width, length, 1)` is a short vertical bar.
5. **The tail drawn from the missile to the start for the whole flight** (a persistent line). The length is clamped to `TrailLen` and grows from 0.1 (3.4).
6. **`UpdateTrail` in the arrival branch of the ABM** (the actor is being destroyed) or before `Init` has set `OriginPos`. It goes in the moving branch only and begins with `If NOT bActive → return`.
7. **`AimTrail` before `Target` and the start are set.** It is the last call in `Init`.
8. **`bHiddenInGame` left true** on `HaloSmart` or `TailSmart` (the invisible smart bomb again). `bVisible` false, `bHiddenInGame` false, read back.
9. **`ArmSmart` pins left at default:** `SetVisibility` needs the right New Visibility on each of the four calls, Propagate false. A smart bomb with two heads or two tails means a wrong pin.
10. **A Static component** among the new ones. All are Movable (rule 12).
11. **The gradient bright at the wrong end** or across the width: the calibration parameters `Axis` and `FlipU` on the **`M_DD_Tail` defaults** (not on the instances) fix it (Step 4, D2).
12. **A two-sided material missing**: the plane faces away from the camera in one orientation and disappears. All four materials of this phase are two-sided.
13. **An opaque or translucent blend mode** instead of Additive: the tail would draw as a box. Additive, opacity constant 1.
14. **Editing `MI_DD_Enemy`, `MI_DD_Abm` or `MI_DD_SmartBomb`.** They are frozen; the cores use new instances.
15. **Leaving the Step 4 actor in the level.** The actor count must match the count before.
16. **Marking D or T items PASS.** You cannot see a missile in PIE. They are NEEDS-HUMAN and NEEDS-DEVICE.
17. **Touching `Spike` or cleaning up what this document does not list.**
