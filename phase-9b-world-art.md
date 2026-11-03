# Phase 9b — World Art (Sky, Ground, Cities, Bases)

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Revision** | 2. The answers given to the agent's questions during the 9b build (Q1 to Q7) are now part of this document, in the sections they affect. Index: Section 11. |
| **Depends on** | Phase 9a (toolbox check, accepted) and Phase 8c (accepted, with the `DotSmart` Hidden-in-Game fix committed) |
| **Phase goal** | **Visual only. No gameplay change.** Replace the flat placeholder world with a night scene built from engine primitives and procedural unlit materials: a gradient sky with stars, a soft additive horizon glow, a row of dark distant mountains, a gradient ground band with a bright top edge, **cities** drawn as small skylines with lit windows, **bases** drawn as dome-and-barrel turrets, and matching **rubble** for destroyed cities and bases. Every gameplay position, size, layer and function stays as it is. The horizon glow is an additive translucent material on purpose: it doubles as the Android smoke test for additive materials. |

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **No bars, no aspect caps.** The backdrop must cover the whole window at any aspect ratio, including very wide ones (see 3.3).
3. **Existing logic must survive** unless a step says to change it. Read each graph (`read_graph_dsl`) before editing it.
4. **Compile after every Blueprint change and recompile every material.** Every Blueprint ends with no errors and no warnings; every material compiles without errors.
5. **Run each step's "Check"** before moving on. **Save All after each step's Check passes.**
6. **You cannot press keys, click, touch, or resize in PIE, and you cannot start a game** (a game starts with a tap on the title). You can verify: compiles, static read-backs of every component, material, parameter and graph, and **the title screen in PIE** (the six cities and three bases are spawned at startup, so the new art is visible there). You **cannot** see a destroyed city, a rebuilt city, enemies in front of the art, or any wave. Those are group D (**NEEDS-HUMAN**); device checks are **NEEDS-DEVICE**. Never mark a D or T item PASS.
7. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
8. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after the run. **Save before loading a level.** **Do not touch Project Settings, input settings or any `Config/*.ini` file.** **Do not use, edit or delete anything in `/Game/DomeDefense/Spike/`.**
9. **NEVER REWRITE AN EXISTING FUNCTION OR EVENT GRAPH** with `write_graph_dsl`. A **brand-new** function (`ApplyStyle`) is written once, in one small write. **Existing graphs are changed only by small node or pin edits** (`create_node`, `connect_pins`, `set_pin_value`).
10. **READ EVERY NEW OR CHANGED GRAPH BACK** right after writing it. If a body reads back empty or incomplete: write it once more; if it is still wrong, **stop and report** under *Tool problems*.
11. **VISIBILITY AUDIT (the lesson of the invisible smart bomb).** `SetVisibility` changes only `bVisible`; it never clears **Hidden in Game** (`bHiddenInGame`). **Every new visual component must have `bVisible` true and `bHiddenInGame` FALSE, and so must the four existing carriers (`Body` and `Rubble` in `BP_City` and in `BP_MissileBase`).** If a carrier has `bHiddenInGame` true or `bVisible` false, set it right (Q4). Read both flags back for **every** new component and for the four carriers, and list them in the report. There are no exceptions: no component starts hidden.
12. **Check every call you build for a boolean or number pin left at its default.** In this phase: both `SetVisibility` calls in `BP_City.ApplyVisual` and `BP_MissileBase.ApplyVisual` (the **Propagate To Children** pin must read **true**), the `SetRelativeScale3D` call in `ApplyStyle`, and the `ApplyStyle` call in `InitCity`. Read each node's input pins back and list them in the report.
13. **After any tool error, do not retry the same large operation.** Stop that item, try one smaller piece at most, and if it still fails **stop and report**. **If the editor stops responding or restarts, stop and report.** Do not continue with a half-edited asset.
14. **Order is part of the spec:** read and measure first, then materials, then the backdrop, then cities, then bases, then the level, then the run.
15. **FROZEN: do not edit** `BP_DDGameMode` and every function in it, `BP_DDPlayerController`, `BP_DDHUD`, `BP_EnemyMissile`, `BP_Flyer`, `BP_ABM`, `BP_Explosion`, `BP_Crosshair`, `BP_TapMarker`, `BP_PlayfieldCamera`, `BP_PlayfieldDebug`, `BP_DDSave`, `BFL_DDDebug`, `PDA_DDConfig`, `DA_DDConfig`, the `Ground` actor in the level, the existing materials `M_DD_Unlit`, and every existing material instance except `MI_DD_Ground`. **The only edits to existing assets are listed in section 4.**
16. **No new gameplay logic, no new config values, no new log lines, no new keys.** The only new function is `BP_City.ApplyStyle` (a visual scale call).
17. **Do not clean up anything this document does not list.**

---

## 2. Scope

### In scope
- New materials and material instances (3.5).
- New Blueprint `BP_Backdrop` (sky, glow, mountains, ground edge), placed once in `L_Game`.
- `BP_City` and `BP_MissileBase`: new art components under the existing `Body` and `Rubble` components; the `ApplyVisual` pin edits; `BP_City.ApplyStyle` and one call to it.
- `MI_DD_Ground` re-parented to the new gradient material.

### Out of scope (do NOT do)
- Missile, ABM, explosion, flyer, crosshair, tap-marker or HUD art; trails; sound; post-processing; any texture import; any Niagara; Paper2D; any Project or Android setting.
- Moving, resizing or re-timing anything that gameplay reads: positions, the ground band (Z 0 to 100, top at exactly Z = 100), the base launch point (Z = 170), `AmmoText`, slot spacing.
- Per-wave palette changes (later phase).

---

## 3. Design (decided; do not change)

### 3.1 Layers (world Y; larger Y is closer to the camera at +Y)

| Layer | Y |
|---|---|
| **Sky plane (new)** | **-120** |
| **Horizon glow plane (new)** | **-100** |
| **Mountains (new)** | **-80** (each 10 deep, spans -85 to -75) |
| Ground cube (existing, spans Y -50 to +50) | 0 |
| **Ground top edge strip (new)** | **+55** (10 deep, spans +50 to +60) |
| Bases, cities, ammo numbers (carriers `Body` / `Rubble`) | +60 |
| Enemies | +70 |
| ABMs | +80 |
| Explosions | +90 |
| Crosshair / tap marker | +100 / +110 |

