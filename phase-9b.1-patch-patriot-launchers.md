# Phase 9b.1 — Patch: Patriot Launchers (art, desktop aim, tap snap)

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Depends on** | Phase 9b, revision 2 (accepted and committed) |
| **Phase goal** | Replace the dome-and-barrel **base art** with a **Patriot-style launcher** drawn in side profile: a wheeled chassis, a hinged launcher box with a cyan end cap, standing **straight up at rest** (the direction of today's barrel). **On desktop** (where the crosshair exists) the launcher of the base that would fire turns toward the crosshair. **On touch devices** the launchers never track; instead, at each shot, the firing launcher **snaps toward the tap** and eases back to vertical ("tap snap"). **The ABM now leaves from the launcher's muzzle.** Everything else (base selection, ammo, speeds, damage, scoring, waves) is unchanged. |

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **This patch is not purely visual.** The only gameplay-relevant change is the **ABM start point** (3.6). Base selection (`SelectBase`, `ConsiderBase`), `LaunchPos`, speeds, ammo, damage, scoring, waves and flow stay exactly as they are.
3. **Existing logic must survive** unless a step says to change it. Read each graph (`read_graph_dsl`) before editing it.
4. **Compile after every Blueprint change and recompile every material.** Every Blueprint ends with no errors and no warnings.
5. **Run each step's "Check"** before moving on. **Save All after each step's Check passes.**
6. **You cannot press keys, click, touch, move the mouse, or resize in PIE, and you cannot start a game.** You can verify: compiles, static read-backs of every component, variable, function and graph, hand-computed geometry (Step 5), and **the title screen in PIE** (the three bases are spawned at startup and stand at rest). You **cannot** see the launcher track the crosshair, snap, or fire. Those are group D (**NEEDS-HUMAN**); device checks are **NEEDS-DEVICE**. Never mark a D or T item PASS.
7. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
8. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after the run. **Save before loading a level.** **Do not touch Project Settings, input settings or any `Config/*.ini` file.** **Do not use, edit or delete anything in `/Game/DomeDefense/Spike/`.**
9. **NEVER REWRITE AN EXISTING FUNCTION OR EVENT GRAPH** with `write_graph_dsl`. A **brand-new** function is written **once**, in one small write. **Existing graphs are changed only by small node or pin edits** (`create_node`, `connect_pins`, `set_pin_value`).
10. **READ EVERY NEW OR CHANGED GRAPH BACK** right after writing it. If a body reads back empty or incomplete: write it once more; if it is still wrong, **stop and report** under *Tool problems*.
11. **VISIBILITY AUDIT.** Every new visual component must have `bVisible` true and `bHiddenInGame` FALSE (the lesson of the invisible smart bomb). Read both flags back for **every** new component and list them in the report. No component starts hidden. The existing carriers `Body` and `Rubble` keep their 9b values (read back, do not change).
12. **MOBILITY.** `Launcher` and every component under it must be **Movable** (it rotates at run time). A Static component cannot be rotated in a game. Read `Mobility` back for each.
13. **CHECK EVERY CALL YOU BUILD for a boolean or number pin left at its default.** Read the input pins back and list them in the report (the list is in Step 4 and in A8).
14. **After any tool error, do not retry the same large operation.** Stop that item, try one smaller piece at most, and if it still fails **stop and report**. **If the editor stops responding or restarts, stop and report.** Do not continue with a half-edited asset.
15. **Order is part of the spec:** read and measure first, then config and material, then the base art, then the base logic, then the controller, then the static proof, then the run.
16. **FROZEN: do not edit** `BP_DDGameMode` and every function in it, `BP_DDHUD`, `BP_EnemyMissile`, `BP_Flyer`, `BP_ABM`, `BP_Explosion`, `BP_Crosshair`, `BP_TapMarker`, `BP_PlayfieldCamera`, `BP_PlayfieldDebug`, `BP_City`, `BP_Backdrop`, `BP_DDSave`, `BFL_DDDebug`, the `Ground` actor, every material and material instance except the one new instance named in Step 1, and `L_Game` (except the optional, fully reverted proof in Step 5b). **The only edits to existing assets are listed in section 4.**
17. **No new keys. No new Tick on any base** (`BP_MissileBase` keeps Start With Tick Enabled = false). **The only new log line** is `[DD][Aim] Snap ...` inside `LaunchAt` (3.6).
18. **Node IDs with parentheses cannot be written in the graph DSL** (`Clamp`, `Abs`, `Max`, `Min`, `Percent`, array `Get`, `FInterpConstantTo`, `Atan2 (Degrees)`). Create those nodes with `create_node` and connect them with `connect_pins`. Use `Select` nodes or a `Branch` where that is simpler. **Guard every division with a Branch** (this patch has none).
19. **ANGLE SIGN (decided; do not flip it elsewhere).** `AimAngle` is in degrees, **0 = straight up, positive = toward +X (screen right)**. The component rotation is `Make Rotator(Roll = 0, Pitch = -AimAngle, Yaw = 0)`: in Unreal a positive pitch tips an up-pointing axis toward -X, so the sign is negated. If the sign proves wrong in Step 5, fix **only** that one negation and record it.
20. **Do not clean up anything this document does not list.**

---

## 2. Scope

### In scope
- Three new config values on `PDA_DDConfig` (3.7) and one new material instance `MI_DD_BaseDark`.
- `BP_MissileBase`: delete the four old art parts, add the launcher art (3.3), two variables, five new functions (Step 3).
- `BP_DDPlayerController`: one new function `UpdateLaunchers`, one call at the start of `Event Tick`, the `LaunchAt` edits (3.6).

### Out of scope (do NOT do)
- Which base is selected, its speed, its ammo, `LaunchPos`, `SelectBase`, `ConsiderBase`, `RequestFire`, `Demolish`, `Rebuild`, `ApplyVisual`, `InitBase`, `AmmoText`.
- The rubble art of the bases, cities, the backdrop, HUD, crosshair, tap marker, ABM art, explosions, trails, muzzle flash, smoke, recoil, sound, post-processing.
- Any key, any Project or Android setting, any texture import, Niagara, Paper2D.
- Per-wave palettes.

---

## 3. Design (decided; do not change)

### 3.1 Behaviour

| Situation | Launcher behaviour |
|---|---|
| **Desktop** (the controller's `Crosshair` is valid), `FlowState == 1`, not game over | The base that `SelectBase(CrosshairWorldPos)` returns (nearest base with ammo, the rule that decides the real shot) turns toward the crosshair. The other two bases stand vertical. |
| Desktop, title (`FlowState 0`), paused (`2`) or game over | All three stand vertical. |
| Desktop, no base has ammo | All three stand vertical. |
| Desktop, a shot | The firing launcher **snaps to the exact target angle** at that moment; the ABM leaves from the muzzle along the barrel. |
| **Touch / no crosshair**, any time | Launchers stand vertical. On each shot the firing launcher **snaps to the tap angle**, holds it `LauncherSnapHold` seconds, then eases back to vertical at `LauncherTurnSpeed`. |
| A destroyed base | Its art is hidden with `Body` (no change). It has no ammo, so it is never selected and its angle eases to 0. |

"Desktop" means **the crosshair exists**. Phase 1.1 already spawns it only when `bDesktopCrosshair` is true and the platform is not Android or iOS. **Do not add a second platform test.** (Setting `bDesktopCrosshair` to false in the controller defaults therefore also gives the touch behaviour on a PC; the human uses this to test it.)

### 3.2 Geometry and conventions

All positions below are relative to the base's `Body` carrier (location (0, 60, 0) on the actor, scale 1, set in 9b). The actor origin is at world Z = 100 (the ground top), so **carrier Z 70 = world Z 170**.

- **Hinge:** the scene component `Launcher` at (0, 0, 30). It is also the rotation pivot.
- **Muzzle:** the scene component `Muzzle` under `Launcher` at local (0, 0, 40). At rest it is at carrier (0, 0, 70): **world X = the actor's X, world Z = 170 = `LaunchPos.Z`.** The launch point does not move at rest.
- **Rotation:** about the world Y axis only (the play plane is XZ, the camera looks along -Y). `Launcher.SetRelativeRotation(Make Rotator(Roll 0, Pitch -AimAngle, Yaw 0))`.
- Muzzle offset from the actor origin for an angle `A` (degrees, rule 19): `(40 sin A, 60, 30 + 40 cos A)`.

| `AimAngle` | Muzzle, actor-relative (X, Y, Z) | World Z |
|---|---|---|
| 0 | (0, 60, 70) | 170 |
| +45 | (28.28, 60, 58.28) | 158.28 |
| +90 | (40, 60, 30) | 130 |
| -45 | (-28.28, 60, 58.28) | 158.28 |
| -90 | (-40, 60, 30) | 130 |

- **Limits:** `AimAngle` stays within `-LauncherMaxAngle` to `+LauncherMaxAngle` (90). A launcher never points below the horizontal.
- **Front-face rule (as in 9b):** no part may have its front face beyond carrier Y + 8. Enemy missiles (world Y 70) must always be drawn in front of the launcher. Rotation about Y does not change any Y extent.
- **Footprint:** at most 100 wide (X -50 to +50) at every angle between -90 and +90. Nothing below carrier Z 0 at any angle.

### 3.3 Art (sizes are target world sizes at rest, X × Y × Z; relative scale = size / 100 for the cube and cylinder meshes, which Step 0 re-confirms as 100 × 100 × 100 with centred pivots; `CastShadow` false and collision `NoCollision` on every part)

**Static parts** (attach under `Body`):

| Part | Mesh | Size | Centre | Material |
|---|---|---|---|---|
| `Chassis` | Cube | 92 × 10 × 8 | (0, 0, 16) | `MI_DD_BaseBody` |
| `Wheel1` | Cylinder, roll 90, scale (0.14, 0.14, 0.08) | 14 × 8 × 14 | (-33, 0, 7) | `MI_DD_BaseDark` |
| `Wheel2` | same | 14 × 8 × 14 | (-11, 0, 7) | `MI_DD_BaseDark` |
| `Wheel3` | same | 14 × 8 × 14 | (11, 0, 7) | `MI_DD_BaseDark` |
| `Wheel4` | same | 14 × 8 × 14 | (33, 0, 7) | `MI_DD_BaseDark` |
| `Cradle` | Cube | 26 × 8 × 12 | (0, -1, 26) | `MI_DD_BaseDark` |
| `Jack1` | Cube | 5 × 6 × 12 | (-46.5, 0, 6) | `MI_DD_BaseDark` |
| `Jack2` | Cube | 5 × 6 × 12 | (46.5, 0, 6) | `MI_DD_BaseDark` |

The wheel cylinders are rolled 90 about X so that their axis lies along Y (the mesh's diameter axes become world X and Z, its height becomes world Y). Check the resulting bounding box against the size column and record it.

**Rotating assembly** (`Launcher` under `Body`; everything below is under `Launcher`; centres are in the `Launcher`'s local frame, local Z is the barrel axis):

| Part | Mesh | Size | Centre | Material |
|---|---|---|---|---|
| `Launcher` | SceneComponent | none | (0, 0, 30) relative to `Body` | none |
| `Box` | Cube | 20 × 10 × 40 | (0, 1, 14) | `MI_DD_BaseBody` |
| `Cap` | Cube | 22 × 11 × 6 | (0, 1, 37) | `MI_DD_BaseAccent` |
| `Band1` | Cube | 21 × 10.5 × 3 | (0, 1, 6) | `MI_DD_BaseAccent` |
| `Band2` | Cube | 21 × 10.5 × 3 | (0, 1, 24) | `MI_DD_BaseAccent` |
| `Seam` | Cube | 2 × 11 × 30 | (0, 1, 15) | `MI_DD_BaseDark` |
| `Muzzle` | SceneComponent | none | (0, 0, 40) | none |

The box spans local Z -6 to 34, the cap 34 to 40: the tip is at local Z 40, the `Muzzle`. Front faces (carrier Y): chassis +5, wheels +4, cradle +3, jacks +3, box +6, bands +6.25, cap +6.5, seam +6.5; all are at most +8 and no two overlapping faces are coplanar.

**Deleted:** `Podium`, `Dome`, `Barrel`, `Accent` (the 9b base art). **Kept as they are:** `Body`, `Rubble`, `Rub1` to `Rub4`, `Ember1`, `AmmoText`.

**Sketch (rest pose, side view; illustrative only):**
```
            ┌────┐   Cap (cyan)  <- Muzzle, Z 70
            │▒▒▒▒│   Band2 (cyan)
            │█║██│   Box (steel) + Seam (dark)
            │▒▒▒▒│   Band1 (cyan)
            │████│
      ──────┴─╥──┴──  hinge `Launcher`, Z 30, on the Cradle
     ┌─────────────┐   Chassis, Z 12..20
   ┌─┘ ◉  ◉   ◉  ◉ └─┐ wheels Z 0..14, jacks at both ends
```

New material instance `MI_DD_BaseDark` [`M_DD_Unlit`]: `Color` (0.12, 0.14, 0.17). `MI_DD_BaseBody` and `MI_DD_BaseAccent` are reused unchanged.

### 3.4 Aim angle

For a base with the `Launcher` at world position `p` and a target `T` (world XZ):
```
dx = T.X - p.X
dz = T.Z - p.Z
a  = Atan2Degrees(dx, dz)        // first argument = dx, second = dz; 0 = up, + = toward +X
angle = Clamp(a, -MaxAngle, +MaxAngle)
```
`Atan2Degrees(0, 0)` returns 0. There is no division. A target below the hinge (`dz < 0`) gives |a| above 90 and is clamped to ±90.

### 3.5 Motion and snap

- `StepAim(Dt, Desired, Speed)` (per frame, called by the controller): if `AimHold > 0`, count it down and stop; otherwise move `AimAngle` toward `Desired` by at most `Speed * Dt` degrees (constant speed, no overshoot) and re-apply the rotation only when the angle changed.
- `SnapAim(Angle, Hold)`: set `AimAngle = Angle` and `AimHold = Hold`, apply the rotation **at once**.
- A swing from +90 to -90 passes through vertical; no angle wrap is needed.

### 3.6 Selection, shot, ABM start

- **Selection is unchanged.** `SelectBase` measures from `LaunchPos` (the fixed point actor + (0, 0, 70)) and `LaunchPos` is never changed. The controller calls the same `SelectBase` each frame (desktop) to know which launcher turns, so **the launcher that turns is always the one that fires**.
- **In `LaunchAt`**, after the base is chosen and before the ABM is spawned:
  ```
  a     = Base.ComputeAim(Target, Cfg.LauncherMaxAngle)                // store
  hold  = Select(IsValid(Crosshair), Option 1 = 0.0, Option 0 = Cfg.LauncherSnapHold)   // store
  Base.SnapAim(a, hold)
  start = Base.GetMuzzleStart()                                         // store; AFTER SnapAim
  DDLog("Snap base=" + GM.GetBaseName(Idx) + " deg=" + Round(a), "Aim", false)
  ```
  The ABM is then spawned at `start` and `Abm.Init(start, Target, Speed, Marker, Idx)` gets `start` as its first argument. **Both** places that read `Base.LaunchPos` for the ABM become `start`. Nothing else in `LaunchAt` changes (`FireCount`, marker, speed choice, `ConsumeAmmo`, the `Fire request` log).
- **Start point:** `start = (Muzzle world X, LaunchPos.Y, Muzzle world Z)`. The Y stays the play-plane Y of `LaunchPos` (the ABM actor sits at that Y and draws itself +80 in front).
- **Accepted quirk:** a target within about 40 units of the hinge is behind the muzzle; the ABM then flies a few units backward. It was already so for targets below the old launch point.
- With the launcher at rest (touch, between shots) the start point is the old `LaunchPos` X and Z, so the touch flight differs from today only by the snap.

### 3.7 Config (new values on `PDA_DDConfig`; Instance Editable; category `Launcher`; these are the class defaults)

| Name | Type | Default |
|---|---|---|
| `LauncherTurnSpeed` | float | 720.0 (degrees per second) |
| `LauncherMaxAngle` | float | 90.0 (degrees) |
| `LauncherSnapHold` | float | 0.25 (seconds; used only when there is no crosshair) |

### 3.8 Budget
15 new components in `BP_MissileBase` (8 static, `Launcher`, 5 launcher parts, `Muzzle`) and 4 deleted, so +11 in the Blueprint, 45 in the world. Shared material instances only. No translucency, no per-actor tick, no per-frame allocation. The only per-frame logic is `UpdateLaunchers` (three bases).

---

## 4. What this patch changes in earlier specs

| Earlier text | Now |
|---|---|
| 9b base art: `Podium`, `Dome`, `Barrel`, `Accent` | Deleted; replaced by the parts of 3.3. The 9b rule "barrel tip at actor-relative Z 70" is now "`Muzzle` at carrier Z 70 at rest". |
| Phase 2 `BP_MissileBase` (Start With Tick Enabled = false; variables `BaseIndex`, `Ammo`, `LaunchPos`) | Unchanged, plus variables `AimAngle`, `AimHold`; functions `ApplyAim`, `SnapAim`, `ComputeAim`, `StepAim`, `GetMuzzleStart`; the `Launcher` and `Muzzle` scene components. `LaunchPos` is not written anywhere new. |
| Phase 1 `BP_DDPlayerController.Event Tick` | One call `UpdateLaunchers(Delta Seconds)` is inserted as the **first** node (before the early-return branch), so it also runs when there is no crosshair. Nothing else in `Event Tick` changes. |
| Phase 2 `LaunchAt` | The aim block of 3.6 is inserted before the ABM spawn; the two `Base.LaunchPos` reads for the ABM (spawn location, `Init` start) become `start`; one new log line. Everything else is unchanged. |
| Phase 2 / 5 `SelectBase`, `ConsiderBase`, `RequestFire` | Unchanged. |
| `PDA_DDConfig` | Three new values (3.7). |
| 9b FROZEN list | `BP_MissileBase`, `BP_DDPlayerController` and `PDA_DDConfig` are unfrozen **for the edits of this document only**. |
| Log categories | New category string `Aim` (just a string). |

---

## 5. Tooling notes (from the 9a, 9b and 8c reports)

1. Components on a Blueprint: `ActorTools.add_component` (engine meshes by path, e.g. `/Engine/BasicShapes/Cube`), then **`set_parent_component`** to attach. After each re-parent **read back the parent and the relative transform** and correct the location if it changed. Delete a component with the matching delete tool; record the tool used.
2. `StaticMeshTools.get_bounds` measures an engine mesh. `CastShadow` (not `bCastShadow`) is the shadow property. `bodyInstance.collisionEnabled` = `NoCollision`.
3. Material instances: `MaterialInstanceTools.create`, `set_parent`, set vector parameter `Color`, `list_parameters` to read back.
4. New graph functions: `BlueprintTools.create` and one small `write_graph_dsl` per **new** function; `read_graph_dsl` right after. Node IDs with parentheses (rule 18) come from `create_node`. The degrees version of Atan2 is found by a node search for "Atan2"; use the one that returns degrees. If no degrees version exists, use the radians one and multiply by 57.29578 (record it).
5. `FInterpConstantTo` (inputs Current, Target, Delta Time, Interp Speed) is the preferred mover. **Fallback (manual):** `diff = Desired - AimAngle`; `step = Speed * Dt`; `move = Clamp(diff, -step, +step)`; `na = AimAngle + move`.
6. `Make Rotator` has the pins Roll, Pitch, Yaw. Wire **Pitch** only; leave Roll and Yaw at 0 and read them back.
7. **PIE cannot spawn actors**; the title screen already shows the three bases. `CaptureEditorImage` captures the PIE title screen. `CaptureViewport` needs a camera transform and shows the **editor** world; use `(0, 3000, 900)` as in 9b.
8. Store every value in a variable or local before something it depends on changes. Loop bounds are stored too. `For Each Loop` over an empty array runs zero times, which is why it is used for `GM.Bases`.
9. The controller's cached references are `GM` (game mode) and `Cfg` (config); `Crosshair` is valid only on desktop; `CrosshairWorldPos` is the clamped cursor position on the play plane (Phase 1 / 1.1).
10. If a tool reports that a property cannot be read or set, say so; do not guess.

---

## 6. Implementation steps

### Step 0 — Read and measure (no changes)
1. **`BP_MissileBase`:** every component with its **parent**, relative transform, mesh, material, `bVisible`, `bHiddenInGame`, `Mobility`; the variables; every function graph; Start With Tick Enabled (expected false). Confirm that `Podium`, `Dome`, `Barrel`, `Accent` exist and that **no function reads or writes them** (search the graphs). If one does, stop and report.
2. **`BP_DDPlayerController`:** the variables `GM`, `Cfg`, `Crosshair`, `CrosshairWorldPos`, `bDesktopCrosshair`; the **`Event Tick`** graph (name its first node and the first branch, and the name of the delta-seconds output); **`LaunchAt`** (the node that sets `Base`, the ABM spawn node, the `Abm.Init` call, **both** `LaunchPos` reads, the `Fire request` log); `SelectBase` and `ConsiderBase` (read only).
3. **`PDA_DDConfig`:** its variables (confirm that `LauncherTurnSpeed`, `LauncherMaxAngle`, `LauncherSnapHold` do not exist) and how `DA_DDConfig` stores values.
4. **`BP_DDGameMode`** (read only): `Bases` (array of `BP_MissileBase`), `FlowState`, `bGameOver`, `GetBaseName`.
5. **Measure** `/Engine/BasicShapes/Cube` and `/Engine/BasicShapes/Cylinder` (size, pivot at the centre?). 9b measured both as 100 × 100 × 100 with centre pivots; confirm. Where a pivot is not at the centre, offset the location so the visual centre lands on the table value and list the offset.
6. Read `MI_DD_BaseBody`, `MI_DD_BaseAccent` and `M_DD_Unlit` (parameter names and values).

**Check:** the report section for Step 0 is complete (components with parents and flags, the Tick and `LaunchAt` node names, the config names, mesh sizes).

### Step 1 — Config and material
1. `PDA_DDConfig`: add `LauncherTurnSpeed` (720.0), `LauncherMaxAngle` (90.0), `LauncherSnapHold` (0.25): float, Instance Editable, category `Launcher`. Compile. Read the three values back **from `DA_DDConfig`**; if the instance does not show the defaults, set them on the instance.
2. Create `MI_DD_BaseDark` in `/Game/DomeDefense/Materials/` [`M_DD_Unlit`], `Color` (0.12, 0.14, 0.17, 1). Read it back.

**Check:** `PDA_DDConfig` compiles clean; the three values read back from `DA_DDConfig`; `MI_DD_BaseDark` reads back. **Save All.**

### Step 2 — `BP_MissileBase` art
Read the Blueprint again first.
1. **Delete** `Podium`, `Dome`, `Barrel`, `Accent`.
2. **Add the static parts** (3.3) and attach each under `Body`: `Chassis`, `Wheel1` to `Wheel4`, `Cradle`, `Jack1`, `Jack2`.
3. **Add `Launcher`** (SceneComponent) under `Body` at (0, 0, 30). **Add `Box`, `Cap`, `Band1`, `Band2`, `Seam`** under `Launcher`, and the SceneComponent **`Muzzle`** under `Launcher` at (0, 0, 40).
4. For every new component: collision disabled (meshes), `CastShadow` false (meshes), `bVisible` true, **`bHiddenInGame` false**, **`Mobility` Movable** (all new components; at minimum `Launcher` and everything under it). Read parent, relative location, rotation, scale, material and the three flags back for **every** one.
5. Compute by hand, for the report: the **bounding box of each wheel** after the roll (must equal 14 × 8 × 14); the `Muzzle` position at rest (carrier (0, 0, 70)); the tallest and widest extent of the whole base at rest.
6. Compile.

**Check:** 15 new components present with the sizes, centres and materials of 3.3; the four old parts are gone; every flag as in rules 11 and 12; the muzzle at rest equals actor-relative (0, 60, 70); `BP_MissileBase` compiles clean and `Demolish`, `Rebuild`, `ApplyVisual`, `InitBase`, `SetAmmo`, `SetAmmoDisplay`, `ConsumeAmmo`, `GetAmmoDisplayText`, `GetBodyShown`, `GetRubbleShown` are unchanged. **Save All.**

### Step 3 — `BP_MissileBase` aim logic
Variables (not Instance Editable): `AimAngle` (float, 0.0), `AimHold` (float, 0.0).

Create the functions in this order. **Each is a new function: write it once, then read it back.** Store every value in a local before use.

**`ApplyAim`** (no inputs, non-Pure):
```
Launcher.SetRelativeRotation(NewRotation = Make Rotator(Roll = 0, Pitch = -AimAngle, Yaw = 0), bSweep = false)
```
(`-AimAngle` is `AimAngle * -1`.)

**`SnapAim`** (inputs `Angle` float, `Hold` float):
```
AimAngle = Angle
AimHold  = Max(Hold, 0.0)          // Max via create_node, or a Branch
ApplyAim()
```

**`ComputeAim`** (inputs `Target` vector, `MaxAngle` float; output `Angle` float):
```
p  = Launcher.GetWorldLocation                      // store
dx = Target.X - p.X                                 // store
dz = Target.Z - p.Z                                 // store
a  = Atan2Degrees(dx, dz)                           // store; first argument dx, second dz
Angle = Clamp(a, -MaxAngle, MaxAngle)               // create_node for Clamp; Min = MaxAngle * -1, Max = MaxAngle
```

**`StepAim`** (inputs `Dt` float, `Desired` float, `Speed` float):
```
If AimHold > 0.0:
    AimHold = AimHold - Dt
    return
na = FInterpConstantTo(AimAngle, Desired, Dt, Speed)        // store (or the manual fallback of tooling note 5)
d  = Abs(na - AimAngle)                                     // store
If d > 0.01:
    AimAngle = na
    ApplyAim()
```

**`GetMuzzleStart`** (no inputs; output `Start` vector):
```
m = Muzzle.GetWorldLocation                                 // store
Start = Make Vector(X = m.X, Y = LaunchPos.Y, Z = m.Z)
```

Compile.

**Check:** all five functions read back non-empty and as above; the two variables exist with default 0; `Make Rotator` reads **Roll 0, Pitch = -AimAngle, Yaw 0**; `SetRelativeRotation` `bSweep` false; `BP_MissileBase` compiles clean with Start With Tick Enabled still **false**. **Save All.**

### Step 4 — `BP_DDPlayerController`
Read the Blueprint again first (Step 0.2 names).

**4a. New function `UpdateLaunchers`** (input `Dt` float; non-Pure). **Write it once; read it back.**
```
If NOT IsValid(GM) OR NOT IsValid(Cfg) → return
desk = IsValid(Crosshair)                                     // store
go   = GM.bGameOver                                           // store
fs   = GM.FlowState                                           // store
nb   = Length(GM.Bases)                                       // store
playing = (fs == 1) AND NOT go                                // store
sel  = -1                                                     // local, default -1
If desk AND playing AND nb >= 3:
    sel = SelectBase(CrosshairWorldPos)
For Each Loop over GM.Bases (Array Element B, Array Index i):
    If IsValid(B):
        If i == sel:   desired = B.ComputeAim(CrosshairWorldPos, Cfg.LauncherMaxAngle)
        Else:          desired = 0.0
        B.StepAim(Dt, desired, Cfg.LauncherTurnSpeed)
```
`SelectBase` is the existing function and keeps its side effects on `SelBest` and `SelBestDist`; that is fine. `desired` is a local float assigned in both branches; the two branches join before `StepAim`.

**4b. `Event Tick`:** insert **`UpdateLaunchers(Delta Seconds)` as the first node**, before the existing early-return branch (re-route the exec wire through it). Read the graph back: the call is first and everything after it is unchanged.

**4c. `LaunchAt`:** with small node and pin edits only (rule 9):
1. Insert the aim block of 3.6 (`a`, `hold`, `SnapAim`, `start`, the `Snap` log) **after** the node that sets `Base` and **before** the ABM spawn node. The `hold` value is a `Select` (float): Option 1 (condition true) = 0.0, Option 0 = `Cfg.LauncherSnapHold`, selected by `IsValid(Crosshair)`.
2. Connect `start` to the ABM spawn node's **Location** pin and to the `Abm.Init` node's **first (start position) argument**. Remove the two `Base.LaunchPos` links; nothing else reads `LaunchPos` in this function afterwards (confirm).
3. Read `LaunchAt` back: the `Fire request` log, `FireCount`, the marker spawn, the speed choice, `ConsumeAmmo` and the return values are unchanged.

Compile.

**Check:** `BP_DDPlayerController` compiles clean; `UpdateLaunchers` exists as above; the Tick graph starts with it; `LaunchAt` as above; `SelectBase`, `ConsiderBase`, `RequestFire` are unchanged; the key and input events are unchanged. **Save All.**

### Step 5 — Static proof (no gameplay state)
**5a. By hand (required).** Using the 3.2 formula, compute and put in the report a table for `AimAngle` = 0, +45, +90, -45, -90: the `Muzzle` actor-relative position, the lowest point of the launcher box, and the widest X of the whole base. Compare the rest row with the component values you read back in Step 2. State the rotation sign you rely on (rule 19).

**5b. Optional placed-instance proof.** Only if it takes a few calls; otherwise skip and record why.
1. List the actors in `L_Game` (record the count). Place **one** temporary `BP_MissileBase` at (0, 0, 100) with `SceneTools.add_to_scene_from_asset`.
2. On that instance set `Launcher`'s relative rotation to pitch **-45**, read the `Muzzle` **world** location (expected X about +28.3, Z about 158.3), then pitch **+45** (expected X about -28.3), and a pitch of **-90** (expected X +40, Z 130). Capture each with `CaptureViewport` from `(0, 3000, 900)` and describe what you see (which way the launcher leans).
3. If the sign is the opposite, apply rule 19 (one negation in `ApplyAim`) and record it.
4. **Revert:** delete the temporary actor, save the level, list the actors again: the count and the list **must equal** the one recorded in step 1 and `L_Game` must not be dirty. If anything cannot be reverted, **stop and report**.

**Check:** 5a table complete. If 5b ran: the level is exactly as before. **Save All.**

### Step 6 — Run
1. **Save All.**
2. `StartPIE`. The title screen appears at once. **Wait about 20 seconds**; nobody taps, so no game starts.
3. `CaptureEditorImage` at about 5 s and again at about 15 s. Describe each image in words: the three launchers at rest (wheels, chassis, box with cyan bands, cyan cap at the top), the ammo numbers, the six cities, the backdrop, the title text.
4. `GetLogEntries` (filter `[DD]`), then `StopPIE`.
5. Fill in the report (section 8).

---

## 7. Acceptance tests

### A. Static checks
- [ ] **A1** New: `MI_DD_BaseDark`. Changed: `PDA_DDConfig` (and `DA_DDConfig` only if Step 1 required it), `BP_MissileBase`, `BP_DDPlayerController`. Nothing else, nothing under `Spike`, no Project Settings, input settings or `.ini` change. `L_Game` is unchanged (or exactly as before if 5b ran).
- [ ] **A2** `BP_MissileBase`, `BP_DDPlayerController` and `PDA_DDConfig` compile with 0 errors and 0 warnings; `BP_MissileBase` still has Start With Tick Enabled = false.
- [ ] **A3** `LauncherTurnSpeed` 720.0, `LauncherMaxAngle` 90.0, `LauncherSnapHold` 0.25 read back from `DA_DDConfig`.
- [ ] **A4** `BP_MissileBase` has exactly the 15 new components of 3.3 (names, parents, sizes, centres, materials; the wheel bounding box 14 × 8 × 14); `Podium`, `Dome`, `Barrel`, `Accent` are gone; `Body`, `Rubble`, `Rub1` to `Rub4`, `Ember1`, `AmmoText` are untouched.
- [ ] **A5** Geometry: `Muzzle` at rest is actor-relative (0, 60, 70) (world X = actor X, world Z = `LaunchPos.Z`); no front face beyond carrier Y + 8; footprint at most 100 wide and nothing below carrier Z 0 at every angle in [-90, +90] (from the 5a table).
- [ ] **A6** `BP_MissileBase` functions `ApplyAim`, `SnapAim`, `ComputeAim`, `StepAim`, `GetMuzzleStart` and variables `AimAngle`, `AimHold` read back as in Step 3.
- [ ] **A7** `BP_DDPlayerController`: `UpdateLaunchers` reads back as in 4a; `Event Tick` starts with it; `LaunchAt` reads back as in 4c (the two `LaunchPos` reads replaced by `start`; one new `Aim` log line; the rest unchanged); `SelectBase`, `ConsiderBase`, `RequestFire` unchanged.
- [ ] **A8** **Call audit (rule 13):** a table of every pin below with its value or source: `Make Rotator` (Roll, Pitch, Yaw); `SetRelativeRotation` (`bSweep`); `SnapAim` (`Angle`, `Hold`); `StepAim` (`Dt`, `Desired`, `Speed`) in `UpdateLaunchers`; `ComputeAim` (`Target`, `MaxAngle`) in both callers; `Clamp` (Min, Max); the `Atan2` arguments (dx first, dz second); `FInterpConstantTo` (or the manual step); the `hold` `Select` (Option 1 and Option 0); the ABM spawn **Location**; `Abm.Init` first argument; `For Each Loop` array; the `Branch` conditions of `UpdateLaunchers`.
- [ ] **A9** Unchanged: every asset named in rule 16 (except as listed); `AmmoText` is still written only inside `SetAmmoDisplay` and keeps its world position; `LaunchPos` is still written only in `InitBase`; `Demolish`, `Rebuild`, `ApplyVisual`, `InitBase`, `GetBodyShown`, `GetRubbleShown` unchanged.
- [ ] **A10** **Visibility audit:** a table of every new component with `bVisible`, `bHiddenInGame`, `Mobility`: all true, false, Movable. The carriers `Body` and `Rubble` read as in 9b.
- [ ] **A11** The 5a table is present and matches the formula; the rotation sign is stated (and 5b results if it ran).
- [ ] **A12** New component count in the Blueprint is 15 added and 4 deleted; the report states the real numbers.

### B. Log checks (agent; the one 20 s run, nobody taps)
- [ ] **B1** The startup lines as in the 9b run: camera, GameMode, config (`Width=2250.0 Height=1800.0`), bases 3, cities 6, PlayerController, view target, crosshair, HUD, `[DD][Area] Viewport=...`.
- [ ] **B2** Exactly one of `[DD][Save] Loaded high=<n>` or `[DD][Save] No file high=<n>`, followed by exactly one `[DD][Flow] Title high=<n>`.
- [ ] **B3** During the 20 s wait: no `Spawner started`, no `[DD][Wave]`, no `[DD][Enemy]`, no `[DD][Flyer]`, no `[DD][Score]`, no `[DD][Damage]`, no `[DD][Input] Fire request`, **no `[DD][Aim]`**, no `[DD][Game]`, and no `[DD][Flow]` line except the one `Title` line.
- [ ] **B4** No line containing `ERROR`, `Accessed None`, `Divide by zero`, `Blueprint Runtime Error`, an array index error (`Attempted to access index`), or a material or shader compile error. (`UpdateLaunchers` runs every frame from the first tick; it must stay silent while `GM`, `Cfg` or `Bases` are not ready.)

### C. Visual checks (captures of the title screen; describe in words; NEEDS-HUMAN if you cannot capture)
- [ ] **C1** Three launchers stand at rest, pointing straight up: each shows wheels, a steel chassis, a steel launcher box with cyan bands and a dark centre seam, and a cyan cap at the top.
- [ ] **C2** The ammo numbers are readable; the launchers do not cover them.
- [ ] **C3** The six cities, the sky, glow, mountains and ground look as in 9b.
- [ ] **C4** No black quad, no hole, and no missing part (nothing invisible that should show).
- [ ] **C5** The tip of each launcher is at about the same height as the old barrel tip (about 70 above the ground top).

### D. Interactive checks (human, PIE) — all NEEDS-HUMAN
List every item as NEEDS-HUMAN in the report; do not run or claim them.
- [ ] **D1** **Tracking:** moving the mouse over the playfield, the launcher of the nearest base with ammo turns smoothly toward the crosshair; the other two stay vertical.
- [ ] **D2** **Handover:** moving the crosshair from one base's area to the next hands the turning from one launcher to the other; the old one eases back to vertical.
- [ ] **D3** **Fire:** a click makes the ABM leave from the cyan cap along the barrel (not from the middle of the launcher), and the explosion appears at the click.
- [ ] **D4** **Ammo and loss:** after a base is empty, the next nearest base turns instead; a destroyed base shows rubble and no launcher; with no ammo anywhere, no launcher turns.
- [ ] **D5** **Limits:** with the crosshair low and far to the side, a launcher turns at most to the horizontal and never into the ground; at +90 and -90 it does not poke beyond the base footprint or into a neighbouring city.
- [ ] **D6** **States:** on the title, while paused and after game over, all launchers stand vertical.
- [ ] **D7** **Tap snap (touch path on a PC):** set `bDesktopCrosshair` to false in the controller defaults (restore it afterwards), play, and click: the crosshair is absent, the launcher does not track, and at each click the firing launcher flicks toward the point, the ABM leaves from the cap, and the launcher eases back to vertical in about half a second.
- [ ] **D8** **Look:** the launchers read as Patriot-style launchers; wheels, box and cap are distinguishable at a small window.
- [ ] **D9** **Layering:** enemy missiles, ABMs and explosions are drawn in front of the launchers at every angle.
- [ ] **D10** **No gameplay change:** base choice, ammo numbers, speeds, scoring, waves, flyers, smart bombs, pause and quit behave as before (the center base still flies faster).

### D-DEVICE. Android (NEEDS-DEVICE)
- [ ] **T-1** A tap makes the firing launcher flick toward the tap point and the ABM leaves from its cap; the launcher returns to vertical; no other launcher moves.
- [ ] **T-2** The launchers are readable at phone size (wheels, box, cyan cap).
- [ ] **T-3** The frame rate is not noticeably lower than before this patch.

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped; the editor is on `L_Game`; nothing under `Spike` was touched; if 5b ran, the level is as before.

---

## 8. Report format (the agent's final message)

```
## Phase 9b.1 report
**Engine version:** x.y.z
**MCP tools used:** (list)

### Step 0 findings
(BP_MissileBase components with parents and flags; Tick and LaunchAt node names; config names;
 mesh sizes and pivots; whether any graph referenced the four deleted parts)

### Assets created or changed
(table: path | type | change | compiled clean)

### Component inventory
(table: component | parent | mesh | size | centre | material | bVisible | bHiddenInGame | Mobility | collision flag)

### Function and graph read-backs
(BP_MissileBase: the five new functions; BP_DDPlayerController: UpdateLaunchers, the start of Event Tick,
 LaunchAt before and after: which nodes and pins changed)

### Call audit (rule 13)
(table: function | call | pin | wired from / value)

### Static proof (Step 5)
(5a table; the rotation sign relied on; 5b results or why it was skipped)

### Acceptance results
A1 PASS/FAIL ... through E1 PASS/FAIL/NEEDS-HUMAN/NEEDS-DEVICE

### Log excerpt
(all [DD] lines of the run, with timestamps)

### Screenshot notes
(paths of the captures and a description of each: C1 to C5)

### Tool problems (errors, retries, editor restarts, writes that read back empty)
### Deviations
(fallbacks used, the Atan2 variant, the interpolation variant, anything else)

### Open questions / risks
```

---

## 9. What this patch hands to later phases

| Provided | How a later phase uses it |
|---|---|
| The `Muzzle` scene component | The attachment point for a muzzle flash, smoke puff or launch sound (9d, 9f). |
| `BP_MissileBase.SnapAim` / `StepAim` | A hook for recoil or a firing animation. |
| `[DD][Aim] Snap base=... deg=...` | One log line per shot: the base and the angle, for checking the aim in play logs. |
| `LauncherTurnSpeed`, `LauncherMaxAngle`, `LauncherSnapHold` | Tuning without a patch. |

**Known leftovers that stay:** `StopSpawner`, `bDirectorSilent`, the two disconnected spawn chains in `SpawnRandomEnemy`, HUD debug lines 2 to 4, the overlay `BP_PlayfieldDebug`, the debug keys, the `Spike` folder (removed in 9g).

---

## 10. Common pitfalls

1. **The launcher tilts the wrong way.** `AimAngle` is positive toward +X and the pitch is `-AimAngle` (rule 19). Verify in Step 5; change only that one negation.
2. **The component cannot rotate:** `Launcher` or a part under it is Static. All must be Movable (rule 12).
3. **Roll or Yaw wired instead of Pitch** on `Make Rotator`. Wire Pitch only; read all three pins back.
4. **`Atan2` arguments swapped.** The first is `dx`, the second `dz`. Swapped, the launcher points sideways when the cursor is above it.
5. **`Atan2` in radians** used as degrees. Use the degrees node, or multiply by 57.29578.
6. **`start` read before `SnapAim`.** The muzzle moves when the rotation is applied; `GetMuzzleStart` must run **after** `SnapAim` (3.6).
7. **Only one of the two `LaunchPos` reads replaced.** The ABM spawn location and the `Init` start must both be `start`, or the ABM jumps from the spawn point to the start on its first frame.
8. **`start` Y taken from the muzzle.** Y comes from `LaunchPos.Y`; the muzzle's world Y is the carrier Y (60 in front).
9. **`LaunchPos` changed or `SelectBase` edited.** Neither may change; selection and flight times stay as before.
10. **A Tick added to `BP_MissileBase`.** The controller drives all three launchers; the base keeps tick disabled.
11. **`UpdateLaunchers` placed after the early-return branch of `Event Tick`.** It would never run on touch devices (no crosshair) and the tap snap would never ease back. It goes first.
12. **`GM.Bases[i]` read on an empty array** at startup (an index error in the log). Use `For Each Loop` and the `nb >= 3` guard before `SelectBase`.
13. **A new platform test.** The crosshair is the test (3.1); do not add one.
14. **Hidden in Game left on** for a new part. Every new component: `bHiddenInGame` false, read back.
15. **A part sticking out in front** (front face beyond carrier Y + 8): enemy missiles would be hidden behind the launcher.
16. **Scales from guessed mesh sizes.** Use the Step 0 measurements; the rolled wheel cylinder needs scale (0.14, 0.14, 0.08), not a uniform scale.
17. **Editing `Demolish`, `Rebuild`, `ApplyVisual` or `InitBase`.** Not needed: a destroyed base has no ammo, is never selected, and its hidden launcher eases back by itself.
18. **Leaving the optional 5b actor in the level.** The actor count must match the count before.
19. **Marking D or T items PASS.** You cannot move the mouse, tap, or start a game. They are NEEDS-HUMAN and NEEDS-DEVICE.
20. **Touching `Spike` or cleaning up what this document does not list.**
