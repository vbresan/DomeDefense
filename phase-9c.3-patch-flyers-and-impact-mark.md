# Phase 9c.3 (patch) — Flyer Look (Bomber, Satellite) and Impact Mark (ABM Target Marker)

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Depends on** | Phase 9c.2 (built and tuned by hand by the human; the repository `master` at commit "Getting the rockets the way I want"). `BP_EnemyMissile` and `BP_ABM` are in their final, hand-tuned state and are **not touched** by this patch. |
| **Phase goal** | **Visual only. No gameplay change.** (1) The **bomber** and the **satellite** (both are `BP_Flyer`) stop being plain cubes: each becomes a small side-view silhouette built from engine shapes, with a soft glow like the missiles. The bomber faces the way it flies. (2) The **ABM target marker** (`BP_TapMarker`, today one 50-unit ball) becomes a white-cyan **X** with a soft glow, and it lives long enough to stay until its ABM explodes. |

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **No gameplay change.** A flyer's position, speed, direction, height, shots, kill test and score are unchanged. The kill test uses one point, the flyer's actor location (it calls `ContainsPoint`; confirm in Step 0 and report); the new shapes are centred on that point. An ABM's target, flight, explosion and the marker's removal logic (`Dismiss`) are unchanged.
3. **HANDS OFF.** Do not change, and report as unchanged after the work:
   - `BP_Flyer`: `DefaultSceneRoot`, `Root`, all variables and all functions and graphs **except the edits of Step 2.4 and 2.5**; the two old cube components `BodyBomber` and `BodySatellite` **except** that their `bVisible` becomes false (Step 2.1; their mesh, material, location, rotation and scale stay).
   - `BP_TapMarker`: `DefaultSceneRoot`, `Root`, all graphs and variables **except** the default of `Life` and, only in case (b) of 3.5, one literal pin value (Step 3.3); the component `Dot` **except** that its `bVisible` becomes false (its mesh, material, location and scale stay).
   - `BP_EnemyMissile`, `BP_ABM`, `BP_Crosshair` and every other Blueprint and asset not named above: **frozen**.
   - **Every existing material and material instance.** (Two new instances are created in Step 1.)
4. **Existing logic must survive.** Read each graph (`read_graph_dsl`) before editing it.
5. **Compile after every Blueprint change.** Every Blueprint ends with no errors and no warnings.
6. **Run each step's "Check"** before moving on. **Save All after each step's Check passes.**
7. **You cannot press keys, click, touch, move the mouse, or resize in PIE, and you cannot start a game.** No flyer and no marker exists on the title screen, so **you cannot see either in PIE.** You can verify: compiles, static read-backs of every component, flag and graph change, and (Step 4) captures of temporary actors in the **editor** world, where no `Init` or `BeginPlay` runs. Everything seen in play is group D (**NEEDS-HUMAN**); device checks are **NEEDS-DEVICE**. Never mark a D or T item PASS.
8. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
9. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after the run. **Save before loading a level.** **Do not touch Project Settings, input settings or any `Config/*.ini` file.** **Do not use, edit or delete anything in `/Game/DomeDefense/Spike/`.**
10. **NEVER REWRITE AN EXISTING FUNCTION OR EVENT GRAPH** with `write_graph_dsl`. `Init` of `BP_Flyer` is changed only with `create_node`, `connect_pins` and (for pins that are not wired) `set_pin_value`. No function is added.
11. **READ EVERY CHANGED GRAPH BACK** right after changing it. If it reads back wrong or incomplete: correct it once more; if it is still wrong, **stop and report** under *Tool problems*.
12. **MEASURE, DO NOT ASSUME.** Sizes below are target sizes in world units. Every scale in this document assumes the engine shapes measure 100 x 100 x 100 with the pivot at the centre. Step 0 measures every engine shape used; **wherever a measured size differs from 100, compute the scale as target size / measured size** and place the part so that the centre of its bounding box lands on the stated centre. Record what you measured and what you used.
13. **VISIBILITY AUDIT.** Every new visual component must have **`bHiddenInGame` FALSE** and be **Movable**; `bVisible` follows rule V below. Read `bVisible`, `bHiddenInGame` and `Mobility` back for **every** new or edited component and list them in the report (A8).
    **Rule V:** each group (`GroupBomber`, `GroupSatellite`) and every part inside it gets the **same `bVisible` default that the Step 0 read-out shows for the matching old cube** (`BodyBomber` for the bomber group, `BodySatellite` for the satellite group). This keeps the existing show/hide logic of `Init` working exactly as it does today.