**Front-face rule:** no part of any city or base art (healthy or rubble) may have its front face beyond **carrier Y + 8** (world Y 68). Enemy missiles (Y = 70) must always be drawn in front of every building and base. All art parts are therefore **thin slabs (at most 15 deep)**.

### 3.2 Gameplay-fixed geometry (must not change)
Ground band: Z 0 to 100, top at exactly 100. Cities and bases stand on Z = 100 (the actor origin). Slot spacing is 252. City footprint at most 120 wide and **at most 100 tall** (art is at most 85 tall before the style scale, 3.7). Base footprint at most 100 wide; **the barrel tip must end at actor-relative Z = 70** (world Z 170, the launch point, `BaseLaunchOffsetZ` = 70).

### 3.3 Coverage of any window
The visible area is always 1800 high (Z 0 to 1800) and between about 2400 and 4400 wide depending on the window. The sky plane is 12000 × 3600, the glow plane 8000 × 1200, the ground cube is 20000 wide, the mountain row spans X -2700 to +2700 (plus half-widths), the ground edge strip is 20000 wide. Nothing here may be made smaller.

### 3.4 Palette (linear RGB; all values are material instance parameters, so they can be tuned later without graph edits)

| Item | Value |
|---|---|
| Sky top / horizon | (0.01, 0.02, 0.07) / (0.07, 0.09, 0.22) |
| Horizon glow colour, intensity | (0.06, 0.10, 0.24), 1.0 |
| Mountains | (0.025, 0.035, 0.085) |
| Ground top / bottom | (0.50, 0.29, 0.07) / (0.22, 0.12, 0.03) |
| Ground edge strip | (0.85, 0.72, 0.42) |
| City body / window | (0.13, 0.16, 0.23) / (1.0, 0.80, 0.40) |
| Base body / accent | (0.30, 0.34, 0.40) / (0.15, 0.85, 0.95) |
| Rubble (existing `MI_DD_Rubble`) | (0.28, 0.28, 0.30) |
| Ember | (0.55, 0.20, 0.04) |

**Readability rule (for the backdrop only: sky, glow, mountains):** no channel above 0.35 anywhere, and no red, orange or magenta hue. Enemies are red, explosions orange, the smart bomb magenta; they must stand out. Only the small city windows are allowed to be bright.

### 3.5 Materials (all Unlit, two-sided, every `Color`-like input goes to Emissive)

Node budget: **at most 60 expressions per material**, **no Noise node**, no texture samples, no Time. Report the expression count of each. The class for linear interpolation is `MaterialExpressionLinearInterpolate`. Use `list_expression_classes` once in Step 0 to get the exact class names for the nodes below (WorldPosition, ComponentMask, Abs, Saturate, Power, Floor, Frac, Sine, Dot, Step (or an If / comparison node), Append, TextureCoordinate, Constant, Constant2Vector, ScalarParameter, VectorParameter, Add, Subtract, Multiply, Divide, OneMinus, Distance).

| Material | Blend | Parameters (exact names) | Logic |
|---|---|---|---|
| `M_DD_Ground` | Opaque | vectors `TopColor`, `BottomColor`; scalars `Z0` (0), `Z1` (100) | `t = saturate((WorldZ - Z0) / (Z1 - Z0))`; emissive = lerp(`BottomColor`, `TopColor`, `t`). |
| `M_DD_Sky` | Opaque | vectors `TopColor`, `HorizonColor`; scalars `GradientZ0` (100), `GradientZ1` (1800), `GradientPower` (0.8), `StarsEnabled` (1), `StarDensity` (0.035), `StarBrightness` (0.9), `StarCell` (70), `StarRadius` (0.06) | Gradient: `t = saturate((WorldZ - GradientZ0) / (GradientZ1 - GradientZ0))^GradientPower`; base = lerp(`HorizonColor`, `TopColor`, `t`). Stars (world X and Z): `p = (WorldX, WorldZ) / StarCell`; `cell = floor(p)`; `f = frac(p)`; `h1 = frac(sin(dot(cell, (12.9898, 78.233))) * 43758.5453)`; `h2` and `h3` the same with the constant pairs (39.346, 11.135) / 24634.6345 and (73.156, 52.235) / 31417.2718; star centre `(0.2 + 0.6 * h2, 0.2 + 0.6 * h3)`; `d = distance(f, centre)`; `star = step(h1, StarDensity) * saturate(1 - d / StarRadius)^2`; fade near the horizon by `saturate((WorldZ - 300) / 200)`; emissive = base + (0.8, 0.85, 1.0) * star * `StarBrightness` * `StarsEnabled` * fade. |
| `M_DD_Glow` | **Additive** | vector `Color`; scalars `Intensity` (1), `GlowZ` (110), `GlowHeight` (420), `GlowPower` (1.6) | `g = saturate(1 - abs(WorldZ - GlowZ) / GlowHeight)^GlowPower`; emissive = `Color` * `Intensity` * `g`; opacity 1. (Same additive technique as the 9a `M_Spike_Fireball`.) |
| `M_DD_Windows` | Opaque | vectors `BodyColor`, `WindowColor`; scalars `Cols`, `Rows`, `LitFraction` (0.7), `Margin` (0.22), `Seed` (0) | `uv = TexCoord * (Cols, Rows)`; `cell = floor(uv)`; `f = frac(uv)`; window mask = `step(Margin, f.x) * step(f.x, 1 - Margin) * step(Margin, f.y) * step(f.y, 1 - Margin)`; `lit = step(frac(sin(dot(cell + Seed, (12.9898, 78.233))) * 43758.5453), LitFraction)`; windowColour = lerp(`BodyColor` * 0.6, `WindowColor`, `lit`); emissive = lerp(`BodyColor`, windowColour, mask). |

**Fallbacks (use in this order, record each in *Deviations*):**
- If `Floor`, `Frac`, `Sine` or `Dot` do not exist: `M_DD_Sky` ships **without stars** (gradient only, `StarsEnabled` removed from the report) and `M_DD_Windows` ships with a **regular** lit grid (no random lit/unlit; `lit` = 1 for every cell).
- If `Floor` or `Frac` are missing entirely: `M_DD_Windows` ships as a flat `BodyColor` material with the same parameter names (windows omitted).
- If `WorldPosition` does not exist: use `TextureCoordinate.V` on the plane instead of `WorldZ`, scaling to the plane (sky: `V` 0..1 over 3600 high from Z -900; glow: `V` over 1200 high from Z -300; ground: `V` over the cube face); record the mapping.

**Over-budget rule (Q5, Q7).** If every node class exists but a material would go past 60 expressions, **do not stop and do not ask.** Apply these in order and record which step was used and the final expression count of each material:
- **(a) Stars, simplified.** One hash per cell (`h1` only). The star centre is `(0.2 + 0.6 * frac(h1 * 17), 0.2 + 0.6 * frac(h1 * 91))`. **Keep the 0.2 / 0.6 inset:** without it a star can sit on a cell edge and be cut in half, because each cell draws only its own star. The `^2` falloff may use the Power node's default exponent (2) instead of a constant node.
- **(b)** If `M_DD_Sky` is still over 60: drop the stars and keep the gradient (`StarsEnabled` is then removed from the report).
- **(c) Windows.** Keep the lit / unlit hash if `M_DD_Windows` fits in 60. If it does not, use the regular grid (every window lit), which is the first fallback above.

(The full three-hash star field counts about 65 expressions, so step (a) is expected to be needed.)

Material instances (parent in brackets):
`MI_DD_Sky` [`M_DD_Sky`, defaults from 3.4], `MI_DD_Glow` [`M_DD_Glow`], `MI_DD_Ground` (**existing**, re-parent to `M_DD_Ground`, set `TopColor`, `BottomColor`), `MI_DD_Mountain` [`M_DD_Unlit`, `Color` from 3.4], `MI_DD_GroundEdge` [`M_DD_Unlit`], `MI_DD_BaseBody` [`M_DD_Unlit`], `MI_DD_BaseAccent` [`M_DD_Unlit`], `MI_DD_Ember` [`M_DD_Unlit`], and four window instances `MI_DD_CityWin_1` to `MI_DD_CityWin_4` [`M_DD_Windows`] with the `Cols`, `Rows` and `Seed` of 3.7 and the shared `BodyColor`, `WindowColor`, `LitFraction` 0.7, `Margin` 0.22.

### 3.6 `BP_Backdrop` (new, placed once in `L_Game` at (0, 0, 0))
Root: `DefaultSceneRoot`. All components: `Mobility` Static, **collision disabled** (`bodyInstance.collisionEnabled` = `NoCollision`), **`bVisible` true and `bHiddenInGame` false**, and if a `CastShadow` property exists, false. Engine meshes under `/Engine/BasicShapes/`. **All sizes below are target world sizes (X × Y × Z); compute each component's relative scale as target size divided by the measured mesh size (Step 0).** Positions are centres.

| Component | Mesh | Target size | Centre | Rotation | Material |
|---|---|---|---|---|---|
| `Sky` | Plane | 12000 × 3600 | (0, -120, 900) | roll 90 (faces the camera, as in 9a) | `MI_DD_Sky` |
| `Glow` | Plane | 8000 × 1200 | (0, -100, 300) | roll 90 | `MI_DD_Glow` |
| `GroundEdge` | Cube | 20000 × 10 × 4 | (0, 55, 102) | none | `MI_DD_GroundEdge` |
| `Mountain_00` to `Mountain_15` | Cone | table below | (x, -80, zc) | none | `MI_DD_Mountain` |

**Plane sizes (Q3).** For the two planes the two numbers are the **world width (X) and the world height (Z)**. Use roll 90 and leave the flat axis at the mesh's native thickness (scale 1 on it). The scale on each of the two used axes is the target size divided by the measured plane size on that axis. Planes are one-sided; the materials are two-sided, so facing does not matter.

**Pivot rule (Q2).** Every centre in this document (3.6, 3.7, 3.8) is the **visual centre** of the part. Step 0 measures where each mesh pivot is. If a mesh's pivot is not at its visual centre, **offset the component's relative location** (taking its scale and, for planes, its roll into account) so that the visual centre lands on the table value. List every offset applied in the report. Verify the mountains by arithmetic: base at Z = 80 and apex at 80 + H, each within 1 unit.

Mountains are triangles seen from the front: target size (W, 10, H), centre Z `zc = 80 + H / 2` (so the apex is at Z = 80 + H, at most 240, and the base is hidden behind the ground band).

| i | x | W | H | zc | | i | x | W | H | zc |
|---|---|---|---|---|---|---|---|---|---|---|
| 00 | -2700 | 300 | 90 | 125 | | 08 | 180 | 440 | 150 | 155 |
| 01 | -2340 | 420 | 140 | 150 | | 09 | 540 | 320 | 95 | 127.5 |
| 02 | -1980 | 360 | 110 | 135 | | 10 | 900 | 380 | 120 | 140 |
| 03 | -1620 | 280 | 70 | 115 | | 11 | 1260 | 260 | 65 | 112.5 |
| 04 | -1260 | 460 | 160 | 160 | | 12 | 1620 | 420 | 140 | 150 |
| 05 | -900 | 340 | 100 | 130 | | 13 | 1980 | 340 | 105 | 132.5 |
| 06 | -540 | 400 | 130 | 145 | | 14 | 2340 | 300 | 85 | 122.5 |
| 07 | -180 | 300 | 80 | 120 | | 15 | 2700 | 400 | 125 | 142.5 |

### 3.7 City art (inside `BP_City`; sizes are target world sizes; centres are relative to the `Body` carrier, Z up from the ground top)

**Carrier change.** `Body` (existing StaticMeshComponent, currently the cube scale (1.2, 0.4, 0.8) at (0, 60, 40)) becomes a **visibility carrier**: mesh set to **None**, relative location **(0, 60, 0)**, relative scale **(1, 1, 1)**, collision stays NoCollision. `Rubble` (existing, cube at (0, 60, 7)) the same: mesh **None**, location **(0, 60, 0)**, scale **(1, 1, 1)**. Their visibility is still toggled by `ApplyVisual`; with **Propagate To Children = true** (Step 3) all art parts attached under them follow.

**Carrier flags (Q4).** If either carrier has `bHiddenInGame` true, set it to false; if `bVisible` is false, set it to true. Otherwise every child would stay invisible in a game. This is part of the carrier change (it does not break the frozen-asset rule). Record the values before and after in the report.

**Other components (Q6).** Read the parent of every existing component first. Any existing component other than `Body` and `Rubble` that is attached to one of them must keep its world transform (see 3.8).