14. **CHECK EVERY PIN YOU BUILD** (the lesson of the Phase 7 pause defect: a boolean pin left at its default). After the edits of Step 2.4 and 2.5, read all input pins of every node you created or re-pointed back, and list them (A9).
15. **After any tool error, do not retry the same large operation.** Stop that item, try one smaller piece at most, and if it still fails **stop and report**. **If the editor stops responding or restarts, stop and report.**
16. **Order is part of the spec:** read and measure first, then the materials, then `BP_Flyer`, then `BP_TapMarker`, then the calibration, then the run.
17. **FROZEN: do not edit** every asset except `BP_Flyer`, `BP_TapMarker`, the two new instances of Step 1, and `L_Game` (only the fully reverted proof of Step 4).
18. **No new keys, no new log lines, no new config values, no new variables, no new functions, no new Tick.** The only additions are the components, the two material instances and the graph edits named in this document.
19. **Do not clean up anything this document does not list.** (The hidden cubes and the hidden `Dot` stay.)
20. **If the project is not in the state described in "Depends on"** (a component name of section 3 already exists, `BodyBomber`/`BodySatellite` are missing, or `BP_TapMarker` does not have the single `Dot` described), **stop and report** before changing anything.

---

## 2. Scope

### In scope
- Two new material instances `MI_DD_HaloBomber` and `MI_DD_HaloSatellite`.
- `BP_Flyer`: new components, the old cubes hidden, `Init` edited in two places (re-pointed visibility calls; mirroring).
- `BP_TapMarker`: new components, `Dot` hidden, default `Life` raised.

### Out of scope (do NOT do)
- Any change to a flyer's hit test or size-aware hits (a gameplay question for later).
- Any change to colours of existing materials.
- Any animation, tail, light blinking or new tick logic.
- Anything in `BP_EnemyMissile`, `BP_ABM`, `BP_Explosion`, the HUD or the game mode.

---

## 3. Design (decided; do not change)

### 3.1 Coordinate conventions
The play plane is X (right) and Z (up). The camera looks along -Y, so **Y is depth**: a larger Y is nearer to the camera. All positions below are given as (X, Y, Z) relative to the parent. A **positive Pitch** tilts a part's +X axis toward +Z; for a bar whose long axis is Z it tilts the top of the bar toward **-X** (backward, for a shape facing +X).

### 3.2 `BP_Flyer`: structure

| New component | Type | Parent | Location | Notes |
|---|---|---|---|---|
| `Visual` | SceneComponent | `Root` | (0, 0, 0) | Carries the left/right mirroring. |
| `GroupBomber` | SceneComponent | `Visual` | (0, 0, 0) | Visibility switch of the bomber look. |
| `GroupSatellite` | SceneComponent | `Visual` | (0, 0, 0) | Visibility switch of the satellite look. |

The old cubes `BodyBomber` and `BodySatellite` stay where they are (children of `Root`) with `bVisible` set to **false**.

Every part below: relative rotation as stated (otherwise 0), collision `NoCollision`, `CastShadow` false, Movable, `bHiddenInGame` false, material as stated, parent as stated. The design faces **+X** (to the right); mirroring is done once, on `Visual` (3.4).

### 3.3 `BP_Flyer`: the parts