**Healthy art** (attached under a new SceneComponent `CityArt`, which is attached under `Body`, relative location (0, 0, 0)):

| Part | Mesh | Size | Centre | Material | `Cols` / `Rows` / `Seed` |
|---|---|---|---|---|---|
| `Bld1` | Cube | 28 × 10 × 62 | (-42, 0, 31) | `MI_DD_CityWin_1` | 3 / 6 / 1 |
| `Bld2` | Cube | 26 × 10 × 85 | (-14, 2, 42.5) | `MI_DD_CityWin_2` | 3 / 8 / 2 |
| `Bld3` | Cube | 26 × 10 × 70 | (14, 0, 35) | `MI_DD_CityWin_3` | 3 / 7 / 3 |
| `Bld4` | Cube | 28 × 10 × 50 | (42, 2, 25) | `MI_DD_CityWin_4` | 3 / 5 / 4 |

**Rubble art** (attached directly under `Rubble`; `MI_DD_Rubble`, embers `MI_DD_Ember`; `pitch` is rotation about Y):

| Part | Mesh | Size | Centre | Pitch |
|---|---|---|---|---|
| `Rub1` | Cube | 34 × 10 × 10 | (-36, 0, 6) | +6 |
| `Rub2` | Cube | 28 × 10 × 16 | (-6, 0, 10) | -9 |
| `Rub3` | Cube | 30 × 10 × 12 | (24, 0, 8) | +12 |
| `Rub4` | Cube | 22 × 10 × 8 | (46, 0, 5) | -5 |
| `Ember1` | Sphere | 7 × 7 × 7 | (-10, 3, 17) | 0 |
| `Ember2` | Sphere | 7 × 7 × 7 | (28, 3, 13) | 0 |

**Per-city variety (`ApplyStyle`, new function, 3.8):** by `CityIndex`, `CityArt` is scaled in Z by 0.85, 1.0 or 1.1 (index mod 3 = 0, 1, 2) and mirrored in X for odd indices (X scale -1). Tallest result: 85 * 1.1 = 93.5, within the 100 limit.

### 3.8 Base art (inside `BP_MissileBase`; centres relative to the `Body` carrier)

**Carrier change.** The same as the city, including the carrier flags of 3.7 (`bVisible` true, `bHiddenInGame` false; Q4): `Body` mesh None, location (0, 60, 0), scale (1, 1, 1); `Rubble` mesh None, location (0, 60, 0), scale (1, 1, 1). The `AmmoText` and `LaunchPos` logic is **not edited**, and **their world positions must not change** (Q6):
- Read the parent of `AmmoText` first. If it is attached to the root (as the Phase 2 table says), nothing needs to change. If it is attached to `Body` or `Rubble`, the new carrier transform would move it: either re-attach it to the root with the **same world transform**, or adjust its relative transform so that its world location, rotation and effective scale are identical to before. The ammo digits keep the same position and size.
- `LaunchPos` is probably a variable computed in `InitBase`, not a component. If so, leave it alone. The launch point stays at world Z 170 (actor Z + `BaseLaunchOffsetZ` 70). If it is a component attached to `Body`, handle it as `AmmoText` above.
- Report what you found (parents, world transforms before and after) and what you did.

**Healthy art** (attached under `Body`):

| Part | Mesh | Size | Centre | Material |
|---|---|---|---|---|
| `Podium` | Cylinder | 100 × 14 × 16 | (0, 0, 8) | `MI_DD_BaseBody` |
| `Dome` | Sphere | 64 × 10 × 56 | (0, 0, 28) | `MI_DD_BaseBody` |
| `Barrel` | Cube | 8 × 6 × 34 | (0, 3, 53) | `MI_DD_BaseBody` |
| `Accent` | Cube | 100 × 15 × 3 | (0, 0, 16) | `MI_DD_BaseAccent` |

(The podium is deeper than the dome, so its front face hides the lower half of the sphere; nothing sticks out below Z = 0. The barrel spans Z 36 to 70: its tip is at the launch height.)

**Rubble art** (attached under `Rubble`; `MI_DD_Rubble`, ember `MI_DD_Ember`):

| Part | Mesh | Size | Centre | Pitch |
|---|---|---|---|---|
| `Rub1` | Cube | 30 × 10 × 9 | (-30, 0, 6) | +7 |
| `Rub2` | Cube | 26 × 10 × 14 | (-4, 0, 9) | -10 |
| `Rub3` | Cube | 28 × 10 × 10 | (22, 0, 7) | +11 |
| `Rub4` | Cube | 18 × 10 × 7 | (42, 0, 4.5) | -6 |
| `Ember1` | Sphere | 6 × 6 × 6 | (6, 3, 16) | 0 |

### 3.9 Budget
New components in total at most **120** (backdrop 19, six cities 66, three bases 27 = 112). Shared material instances only. No post-processing, no translucency except the single additive glow plane, no per-frame logic.

---

## 4. What this phase changes in earlier specs

| Earlier text | Now |
|---|---|
| Phase 0 `MI_DD_Ground` (parent `M_DD_Unlit`, `Color`) | Parent becomes `M_DD_Ground`; the `Ground` actor in the level is **not** edited (it keeps referencing `MI_DD_Ground`). |
| Phase 5 `BP_City.Body` / `Rubble` cube meshes | Mesh None, scale 1, `bVisible` true, `bHiddenInGame` false; they carry art children. |
| Phase 2 and 5 `BP_MissileBase.Body` / `Rubble` cube meshes | Mesh None, scale 1, `bVisible` true, `bHiddenInGame` false; they carry art children. `AmmoText` and the launch point keep their world positions (3.8). |
| Phase 5 `ApplyVisual` (both Blueprints) | The two `SetVisibility` calls get **Propagate To Children = true**. Nothing else in the function changes. |
| Phase 5 `InitCity` | One call `ApplyStyle()` added after `ApplyVisual()`. |
| Layer table | Three new layers behind the ground and a ground-edge strip (3.1). |
| Phase 0 / 1.1 `L_Game` | One placed actor `BP_Backdrop`. |

---

## 5. Tooling notes (from the 9a report)

1. `list_expression_classes` gives class names; the interpolation node is `MaterialExpressionLinearInterpolate` (a search for "Lerp" finds nothing). Nodes created and left unused should be deleted (`delete_unused_expressions`).
2. Material work: `MaterialTools.create_material`, `add_expression`, `connect_expressions`, `connect_to_output`, `get_expressions`, `recompile`, `layout_expressions`; instances: `MaterialInstanceTools.create`, `set_parent`, `list_parameters`, set scalar and vector parameters. Instruction counts are not readable; report the expression count instead.
3. **Components on a Blueprint** are added with `ActorTools.add_component` (engine meshes by path). `PrimitiveTools.add_cube` and its siblings need a **placed actor instance** and fail on a Blueprint asset. `add_component` has no parent argument; **`ActorTools.set_parent_component` counts as attach support** (Q1). Add the component, set its parent, then read back its parent and its relative transform and correct the location if it changed. Use Fallback F1 only if the parent cannot be set or does not read back.
4. Create Blueprints with `BlueprintTools.create`. There is no create-level tool; `L_Game` already exists. `SceneTools.add_to_scene_from_asset` places an actor; save the level with `AssetTools.save_assets`.
5. **PIE cannot spawn actors** (`Cannot create actors while PIE is active`). The title screen already shows the cities and bases because the game mode spawns them at startup, so a PIE capture of the title is the verification view.
6. `CaptureViewport` and `CaptureEditorImage` work; describe in words what each capture shows. You cannot see motion.
7. Key function patterns from earlier phases still hold: stored locals, Select nodes, the integer remainder node (`Percent`) built with `create_node`; a void function has no ReturnNode.
8. If a tool reports that a property cannot be read or set, say so; do not guess.

---

## 6. Implementation steps

### Step 0 — Read and measure (no changes)
1. Read `BP_City` and `BP_MissileBase`: components (**with the parent of each**), their transforms and meshes, variables, the graphs of `ApplyVisual`, `InitCity` / `InitBase`, `Demolish`, `Rebuild`. For each `SetVisibility` call in the two `ApplyVisual` functions, record the current value of its **Propagate To Children** pin. Record `bVisible` and `bHiddenInGame` of the existing `Body` and `Rubble` in both Blueprints (Q4). In `BP_MissileBase` record also the parent and the **world transform** of `AmmoText`, and whether `LaunchPos` is a component or a variable (Q6).
2. Read `MI_DD_Ground`, `MI_DD_Rubble`, `M_DD_Unlit`.
3. **Measure the engine meshes** `Cube`, `Sphere`, `Cylinder`, `Cone`, `Plane` under `/Engine/BasicShapes/` (bounding box size and whether the pivot is at the centre). Put the five sizes in the report; all scales below derive from them. Where a pivot is not at the visual centre, the pivot rule of 3.6 applies (Q2).
4. Read the camera component's orthographic clip settings on `BP_PlayfieldCamera` (read only): auto-calculate planes, near, far. Record them. **Do not change them.**
5. Run `list_expression_classes` and record the exact class names for the nodes in 3.5. Decide which fallbacks of 3.5 apply.
6. **Attach support (Q1).** `add_component` has no parent argument, but `set_parent_component` exists and **counts as attach support**. Confirm that it can attach a new component under an existing one and that the parent and the relative transform read back. Do this on a throwaway test in the `BP_City` editor only if reading the tool cannot settle it, then remove the test component and compile (record this in the report). Only if the parent cannot be set, or does not read back, do Step 3 and 4 use the **tag fallback** (F1).

**Check:** the report section for Step 0 is complete (mesh sizes and pivots, camera clip values, node class list, propagate pins, carrier flags, parent of every component, `AmmoText` parent and world transform, attach support).

### Step 1 — Materials
Create, in `/Game/DomeDefense/Materials/`, in this order: `M_DD_Ground`, `M_DD_Sky`, `M_DD_Glow`, `M_DD_Windows` (3.5). Recompile each and read back `get_expressions`. Then create the instances of 3.5, set every parameter to the palette values of 3.4, and **re-parent `MI_DD_Ground` to `M_DD_Ground`** and set its `TopColor` and `BottomColor`. Read all parameters back (`list_parameters`).

**Fallback (re-parenting):** if `set_parent` fails on `MI_DD_Ground`, create `MI_DD_GroundArt` instead and set the `Ground` actor's material override to it in `L_Game` (the one permitted `Ground` edit); record it.

**Check:** every material recompiles without errors; every parameter reads back; `MI_DD_Ground` reads parent `M_DD_Ground`; expression counts listed (each at most 60). **Save All.**

### Step 2 — `BP_Backdrop`
Create `BP_Backdrop` in `/Game/DomeDefense/Actors/`. Add the 19 components of 3.6 (names exact), set meshes, scales (from the Step 0 measurements), relative locations, rotations, materials, Mobility Static, collision disabled, and `CastShadow` false if the property exists. **Read back every component**: mesh, location, scale, rotation, material, `bVisible`, `bHiddenInGame`, collision flag. Compile.

**Check:** 19 components present; every `bVisible` true and `bHiddenInGame` false; the mountain apex values match the table (max 240); `BP_Backdrop` compiles clean. **Save All.**

### Step 3 — `BP_City`
Read the Blueprint again first.
1. **Carriers:** set `Body` and `Rubble` as in 3.7 (mesh None, location (0, 60, 0), scale (1, 1, 1), `bVisible` true, `bHiddenInGame` false; Q4); read back, and record the flags before and after.
2. **Healthy art:** add a SceneComponent `CityArt` attached under `Body` at (0, 0, 0); add `Bld1` to `Bld4` under `CityArt`. Attach every new component with `set_parent_component` (Q1); after each re-parent read back its parent and relative transform and correct the location if it changed. Apply the pivot rule of 3.6 to every part (Q2).
3. **Rubble art:** add `Rub1` to `Rub4`, `Ember1`, `Ember2` under `Rubble`.
4. Every new component: collision disabled, `bVisible` true, **`bHiddenInGame` false**, `CastShadow` false if available. Read all back.
5. **`ApplyVisual` pins:** on the two `SetVisibility` calls (on `Body` and on `Rubble`) set **Propagate To Children = true** (`set_pin_value`). Read both nodes back.
6. **`ApplyStyle`** (new function, no inputs, one small write, read back):
```
idx = CityIndex                                        // store
m3  = idx Percent 3                                    // integer remainder via create_node; store
m2  = idx Percent 2                                    // store
sz  = Select(m3 == 0, 0.85, Select(m3 == 1, 1.0, 1.1)) // store
sx  = Select(m2 == 0, 1.0, -1.0)                       // store
CityArt.SetRelativeScale3D(Make Vector(sx, 1.0, sz))   // rule 12: read back all three inputs
```
7. **`InitCity`:** insert one call `ApplyStyle()` **after** the existing `ApplyVisual()` call (small node edit; read `InitCity` back; nothing else changes).
8. Compile.