**Bomber** (material `MI_DD_Bomber`; parent `GroupBomber`; overall about 93 long x 35 tall, centred on the flyer's position)

| Component | Mesh | Target size (X x Z x depth Y) | Location (X, Y, Z) | Rotation | Scale if the mesh is 100 |
|---|---|---|---|---|---|
| `BomberBody` | Sphere | 92 x 14 x 8 | (0, 70, 0) | none | (0.92, 0.08, 0.14) |
| `BomberFin` | Cube | 8 x 22 x 8 | (-38, 70, 12) | Pitch +30 | (0.08, 0.08, 0.22) |
| `BomberWing` | Cube | 6 x 28 x 8 | (4, 69, -2) | Pitch +62 | (0.06, 0.08, 0.28) |
| `BomberGlow` | Plane | 150 x 65 | (0, 68.5, 4) | Roll 90 | (1.5, 0.65, 1.0) |

`BomberGlow` material: `MI_DD_HaloBomber` (Step 1). The glow's scale X is its length, Y its height (the plane lies on its side exactly as the missile halos do).

**Satellite** (material `MI_DD_Satellite`; parent `GroupSatellite`; overall about 72 wide x 38 tall, centred on the flyer's position)

| Component | Mesh | Target size (X x Z x depth Y) | Location (X, Y, Z) | Rotation | Scale if the mesh is 100 |
|---|---|---|---|---|---|
| `SatBody` | Cube | 20 x 20 x 8 | (0, 70, 0) | none | (0.20, 0.08, 0.20) |
| `SatPanelL` | Cube | 22 x 14 x 4 | (-25, 70, 0) | none | (0.22, 0.04, 0.14) |
| `SatPanelR` | Cube | 22 x 14 x 4 | (25, 70, 0) | none | (0.22, 0.04, 0.14) |
| `SatMast` | Cylinder | 2.5 wide x 14 tall x 2.5 deep | (0, 70, 17) | none | (0.025, 0.025, 14 / measured height) |
| `SatTip` | Sphere | 6 x 6 x 6 | (0, 70, 25) | none | (0.06, 0.06, 0.06) |
| `SatGlow` | Plane | 105 x 60 | (0, 68.5, 6) | Roll 90 | (1.05, 0.6, 1.0) |

`SatGlow` material: `MI_DD_HaloSatellite` (Step 1). The cylinder's long axis is its local Z, so it stands upright with no rotation (confirm with the Step 0 measurement; if the long axis is not Z, stop and report).

**Layering.** Every solid part is at most 8 deep and centred on world Y 69 to 70 (the same layer as the old cubes), and the glows are at 68.5, behind the front faces of the parts (front face at Y 73 to 74). Do not change the Y values.

**Arithmetic for the report.** Bomber: fuselage spans X -46 to 46; fin centre (-38, 12), height 22 pitched 30 degrees reaches about Z 23.5 and X -47 to -29; wing centre (4, -2), length 28 pitched 62 degrees spans about X -10 to 18 and Z -11 to 7; so the bomber is about 93 x 35 (the old cube was 100 x 30). Satellite: body +-10, panels from +-14 to +-36 (a 4-unit gap from the body), mast Z 10 to 24, tip Z 22 to 28; so the satellite is about 72 x 38 (the old diamond was about 71 across).

### 3.4 `BP_Flyer`: show/hide and mirroring (the only graph edits)
Today `Init` shows the right cube for the flyer's type (it contains `SetVisibility` calls on `BodyBomber` and `BodySatellite`; Step 0 records them exactly). The new look reuses that logic:

1. **Re-point.** Every `SetVisibility` call in `BP_Flyer` whose target is `BodyBomber` gets the target `GroupBomber`; every one whose target is `BodySatellite` gets `GroupSatellite`. On each re-pointed call, **`Propagate To Children` is TRUE** (so every part follows its group); the `New Visibility` wiring stays exactly as it was. (If Step 0 finds such a call anywhere else than in `Init`, for example in `Die`, re-point it the same way and report it.)
2. **Mirror.** Directly after the node in `Init` that sets `Dir`, insert `Visual.SetRelativeScale3D(NewScale3D = MakeVector(X = Dir, Y = 1.0, Z = 1.0))`. `Dir` is 1.0 for a flyer moving right and -1.0 for one moving left (Step 0 confirms the values it can take; if it can take any other value, stop and report). A scale X of -1 turns the whole silhouette around, so the bomber's nose points the way it flies. Insert the node by breaking the link `Set Dir.then -> NEXT`, wiring `Set Dir.then` to the new node and the new node's `then` to `NEXT` (when `NEXT` existed).

### 3.5 `BP_TapMarker` (the impact mark)
Today: `Root` with one component `Dot` (Sphere, location (0, 110, 0), scale 0.5, material `MI_DD_TapMarker`), default `Life` 1.0 (the marker destroys itself after `Life` seconds; `BP_ABM` normally removes it earlier through `Dismiss`).

| New component | Mesh | Target size | Location | Rotation | Scale if the mesh is 100 | Material |
|---|---|---|---|---|---|---|
| `BarA` | Cube | 64 x 6 x 6 | (0, 110, 0) | Pitch +45 | (0.64, 0.06, 0.06) | `MI_DD_AbmCore` |
| `BarB` | Cube | 64 x 6 x 6 | (0, 110, 0) | Pitch -45 | (0.64, 0.06, 0.06) | `MI_DD_AbmCore` |
| `MarkGlow` | Plane | 100 x 100 | (0, 106, 0) | Roll 90 | (1.0, 1.0, 1.0) | `MI_DD_HaloAbm` |

Both bars and the glow are children of `Root`. The two bars form an **X about 50 units across**, centred on the marker's position (which is the ABM's target). `Dot` gets `bVisible` false. The default of `Life` becomes **5.0** (a fallback only: your side-base ABMs fly at 1200 units per second, so the 1.0 second fallback ends the mark before the ABM arrives for targets farther than about 1200 units; the normal removal is unchanged).

**How the lifetime is applied (decided; Step 0 tells which case it is):**
- **(a)** `SetLifeSpan`'s `InLifespan` is fed from `Life` (a variable get, with or without the implicit double-to-float conversion): change **only the default** of `Life` to 5.0.
- **(b)** `InLifespan` is an **unwired literal** (for example 1.0) that does not read `Life`: set that one pin literal to **5.0** with `set_pin_value`, and also set the default of `Life` to 5.0 so the two stay in step. Read the node back. This is the only graph edit allowed in `BP_TapMarker`; nothing else in the graph changes.
- **(c)** `InLifespan` is fed from anything else, or there is no `SetLifeSpan` call, or the lifetime is controlled somewhere else: **do not change the lifetime at all** (leave `Life` at its current default), do every other part of this patch, and report the finding under *Open questions / risks* with the read-out. The human decides.

### 3.6 New material instances (Step 1)
`MI_DD_HaloBomber` and `MI_DD_HaloSatellite`: instances of `M_DD_Halo`, the glow colours of the two flyers. All parameter values are copied from `MI_DD_HaloAbm`, except the colour: `Color` is taken from `MI_DD_Bomber` and from `MI_DD_Satellite`. Compare the `Color` of `MI_DD_HaloAbm` with the `Color` of `MI_DD_AbmCore` channel by channel (R, G, B; alpha ignored), then use the matching case for the new `Color` values (decided, no other options):
- **(a) Equal:** copy the flyer material's `Color` unchanged.
- **(b) Same constant factor k in all three channels** (each halo channel = the core's channel x k): multiply the flyer material's R, G and B by that k (alpha unchanged).
- **(c) Any other relation** (different factors per channel, a different hue, or anything that is not one constant): copy the flyer material's `Color` unchanged.

Record in the report the two colours, the case (a, b or c) and the values used. The human tunes the glow colour by hand if wanted.

---

## 4. What this phase changes in earlier specs

| Earlier text | Now |
|---|---|
| Flyer bodies are two plain cubes (`BodyBomber`, `BodySatellite`) | Hidden; the look is the groups `GroupBomber` and `GroupSatellite` |
| `Init` shows or hides the cubes | The same calls act on the groups, with `Propagate To Children` true |
| A flyer always looks the same in both directions | `Visual` is mirrored with `Dir`, so the bomber faces its flight direction |
| The marker is a 50-unit ball that lives 1 second at most | A white-cyan X (about 50 across) with a glow; fallback lifetime 5 seconds |
| Frozen list | `BP_Flyer` and `BP_TapMarker` are unfrozen for the edits of this document only |

---

## 5. Tooling notes

1. Components on a Blueprint: `ActorTools.add_component` (engine meshes by path), then **`set_parent_component`**; after each re-parent **read back the parent and the relative transform** and correct it if the tool changed it. A SceneComponent has no mesh.
2. `StaticMeshTools.get_bounds` measures an engine mesh. `CastShadow` (not `bCastShadow`) is the shadow property.
3. Re-pointing a call's target: `create_node` a variable-get for the new component, `connect_pins` it to the call's `Target`, and remove the old link; read the call back. `set_pin_value` on a wired pin changes nothing.
4. Material instances: use the material tools you used in Phase 9c for `MI_DD_HaloEnemy`; read all parameters back.
5. `CaptureViewport` needs a camera transform and shows the **editor** world (use `(0, 3000, 900)`). **PIE cannot spawn actors**; nothing flies on the title screen. In the editor world `Init` does not run, so both flyer groups show wherever their defaults say.
6. If a tool reports that a property cannot be read or set, say so; do not guess.

---

## 6. Implementation steps

### Step 0 — Read and measure (no changes)
1. **Measure** `/Engine/BasicShapes/Cube`, `Sphere`, `Cylinder` and `Plane` (size at scale 1, bounds centre, and which local axis the cylinder's length lies along; expected Z). Record the values and any scale that you computed differently from section 3.3 and 3.5.
2. **`BP_Flyer`:** every component (parent, relative transform, mesh, material, `bVisible`, `bHiddenInGame`, `Mobility`, collision flag); every variable with its default; every function. Expected components: `DefaultSceneRoot`, `Root`, `BodyBomber` (Cube, (0, 70, 0), scale (1.0, 0.1, 0.3)), `BodySatellite` (Cube, (0, 70, 0), Pitch 45, scale (0.5, 0.1, 0.5)). **A new component name of section 3 already present, or `BodyBomber`/`BodySatellite` missing: stop and report (rule 20).**
3. **`Init`:** read it completely. Record (a) where `Dir` is set and which values it can take, (b) every `SetVisibility` call with its target, both boolean pins and the condition that drives them, (c) which node follows `Set Dir`. List **every node in every graph that references `BodyBomber`, `BodySatellite` or `Dir`**. Confirm that `IsInsideExplosion` calls `ContainsPoint` on the actor location.
4. **`BP_TapMarker`:** every component with details; the event graph and `Dismiss`; the variable `Life` and where `SetLifeSpan` is called, and **how its `InLifespan` pin is fed** (wired from `Life`, an unwired literal, or something else: this decides case (a), (b) or (c) of 3.5).
5. **`BP_ABM`:** where it stores the marker (`Marker`) and every call of `Dismiss` (which paths end an ABM's life, and whether each calls `Dismiss`). `BP_DDPlayerController`: where it spawns the marker and what it does with `LastMarker` (read only).
6. **Materials:** `Color` of `MI_DD_Bomber`, `MI_DD_Satellite`, `MI_DD_AbmCore`; all parameters of `MI_DD_HaloAbm`, `MI_DD_HaloEnemy` and `MI_DD_TapMarker`.

**Check:** the report section for Step 0 is complete.

### Step 1 — Glow materials
Create `MI_DD_HaloBomber` and `MI_DD_HaloSatellite` as described in 3.6. Read every parameter of both back.

**Fallback:** if instances with these parameters cannot be created, use `MI_DD_HaloEnemy` for both flyer glows and record why.

**Check:** both instances exist, parented to `M_DD_Halo`, with the required values. **Save All.**

### Step 2 — `BP_Flyer`
Read the Blueprint again first.
1. Set `bVisible` to **false** on `BodyBomber` and `BodySatellite`. Read back: mesh, material and transform unchanged.
2. Add `Visual`, `GroupBomber`, `GroupSatellite` (3.2). Read parents and transforms back.
3. Add the bomber parts and then the satellite parts (3.3) under their groups, with the materials and values of 3.3 (scales corrected by the Step 0 measurements). Apply rule V to the groups and parts. Read parent, transform, material and flags back for every part.
4. **Re-point** the `SetVisibility` calls (3.4.1). Read every changed call back.
5. **Mirror** (3.4.2). Read `Init` back.
6. Compile.

**Check:** `BP_Flyer` compiles clean; all parts read back as required; the old cubes are hidden and unchanged otherwise; `Init` reads back as before except for the re-pointed targets and the one new node; every item of rule 3 reads exactly as before; the tick settings are unchanged. **Save All.**

### Step 3 — `BP_TapMarker`
Read the Blueprint again first.
1. Set `bVisible` false on `Dot`. Read back.
2. Add `BarA`, `BarB`, `MarkGlow` (3.5) with the given materials and flags. Read back.
3. Apply the lifetime case of 3.5 (a, b or c). Read back the default of `Life` and, in case (b), the edited pin.
4. Compile.

**Check:** `BP_TapMarker` compiles clean; components and the default read back as required; its graphs and the other variables read exactly as before (except the one literal of case (b)). **Save All.**

### Step 4 — Calibration of shape and direction (recommended)
Goal: see the new shapes and confirm directions.
1. List the actors in `L_Game` (record the count). Place temporary actors: `TestBomber` (`BP_Flyer`) at (-300, 0, 900), `TestSat` (`BP_Flyer`) at (300, 0, 900), `TestMark` (`BP_TapMarker`) at (0, 0, 600).
2. In the editor world `Init` does not run. For each `BP_Flyer` instance, set `bVisible` false (instance only) on every part of the group that is not under test, so that `TestBomber` shows only the bomber and `TestSat` only the satellite.
3. **Capture A** (`CaptureViewport` from `(0, 3000, 900)`). Describe each object: the bomber's nose pointing **right**; the fin leaning **backward** (top toward the tail); the wing; the glow; the satellite's body, panels, mast and tip; the marker as an **X** with a glow. Estimate the sizes (bomber about 93 x 35, satellite about 72 x 38, X about 50).
4. **Capture B (mirror):** set `Visual` scale on the `TestBomber` instance to (-1, 1, 1): the bomber must point **left**, with the fin still leaning toward its tail.
5. **Fix rules:** if the fin or wing lean **forward** in capture A, negate their Pitch values (to -30 and -62) on the **Blueprint defaults**, recompile and repeat. If a glow is not centred on its flyer, or a part sits in front of the wrong layer, re-check the arithmetic of 3.3 against the Step 0 measurements. If the shapes are still not recognisable as a plane and a satellite, record it and stop; do not invent another shape.
6. **Revert:** delete all temporary actors, save the level, list the actors again: the count and the list **must equal** the one recorded in step 1 and `L_Game` must not be dirty. If anything cannot be reverted, **stop and report**.

**Fallback:** if actors cannot be placed or the capture is unclear, skip the step, leave the values as built and record why; the human checks the shapes in play (D1 to D3).

**Check:** shapes and directions recorded (and the fix applied if needed); the level is exactly as before. **Save All.**

### Step 5 — Run
1. **Save All.**
2. `StartPIE`. The title screen appears at once. **Wait about 20 seconds**; nobody taps, so no game starts and no flyer and no marker exists.
3. `CaptureEditorImage` at about 5 s and again at about 15 s. Describe each image (the title screen as in 9c; nothing else changed).
4. `GetLogEntries` (filter `[DD]`), then `StopPIE`.
5. Fill in the report (section 8).

---

## 7. Acceptance tests

### A. Static checks
- [ ] **A1** Changed assets: `BP_Flyer`, `BP_TapMarker` and the two new instances only. Nothing under `Spike`, no existing material touched, no Project Settings, input settings or `.ini` change. `L_Game` unchanged.
- [ ] **A2** `BP_Flyer` and `BP_TapMarker` compile with 0 errors and 0 warnings.
- [ ] **A3** Step 0 findings are in the report: mesh measurements and the formulas used, the full read-out of `Init` (including every `SetVisibility` call and the values `Dir` can take), the references to the cubes, the `Dismiss` paths of `BP_ABM`, the colours and parameters read.
- [ ] **A4** `MI_DD_HaloBomber` and `MI_DD_HaloSatellite` exist with the required parameter values.
- [ ] **A5** `BP_Flyer`: all components of 3.2 and 3.3 exist with the required parents, materials, locations, rotations and scales (or the measured-size variants, recorded); the old cubes are hidden and otherwise unchanged.
- [ ] **A6** `BP_TapMarker`: `BarA`, `BarB`, `MarkGlow` as in 3.5; `Dot` hidden; lifetime per case (a), (b) or (c) of 3.5 (default `Life` 5.0 in (a) and (b); unchanged and reported in (c)).
- [ ] **A7** `Init` and any other changed graph: every re-pointed call targets the right group with `Propagate To Children` true and its old `New Visibility` wiring; the mirror node sits directly after `Set Dir`; nothing else changed.
- [ ] **A8** **Visibility audit:** a table of every new or edited component with `bVisible`, `bHiddenInGame` and `Mobility`: `bHiddenInGame` false and Movable everywhere; `bVisible` per rule V (groups and parts equal the old cube's default), false on the two cubes and on `Dot`.
- [ ] **A9** **Pin audit (rule 14):** a table of every input pin of every re-pointed or new node (target, new visibility, propagate, new scale X/Y/Z, wiring).
- [ ] **A10** **Unchanged (rule 3):** a table that shows, for each item of rule 3, that it reads exactly as before, including the hit-test read-out (`ContainsPoint` on the actor location).
- [ ] **A11** Arithmetic of 3.3 is in the report (bomber and satellite extents, layers).
- [ ] **A12** Calibration (Step 4): the shapes and directions in captures A and B, and any fix; or the recorded fallback; if the step ran, `L_Game` has the same actors as before.

### B. Log checks (agent; the one 20 s run, nobody taps)
- [ ] **B1** The startup lines as in the 9c.2 run: camera, GameMode, config (`Width=2250.0 Height=1800.0`), bases 3, cities 6, PlayerController, view target, crosshair, HUD, `[DD][Area] Viewport=...`.
- [ ] **B2** Exactly one of `[DD][Save] Loaded high=<n>` or `[DD][Save] No file high=<n>`, followed by exactly one `[DD][Flow] Title high=<n>`.
- [ ] **B3** During the 20 s wait: no `Spawner started`, no `[DD][Wave]`, `[DD][Enemy]`, `[DD][Flyer]`, `[DD][Score]`, `[DD][Damage]`, `[DD][Input] Fire request`, `[DD][Aim]`, `[DD][Game]`, and no `[DD][Flow]` line except the one `Title` line.
- [ ] **B4** No line containing `ERROR`, `Accessed None`, `Divide by zero` or `Blueprint Runtime Error`.

### C. Visual checks (captures; describe in words)
- [ ] **C1** The title screen is unchanged from 9c. No new object appears.
- [ ] **C2** Step 4 captures (if run): as described in A12.

### D. Interactive checks (human, PIE) — all NEEDS-HUMAN
List every item as NEEDS-HUMAN in the report; do not run or claim them.
- [ ] **D1** **Bomber:** it reads as a small plane (fuselage, fin, wing) with a soft glow; its nose points the way it flies, to the right on left-to-right passes and to the left on right-to-left passes; no old cube is visible.
- [ ] **D2** **Satellite:** it reads as a satellite (body, two panels, mast with a tip) with a soft glow; no old diamond is visible.
- [ ] **D3** **Size and layering:** neither flyer looks out of proportion next to missiles and the smart bomb (bomber about 93 x 35, satellite about 72 x 38); missiles and explosions draw sensibly over and under them.
- [ ] **D4** **No gameplay change:** flyers move, shoot (missiles leave from the flyer's centre area), die and score exactly as before; a flyer dies when its **centre** is inside a blast (as before).
- [ ] **D5** **Marker:** tapping shows a white-cyan **X** with a soft glow exactly at the tap point; it stays until its ABM explodes there, **including the farthest targets from a side base** (flights longer than 1 second); it disappears when the ABM explodes; several ABMs in flight each have their own mark.
- [ ] **D6** **No leftovers:** no marker stays on screen after its ABM has exploded or after a game ends or restarts (report how long, if any); no old ball is visible.
- [ ] **D7** No gameplay change from the marker and no errors in the play log (`Blueprint Runtime Error`).
- [ ] **D8** A busy late wave with flyers and many marks runs without a hitch.

### D-DEVICE. Android (NEEDS-DEVICE)
- [ ] **T-1** On a 1080-pixel-high screen the bomber is about 56 x 21 pixels and still reads as a plane; the X mark is about 30 pixels across and visible under and around the finger.
- [ ] **T-2** The frame rate in a busy wave is not noticeably lower than before.

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped; the editor is on `L_Game`; nothing under `Spike` was touched; if Step 4 ran, the level is as before.

---

## 8. Report format (the agent's final message)

```
## Phase 9c.3 report (flyer look and impact mark)
**Engine version:** x.y.z
**MCP tools used:** (list)

### Step 0 findings
(mesh measurements and formulas; BP_Flyer components and defaults; full read-out of Init; references to the cubes and Dir; hit test; BP_TapMarker; Dismiss paths in BP_ABM; colours and parameters)

### Materials (Step 1)
(the two new instances with every parameter)

### Component changes (Steps 2 and 3)
(table: Blueprint | component | parent | mesh | location | rotation | scale | material | bVisible | bHiddenInGame | Mobility | collision flag)

### Graph changes (Step 2.4, 2.5)
(Init before and after; each re-pointed call with its pins; the mirror node with its pins)

### Pin audit (rule 14)
(table: function | node | pin | wired from / value)

### Hands-off audit (rule 3)
(table: item | unchanged? | how it was checked)

### Arithmetic (3.3)

### Calibration (Step 4)
(what captures A and B showed; the fix if any; level actor count before and after; or why it was skipped)

### Acceptance results
A1 PASS/FAIL ... through E1 PASS/FAIL/NEEDS-HUMAN/NEEDS-DEVICE

### Log excerpt
(all [DD] lines of the run, with timestamps)

### Screenshot notes
(paths of the captures and a description of each)

### Tool problems (errors, retries, editor restarts, writes that read back empty)
### Deviations
### Open questions / risks
```

---

## 9. What this phase hands to later phases

| Provided | How a later phase uses it |
|---|---|
| `Visual`, `GroupBomber`, `GroupSatellite` in `BP_Flyer` | Any later flyer look change is an edit of the parts inside a group; the mirroring and the show/hide stay as they are. |
| Part tables in 3.3 and 3.5 | Sizes are plain target sizes and scale values; the human can tune them by hand in the Blueprint editor. |
| Marker as an X with a glow | A later "closing ring" animation (the earlier option C) would add a ring part and a small animation to `BP_TapMarker`. |

**Known leftovers that stay (removed in 9g):** the hidden cubes `BodyBomber` and `BodySatellite`, the hidden `Dot` of `BP_TapMarker`, the hidden `Dot` of the missiles, the old `UpdateSmartTail` (uncalled), `StopSpawner`, `bDirectorSilent`, the two disconnected spawn chains in `SpawnRandomEnemy`, the disconnected older fire chain in `LaunchAt`, HUD debug lines 2 to 4, `BP_PlayfieldDebug`, the debug keys, the unused `MI_DD_Enemy` and `MI_DD_Abm`, the `Spike` folder.

---

## 10. Common pitfalls

1. **Leaving the old cubes visible.** `Init` used to show them; after the re-pointing nothing may show them. Their `bVisible` must be false and no call may target them.
2. **Forgetting `Propagate To Children` TRUE** on a re-pointed call: only the (empty) group would change and the parts would keep their defaults.
3. **Skipping rule V:** if the groups or parts get a different default than the old cubes had, a flyer type could start hidden or both could show.
4. **Mirroring with the wrong sign or value.** The scale X of `Visual` is exactly `Dir` (1.0 or -1.0). Scaling any other component, or the actor, would also mirror the hit test or the movement.
5. **Fin and wing leaning forward.** Positive Pitch leans the top of a vertical bar toward -X (backward). Calibration (Step 4) proves it.
6. **Assuming mesh sizes.** Measure (Step 0) and compute scales from the measured sizes (rule 12).
7. **Changing the Y values.** They keep the solid parts at 66 to 74 and the glows behind them; moving them breaks the layering.
8. **Touching the hit test, speeds, shots or scoring.** This patch is visual only.
9. **Rewriting `Init`** with the graph-writing tool (rule 10). Only the re-pointing and the one new node are allowed.
10. **Changing `Life` before reading the `Dismiss` paths of `BP_ABM`.** Record which paths remove the marker.
11. **Leaving the Step 4 actors in the level.** The actor count must match the count before.
12. **Marking D or T items PASS.** You cannot see a flyer or a marker in PIE. They are NEEDS-HUMAN and NEEDS-DEVICE.
13. **Touching `Spike`, the missiles, `BP_ABM` or any other frozen asset**, or cleaning up what this document does not list.