**Fallback F1 (the parent cannot be set or does not read back, Step 0.6 and Q1):** give every healthy art part the component tag `Art` and every rubble part the tag `Ruin`; in `ApplyVisual`, after the two existing `SetVisibility` calls, add two loops over `GetComponentsByTag(StaticMeshComponent, "Art")` and `("Ruin")` calling `SetVisibility(NOT bDestroyed, false)` and `SetVisibility(bDestroyed, false)` (stored loop bounds; read back). In that case `CityArt` and `ApplyStyle` are **skipped** (all six cities identical) and the *Propagate* edit is not needed; record all of it.
**Fallback F2 (negative scale rejected or logged as a warning):** in `ApplyStyle` use `CityArt.SetRelativeRotation` with yaw 180 for odd indices instead of the negative X scale (the materials are two-sided); record it.

**Check:** `BP_City` compiles clean; carriers read back; 10 healthy and rubble parts plus `CityArt` present with the visibility flags of rule 11; `ApplyVisual` pins read back **true**; `ApplyStyle` reads back non-empty and is called from `InitCity` only; `Demolish`, `Rebuild`, `GetBodyShown`, `GetRubbleShown` unchanged. **Save All.**

### Step 4 — `BP_MissileBase`
Read the Blueprint first. Same as Step 3 without `CityArt` and without `ApplyStyle`:
1. Carriers: `Body` and `Rubble` as in 3.8, including `bVisible` true and `bHiddenInGame` false (Q4); read back, and record the flags before and after.
2. Add `Podium`, `Dome`, `Barrel`, `Accent` under `Body`; `Rub1` to `Rub4` and `Ember1` under `Rubble`. Attach with `set_parent_component`, read back each parent and relative transform and correct the location if it changed (Q1); apply the pivot rule of 3.6 (Q2).
3. Every new component: collision disabled, `bVisible` true, `bHiddenInGame` false, `CastShadow` false if available. Read back.
4. `ApplyVisual`: **Propagate To Children = true** on both `SetVisibility` calls; read back. (Fallback F1 applies here too if the parent cannot be set or does not read back.)
5. Do **not edit** `AmmoText`, `SetAmmo`, `SetAmmoDisplay`, `InitBase`, `Demolish`, `Rebuild`, `LaunchPos`. **Keep the world positions of `AmmoText` and the launch point exactly as they were (3.8, Q6):** after the carrier change read `AmmoText`'s parent and world transform again and compare them with the Step 0 values; if it moved, correct it as 3.8 says. Report what you found and what you did.
6. Compile.

**Check:** `BP_MissileBase` compiles clean; the barrel tip centre plus half height equals actor-relative Z 70 (read back and compute); no part's front face beyond carrier Y + 8; visibility flags as in rule 11; `AmmoText` is still written only inside `SetAmmoDisplay` (search the graphs); `AmmoText` world location, rotation and size equal the Step 0 values; the launch point is still world Z = actor Z + 70. **Save All.**

### Step 5 — Place the backdrop in the level
Load `L_Game`, place **one** `BP_Backdrop` at (0, 0, 0) with `SceneTools.add_to_scene_from_asset`, read its transform back, and **save the level** (`save_assets`). Confirm that the level contains exactly one `BP_Backdrop`. Do not modify any other actor.

**Fallback:** if the actor cannot be placed or the level cannot be saved, **stop and report**; do not edit `BP_DDGameMode` without approval.

**Check:** exactly one `BP_Backdrop` in `L_Game`; the level is saved and not dirty. **Save All.**

### Step 6 — Run
1. **Save All.**
2. `StartPIE`. The title screen appears at once. **Wait about 20 seconds**; nobody taps, so no game starts.
3. `CaptureViewport` (or `CaptureEditorImage`) at about 5 s and again at about 15 s. Describe each image in words: sky gradient and stars, glow at the horizon, mountains behind the ground, ground band with its top edge, six city skylines, three bases (dome, barrel, accent stripe), the ammo numbers, the title text.
4. `GetLogEntries` (filter `[DD]`), then `StopPIE`.
5. Fill in the report (section 8).

---

## 7. Acceptance tests

### A. Static checks
- [ ] **A1** New assets: `M_DD_Ground`, `M_DD_Sky`, `M_DD_Glow`, `M_DD_Windows`, `MI_DD_Sky`, `MI_DD_Glow`, `MI_DD_Mountain`, `MI_DD_GroundEdge`, `MI_DD_BaseBody`, `MI_DD_BaseAccent`, `MI_DD_Ember`, `MI_DD_CityWin_1` to `_4`, `BP_Backdrop`. Changed: `MI_DD_Ground`, `BP_City`, `BP_MissileBase`, `L_Game`. Nothing else, and nothing under `Spike`. No Project Settings, input settings or `.ini` change.
- [ ] **A2** `BP_Backdrop`, `BP_City`, `BP_MissileBase` compile with 0 errors and 0 warnings; every new material recompiles without errors; expression counts each at most 60; no Noise node; the star and window variant used (Q5) is recorded.
- [ ] **A3** All material parameters read back with the 3.4 and 3.5 values; `MI_DD_Ground` parent is `M_DD_Ground` (or the recorded fallback).
- [ ] **A4** `BP_Backdrop` has exactly 19 components matching 3.6 (positions, target sizes derived from the measured meshes, materials); every mountain apex is at most 240; every pivot offset applied (Q2) is listed, and the mountain base (Z = 80) and apex (80 + H) check out within 1 unit.
- [ ] **A5** `BP_City`: `Body` and `Rubble` are mesh-less carriers at (0, 60, 0), scale 1, `bVisible` true, `bHiddenInGame` false (Q4); `CityArt` under `Body`; `Bld1` to `Bld4`; `Rub1` to `Rub4`, `Ember1`, `Ember2`; sizes and centres as in 3.7; each building's material and `Cols`/`Rows`/`Seed` as in the table.
- [ ] **A6** `BP_MissileBase`: carriers as in 3.8; `Podium`, `Dome`, `Barrel`, `Accent`; `Rub1` to `Rub4`, `Ember1`; barrel tip at actor-relative Z 70; nothing below Z = -1 or beyond the front-face limit; carrier flags as in A5; `AmmoText` world location, rotation and size unchanged from Step 0 (Q6).
- [ ] **A7** **Visibility audit:** a table of every new component **and of the four carriers** with `bVisible` and `bHiddenInGame`: all true and false respectively. No component starts hidden.
- [ ] **A8** Both `ApplyVisual` functions: the `SetVisibility` calls read **Propagate To Children = true** (or Fallback F1 recorded); `Demolish`, `Rebuild`, `GetBodyShown`, `GetRubbleShown` unchanged.
- [ ] **A9** `ApplyStyle` reads back as in Step 3 (the values 0.85 / 1.0 / 1.1, the sign of X); it is called from `InitCity` only, after `ApplyVisual()`.
- [ ] **A10** Call audit (rule 12): every pin listed with its value or source.
- [ ] **A11** `L_Game` contains exactly one `BP_Backdrop` at (0, 0, 0) and is saved. The `Ground` actor, `PlayerStart`, `PlayfieldCamera`, `PlayfieldDebug` are unchanged.
- [ ] **A12** Unchanged: every asset named in rule 15; `AmmoText` is still written only inside `SetAmmoDisplay`; `AmmoText` keeps its world position, rotation and size, and the launch point stays at world Z = actor Z + 70 (`LaunchPos` not edited).
- [ ] **A13** Component total (new) at most 120; the report states the real number.

### B. Log checks (agent; the one 20 s run, nobody taps)
- [ ] **B1** The startup lines as in the 8c run: camera, GameMode, config (`Width=2250.0 Height=1800.0`), bases 3, cities 6, PlayerController, view target, crosshair, HUD, `[DD][Area] Viewport=...`.
- [ ] **B2** Exactly one of `[DD][Save] Loaded high=<n>` or `[DD][Save] No file high=<n>`, followed by exactly one `[DD][Flow] Title high=<n>`.
- [ ] **B3** During the 20 s wait: no `Spawner started`, no `[DD][Wave]`, no `[DD][Enemy]`, no `[DD][Flyer]`, no `[DD][Score]`, no `[DD][Damage]`, no `[DD][Input] Fire request`, no `[DD][Game]`, and no `[DD][Flow]` line except the one `Title` line.
- [ ] **B4** No line containing `ERROR`, `Accessed None`, `Divide by zero`, `Blueprint Runtime Error`, or a material or shader compile error.

### C. Visual checks (captures of the title screen; describe in words; NEEDS-HUMAN if you cannot capture)
- [ ] **C1** The sky is a dark blue gradient lighter near the horizon, with small stars in the upper part.
- [ ] **C2** A soft blue glow sits at the horizon, behind the mountains (not a solid box).
- [ ] **C3** Dark mountain silhouettes show above the ground band, behind the cities.
- [ ] **C4** The ground band shows a gradient and a bright sand-coloured top edge; the three ammo numbers are readable on it.
- [ ] **C5** Six city skylines with lit windows stand on the ground, with visible variety in height and mirroring; the three bases show a dome, a barrel and a cyan accent stripe.
- [ ] **C6** The title text and exit icon are unchanged and readable.
- [ ] **C7** No black quad, no hole at the edges of the window, and no missing component (anything invisible that should show).

### D. Interactive checks (human, PIE) — all NEEDS-HUMAN
List every item as NEEDS-HUMAN in the report; do not run or claim them.
- [ ] **D1** `X` destroys a random city: its skyline disappears and rubble with embers shows; the next wave start (or the rebuild path) restores the skyline. A base destroyed in play shows its rubble and no dome; a rebuilt base shows its dome again.
- [ ] **D2** Enemy missiles, ABMs, explosions, the tap marker and the crosshair are always drawn **in front of** every building, base, mountain and the glow.
- [ ] **D3** Readability: red enemies, orange explosions and the magenta smart bomb stand out against the sky and mountains at a small window size and at a very wide window (the backdrop reaches every edge at every aspect ratio).
- [ ] **D4** The ammo numbers stay readable; the base muzzle still looks like the launch point.
- [ ] **D5** The six cities look different from each other and nothing pops or flickers when cities are rebuilt.
- [ ] **D6** No gameplay change: waves, scoring, flow, pause, quit to title, flyers, smart bombs and MIRVs behave as before.

### D-DEVICE. Android (NEEDS-DEVICE)
- [ ] **T-1** The horizon glow appears as a soft glow (not a solid or black box), which shows that additive materials work on the device.
- [ ] **T-2** The sky gradient, stars, mountains, skylines and bases render correctly; no flicker or shimmer on the stars or the window pattern.
- [ ] **T-3** The frame rate is not noticeably lower than before this phase.

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped; the editor is on `L_Game`; nothing under `Spike` was touched.

---

## 8. Report format (the agent's final message)

```
## Phase 9b report
**Engine version:** x.y.z
**MCP tools used:** (list)

### Step 0 findings
(mesh sizes of Cube, Sphere, Cylinder, Cone, Plane and pivot positions; pivot offsets applied (Q2); camera ortho clip settings;
 node class names found; fallbacks applied; attach support via set_parent_component yes/no (Q1); propagate pins before the edit;
 carrier flags before and after (Q4); parent of every component; AmmoText parent and world transform before and after,
 and whether LaunchPos is a component or a variable (Q6))

### Assets created or changed
(table: path | type | change | compiled clean)

### Materials
(table: material | blend mode | expression count | parameters read back | notes)

### Component inventory
(table: Blueprint | component | mesh | size | centre | material | bVisible | bHiddenInGame | collision flag; include the four carriers)

### Function and graph read-backs
(ApplyStyle, InitCity, ApplyVisual in both Blueprints: what the graph looked like before, which nodes or pins changed)

### Call audit (rule 12)
(table: function | call | pin | wired from / value)

### Acceptance results
A1 PASS/FAIL ... through E1 PASS/FAIL/NEEDS-HUMAN/NEEDS-DEVICE

### Log excerpt
(all [DD] lines of the run, with timestamps)

### Screenshot notes
(paths of the captures and a description of each: C1 to C7)

### Tool problems (errors, retries, editor restarts, writes that read back empty)
### Deviations
(fallbacks used; which star and window variant was used and the final expression count of each material (Q5); anything else)

### Open questions / risks
```

---

## 9. What this phase hands to later phases

| Provided | How a later phase uses it |
|---|---|
| The material instance parameters of 3.5 and 3.4 | Per-wave or per-theme palettes by setting `MI_DD_Sky.TopColor` / `HorizonColor`, `MI_DD_Glow.Color`, `MI_DD_Ground` colours at runtime through dynamic instances. |
| `BP_Backdrop` | The place to add a moon, clouds or parallax later. |
| The carrier pattern (`Body` / `Rubble` as mesh-less carriers, `ApplyVisual` with propagate) | Any later art swap of cities and bases needs no logic change. |
| `ApplyStyle` | A hook for later per-city looks. |
| The title-screen capture method | The same verification view for 9c to 9e. |

**Known leftovers that stay:** `StopSpawner`, `bDirectorSilent`, the two disconnected spawn chains in `SpawnRandomEnemy`, HUD debug lines 2 to 4, the overlay `BP_PlayfieldDebug`, the debug keys, the `Spike` folder (removed in 9g).

---

## 10. Common pitfalls

1. **Hidden in Game left on** (the invisible smart bomb). `SetVisibility(true)` never clears it. Every new component **and the carriers `Body` and `Rubble`**: `bHiddenInGame` false, read back.
2. **`Propagate To Children` left false** in the two `ApplyVisual` functions: the art would stay visible after a city is destroyed. Set it true and read it back.
3. **Art parts attached to the wrong carrier:** healthy parts under `Body`, rubble parts under `Rubble`. A part under the wrong one shows in the wrong state.
4. **Scale inherited from the old cube:** `Body` and `Rubble` must be reset to scale (1, 1, 1) when their mesh is removed, or every child is squashed.
5. **Scales computed from guessed mesh sizes.** Use the sizes measured in Step 0; cylinders and cones are not 100 tall.
6. **A part sticking out in front** (front face beyond carrier Y + 8): enemy missiles would be hidden behind buildings.
7. **Dome showing below the ground line:** the dome is centred at Z = 28 with height 56 so that its bottom is at 0; the podium hides the lower half.
8. **Touching gameplay geometry:** ground top (Z = 100), launch height (Z = 170), `AmmoText`, slot spacing. None of these may move.
9. **Sky not visible in PIE:** check the camera's ortho clip settings in Step 0 (the sky is about 3120 from the camera; the ground at 3000 already renders). Report the values; do not change the camera.
10. **A Noise node, a texture sample or a Time node** in a backdrop material: not allowed (mobile cost).
11. **Backdrop colours too bright or too warm:** no channel above 0.35, no red, orange or magenta hue (3.4).
12. **A Select used to guard a division:** none is needed in this phase (the gradient divisors are material constants); if a division by `Z1 - Z0` is built, keep `Z1 > Z0`.
13. **Editing `Ground` in the level** (only allowed under the `MI_DD_Ground` re-parent fallback) or any frozen asset.
14. **Marking D or T items PASS.** You cannot start a game, press keys or tap. They are NEEDS-HUMAN and NEEDS-DEVICE.
15. **Touching `Spike` or cleaning up what this document does not list.**
16. **Re-parenting moved a part (Q1).** The relative location can change when `set_parent_component` runs. Read the parent and the relative transform back after each re-parent and correct the location.
17. **`AmmoText` or the launch point moved (Q6).** Setting `Body` to location (0, 60, 0) and scale 1 moves anything attached to it. Read the parents first and keep the world transforms.
18. **Pivot not at the visual centre (Q2).** A cone, cylinder or plane may not have its pivot at its centre. Offset the location, and verify the mountains by arithmetic.
19. **Over the 60-expression cap (Q5, Q7).** Do not stop or ask. Use the over-budget rule of 3.5, and keep the star inset.

---

## 11. Clarifications given during the build (index)

These answers to the agent's questions are now written into the sections named below. The Q numbers appear in the text.

| Q | The agent asked | Decision | Where |
|---|---|---|---|
| **Q1** | Does `set_parent_component` count as attach support, or does the missing parent argument on `add_component` force Fallback F1? | It counts. Build `CityArt` and `ApplyStyle`. After each re-parent read back the parent and the relative transform and correct the location. F1 only if the parent cannot be set or does not read back. | 5 (note 3), Step 0.6, Steps 3 and 4, F1, pitfall 16 |
| **Q2** | If a mesh pivot is not at its visual centre, should the location be offset? | Yes: offset the location (scale and, for planes, roll included) so the visual centre lands on the table value. Verify the mountains by arithmetic (base Z = 80, apex 80 + H, within 1 unit). List every offset. | 3.6, Step 0.3, A4, pitfall 18 |
| **Q3** | Do the two plane numbers mean world width (X) and world height (Z)? | Yes. Roll 90, flat axis at the native thickness, scale = target / measured size on each used axis. | 3.6 |
| **Q4** | May a carrier's `bHiddenInGame` be set false? | Yes, and it must be: `bHiddenInGame` false and `bVisible` true on `Body` and `Rubble` in both Blueprints, with before and after recorded. | Rule 11, 3.7, 3.8, 4, Steps 3 and 4, A5 to A7 |
| **Q5** | What if every node exists but a material goes past 60 expressions? | Do not stop. (a) simplify the stars to one hash; (b) if still over 60, drop the stars; (c) windows keep the lit / unlit hash if it fits, otherwise the regular grid. Record the step and the final count. | 3.5 (over-budget rule), A2, pitfall 19 |
| **Q6** | If `AmmoText` or `LaunchPos` is attached to `Body`, should their relative transforms be adjusted? | Their world positions must not change. Read the parents first. Root-attached: nothing to do. Attached to `Body` or `Rubble`: re-attach to the root with the same world transform, or adjust the relative transform. `LaunchPos` is probably a variable in `InitBase`: leave it. Report what was found and done. | 3.7, 3.8, Step 0.1, Step 4, A6, A12, pitfall 17 |
| **Q7** | Should the simplified star centre keep the 0.2 / 0.6 inset? | Yes: `(0.2 + 0.6 * frac(h1 * 17), 0.2 + 0.6 * frac(h1 * 91))`. Without it a star can be cut in half on a cell edge. | 3.5 (over-budget rule, step a) |
