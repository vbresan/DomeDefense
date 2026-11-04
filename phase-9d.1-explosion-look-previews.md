# Phase 9d.1 — Explosion Look Previews (Six Variations, Keys 0 to 7)

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Depends on** | Phase 9c.3 (built and committed; the repository `master` at commit "Phase 9c.3"). |
| **Phase goal** | **A look-development tool, not a game change.** A new preview actor draws **six explosion looks** at the exact size and timing of the current explosion, so the human can press keys on the desktop, compare them with the current look, and pick the winner. The looks are **A, B, C**, each in an **orange** and a **cyan-white** colour scheme. **The real explosions, the kill logic and the whole game stay exactly as they are**: a preview never kills, never scores and never registers as an explosion. A later phase (9d.2) wires the winner into the real explosions. |

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **No gameplay change.** The preview actor is **not** registered in `ActiveExplosions`, never calls anything in the game mode, never changes score, ammo, damage or waves, and has no collision. `BP_Explosion` is **read only**.
3. **HANDS OFF.** Do not change, and report as unchanged after the work: `BP_Explosion` (everything), `BP_DDGameMode`, `BP_DDHUD`, `BP_EnemyMissile`, `BP_ABM`, `BP_Flyer`, `BP_TapMarker`, `BP_City`, `BP_MissileBase`, `BP_Crosshair`, `PDA_DDConfig`, `DA_DDConfig`, `L_Game`, **every existing material and material instance** (including `MI_DD_Explosion`), and everything in `BP_DDPlayerController` **except** the additions of Step 3 (three new functions and eight new key events).
4. **NO SCREENSHOTS.** The human has ruled out capturing and describing images. **Do not use `CaptureViewport` or `CaptureEditorImage`, and do not place temporary actors in the level.** Verify only by compiling and by reading assets back.
5. **Existing logic must survive.** Read `BP_DDPlayerController` (`read_graph_dsl`) before editing it.
6. **Compile after every Blueprint or material change.** Every asset ends with no errors and no warnings.
7. **Run each step's "Check"** before moving on. **Save All after each step's Check passes.**
8. **You cannot press keys, click, touch, move the mouse, or resize in PIE, and you cannot start a game.** You therefore cannot spawn a preview or see one. Everything seen in play is group D (**NEEDS-HUMAN**). Never mark a D item PASS.
9. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
10. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after the run. **Save before loading a level.** **Do not touch Project Settings, input settings or any `Config/*.ini` file.** **Do not use, edit or delete anything in `/Game/DomeDefense/Spike/`.**
11. **NEVER REWRITE AN EXISTING FUNCTION OR EVENT GRAPH** of `BP_DDPlayerController` with `write_graph_dsl`. New functions are written once with `write_graph_dsl` and read back. Key events are created with `create_node` and `connect_pins` (key event nodes cannot be written in DSL; find the node type with `find_node_types`, query for example "One" or "Zero"); only the `Pressed` pin is connected.
12. **READ EVERY NEW OR CHANGED GRAPH BACK** right after writing it. If it reads back wrong or incomplete: correct it once more; if it is still wrong, **stop and report** under *Tool problems*.
13. **CHECK EVERY PIN YOU BUILD** (the lesson of the Phase 7 pause defect: a pin left at its default). After building the spawn function, the eight key calls and the preview graphs, read all input pins of every call back and list them (A9). Literal integer arguments must be wired explicitly.
14. **VISIBILITY AUDIT.** Every new visual component has **`bHiddenInGame` FALSE**, `bVisible` TRUE as its Blueprint default (the preview hides what its look does not use, in `BeginPlay`), collision `NoCollision`, `CastShadow` false, and is **Movable**. Read the three flags back for every new component and list them (A8).
15. **No cheap shortcuts in materials:** no Noise node, no Custom (HLSL) node, no texture, no Niagara. Only basic math, sine/cosine, length, arctangent2, lerp, saturate, power, texture coordinate, scalar and vector parameters. (The effects must stay cheap on Android.)
16. **After any tool error, do not retry the same large operation.** Stop that item, try one smaller piece at most, and if it still fails **stop and report**. **If the editor stops responding or restarts, stop and report.**
17. **Order is part of the spec:** read first, then materials, then the preview actor, then the controller, then the run.
18. **FROZEN: do not edit** every asset not named in rule 3's exception or created by this document.
19. **No new config values, no new log lines, no new keys other than the eight of 3.8.**
20. **If the project is not in the state described in "Depends on"** (an asset name created by this document already exists, a key of 3.8 is already bound, `CrosshairWorldPos` is missing or is not a vector), **stop and report** before changing anything.

---

## 2. Scope

### In scope
- Five new materials and five material instances (3.4).
- One new Blueprint `BP_ExplosionPreview` (3.2, 3.3, 3.5, 3.6).
- `BP_DDPlayerController`: three new functions and eight new key events (3.8).

### Out of scope (do NOT do)
- Any change to the real explosion, its radius, its timing, its colour, its kill logic, or to which explosions are player made.
- The colour split for the real explosions (a later phase wires the winner).
- Sound, screen shake, any HUD change.

---

## 3. Design (decided; do not change)

### 3.1 What the human gets
Keys on the desktop (they work on the title screen and in a game, like the other debug keys). Keys 1 to 6 spawn at the **mouse crosshair** (`CrosshairWorldPos`).

| Key | Look | Colour scheme |
|---|---|---|
| `Zero` | **0 = Reference**: the current plain disc (same component settings and material as `BP_Explosion`'s `Disc`) | orange (its own material) |
| `One` | **A** "Neon classic": a glowing disc, white-hot core | orange |
| `Two` | **B** "Fireball": turbulent fire with a flash | orange |
| `Three` | **C** "Fireball with embers": B plus 8 flying sparks | orange |
| `Four` | A | cyan-white |
| `Five` | B | cyan-white |
| `Six` | C | cyan-white |
| `Seven` | all six at once in a 3 x 2 grid at the centre of the screen: orange A, B, C on the top row, cyan-white A, B, C on the bottom row | both |

Size and timing are exactly those of the real explosion: maximum radius **160**, grow **0.4 s**, hold **0.4 s**, shrink **0.5 s** (1.3 s in total). **Every visible pixel of every look stays inside the circle of radius 160 around the explosion's centre.**

### 3.2 `BP_ExplosionPreview`: variables
Actor Blueprint, `Root` is a SceneComponent, Tick enabled.

| Variable | Type | Default | Flags |
|---|---|---|---|
| `Look` | integer | 0 | Instance Editable, **Expose on Spawn** (0 reference, 1 A, 2 B, 3 C) |
| `Scheme` | integer | 0 | Instance Editable, **Expose on Spawn** (0 orange, 1 cyan-white) |
| `MaxRadius` | float | 160.0 | Instance Editable, category `Timing` |
| `GrowTime` | float | 0.4 | Instance Editable, `Timing` |
| `HoldTime` | float | 0.4 | Instance Editable, `Timing` |
| `ShrinkTime` | float | 0.5 | Instance Editable, `Timing` |
| `FlashTime` | float | 0.15 | Instance Editable, `Timing` |
| `OrangeCore`, `OrangeMid`, `OrangeEdge` | linear colour | (1.0, 0.92, 0.70), **the `Color` of `MI_DD_Explosion` (Step 0)**, (0.45, 0.08, 0.01) | Instance Editable, category `Colors` |
| `CyanCore`, `CyanMid`, `CyanEdge` | linear colour | (1.0, 1.0, 1.0), (0.45, 0.90, 1.0), (0.05, 0.35, 0.75) | Instance Editable, `Colors` |
| `Age` | float | 0.0 | |
| `Life` | float | 0.0 | |
| `PalCore`, `PalMid`, `PalEdge` | linear colour | black | the palette chosen by `Scheme` |

`MaxRadius`, `GrowTime`, `HoldTime` and `ShrinkTime` defaults must equal the values in `DA_DDConfig` / `PDA_DDConfig` (`ExplosionMaxRadius` 160, `ExplosionGrowTime` 0.4, `ExplosionHoldTime` 0.4, `ExplosionShrinkTime` 0.5); read them in Step 0 and use the values you read.

### 3.3 `BP_ExplosionPreview`: components
All are children of `Root` unless stated, relative rotation 0 unless stated. A **Plane** layer has rotation **Roll 90** (it faces the camera, exactly as the missile halos do) and its location Y is the layer depth (a larger Y is nearer to the camera). Scales assume the engine Plane is 100 x 100 and the Sphere 100 across (measure both in Step 0; if different, divide by the measured size).

| Component | Mesh | Material | Location (X, Y, Z) | Initial scale | Used by looks |
|---|---|---|---|---|---|
| `RefDisc` | Sphere | `MI_DD_Explosion` (the existing instance) | (0, 90, 0) | as `Disc` in `BP_Explosion` (Step 0) | 0 |
| `LayerGlow` | Plane | `MI_DD_ExpGlow` | (0, 89, 0) | (0.01, 0.01, 1) | 1, 2, 3 |
| `LayerBody` | Plane | `MI_DD_ExpDisc` | (0, 90, 0) | (0.01, 0.01, 1) | 1 |
| `LayerFire` | Plane | `MI_DD_ExpFire` | (0, 90, 0) | (0.01, 0.01, 1) | 2, 3 |
| `LayerFlash` | Plane | `MI_DD_ExpFlash` | (0, 91, 0) | (0.01, 0.01, 1) | 2, 3 |
| `EmberGroup` | SceneComponent | none | (0, 0, 0) | 1 | 3 |
| `Ember1` ... `Ember8` | Sphere | `MI_DD_ExpEmber` | (0, 92, 0) | (0.01, 0.01, 0.01) | 3 (children of `EmberGroup`) |

All locations above are the starting values; the tick moves and scales the layers (3.5).

### 3.4 Materials
All five materials: Material Domain **Surface**, Shading Model **Unlit**, Blend Mode **Additive**, **Two Sided** on, used with static meshes. For every one, define the plane coordinate `p = TexCoord[0] - (0.5, 0.5)` and the disc radius `r = Length(p) * 2` (0 at the centre, 1 at the middle of each plane edge; the corners lie outside r = 1). **Each material's emissive output must be exactly zero for r >= 1**, so nothing is ever drawn beyond the radius of 160. Any equivalent node arrangement that computes the same values is fine. Use these names exactly.

Common colour ramp `Ramp(h)` used by Disc and Fire: `c1 = saturate(h / 0.45)`, `c2 = saturate((h - 0.45) / 0.55)`, `Ramp = lerp( lerp(EdgeColor, MidColor, c1), CoreColor, c2 )`.

**`M_DD_ExpDisc`** (look A). Scalars `T` 0, `Fade` 1, `Intensity` 1.2, `Softness` 0.30, `CoreFalloff` 1.6; vectors `CoreColor`, `MidColor`, `EdgeColor`.
- `mask = saturate((1 - r) / Softness)`
- `h = pow(saturate(1 - r), CoreFalloff)`
- `Emissive = Ramp(h) * Intensity * mask * Fade`

**`M_DD_ExpGlow`** (looks A, B, C; the soft glow behind). Scalars `Fade` 1, `GlowIntensity` 0.6, `GlowPower` 2.0; vector `MidColor`.
- `Emissive = MidColor * GlowIntensity * pow(saturate(1 - r), GlowPower) * Fade`

**`M_DD_ExpFire`** (looks B, C). Scalars `T` 0, `Fade` 1, `Seed` 0, `Intensity` 1.3, `Turb` 0.6, `Cool` 0.55, `Flow` 1.0, `Rag` 0.22; vectors `CoreColor`, `MidColor`, `EdgeColor`.
- `q = p * 2`, `Tf = T * Flow`, `ang = ArcTangent2(p.y, p.x)`
- turbulence: `n1 = sin(q.x*4.1 + q.y*2.3 + Seed + Tf*18)`, `n2 = sin(q.y*5.3 - q.x*1.7 - Seed*1.37 - Tf*14)`, `n3 = sin((q.x + q.y)*8.7 + Seed*0.73 + Tf*26)`, `n = 0.5 + 0.5*(0.5*n1 + 0.3*n2 + 0.2*n3)`
- ragged edge: `nAng = 0.5 + 0.5*(0.6*sin(ang*5 + Seed + Tf*8) + 0.4*sin(ang*9 - Seed*1.9 - Tf*11))`, `D = (1 - Rag) + Rag*nAng` (always between 1 - Rag and 1), `rr = r / D`
- `base = saturate(1 - rr)`; `h = saturate( saturate(base*1.25 + (n - 0.5)*Turb) - Cool*T )`
- `m = saturate(h*3.0) * saturate((1 - rr)*4.0)`
- `Emissive = Ramp(h) * Intensity * m * Fade`
(The integer multipliers of `ang` keep the pattern continuous at the angle seam. Because `rr >= r` and `m` is zero for `rr >= 1`, nothing is drawn beyond r = 1.)

**`M_DD_ExpFlash`** (looks B, C; a short white flash). Scalars `Fade` 1, `FlashIntensity` 1.5, `FlashPower` 1.5; vector `CoreColor`.
- `Emissive = CoreColor * FlashIntensity * pow(saturate(1 - r), FlashPower) * Fade`

**`M_DD_ExpEmber`** (look C; the sparks, drawn on small spheres). Scalars `Fade` 1, `EmberIntensity` 1.5; vector `CoreColor`.
- `Emissive = CoreColor * EmberIntensity * Fade`

Create the five instances `MI_DD_ExpDisc`, `MI_DD_ExpGlow`, `MI_DD_ExpFire`, `MI_DD_ExpFlash`, `MI_DD_ExpEmber` (parents the matching materials) with the scalar defaults above and these default colours (the preview overwrites the vectors at run time with its palette; the defaults are the orange ones): `CoreColor` (1.0, 0.92, 0.70), `MidColor` (the `Color` of `MI_DD_Explosion`), `EdgeColor` (0.45, 0.08, 0.01).

### 3.5 `BP_ExplosionPreview`: logic
Four functions and two events. Write each new function once with `write_graph_dsl`, read it back.

**Function `PushPalette(Comp: StaticMeshComponent)`**: `Comp.SetVectorParameterValueOnMaterials("CoreColor", PalCore)`, then `"MidColor"` with `PalMid`, then `"EdgeColor"` with `PalEdge`. (Parameters a material does not have are ignored.)

**Function `UpdateEmber(Comp: StaticMeshComponent, Index: float, T: float)`**:
```
theta = Index * 2.39996                         // radians, the golden angle
k     = 0.55 + 0.40 * Frac(Index * 0.618034)
u     = Clamp(T / 0.8, 0, 1)
d     = 1 - (1 - u) * (1 - u)
rho   = MaxRadius * 0.9 * k * d                 // never more than 0.9 * 0.95 * 160
size  = (0.07 + 0.05 * Frac(Index * 0.37)) * Clamp(1 - T, 0, 1)
Comp.SetRelativeLocation( MakeVector(rho * Cos(theta), 92.0, rho * Sin(theta)) )   // Cos and Sin take radians
Comp.SetRelativeScale3D( MakeVector(size, size, size) )
```
(Largest reach: rho 137 plus half a sphere of 6 is under 143.)

**Function `UpdateEmbers(T: float)`**: eight calls `UpdateEmber(Ember1, 0, T)` ... `UpdateEmber(Ember8, 7, T)` (the indices are literal floats 0 to 7, wired explicitly).

**Function `UpdateLooks()`**:
```
g     = the radius fraction at Age, 0..1: use EXACTLY the rule BP_Explosion's tick uses for its radius
        (read in Step 0). Fallback only if that rule cannot be read:
        Select(Age < GrowTime, Age / GrowTime,
               Select(Age < GrowTime + HoldTime, 1.0,
                      1 - (Age - GrowTime - HoldTime) / ShrinkTime)), clamped to 0..1
R     = MaxRadius * g
S     = R * 0.02                                 // a 100-unit mesh scaled by S has a diameter of 2R
T     = Clamp(Age / Life, 0, 1)
Fade  = Select(Age < GrowTime + HoldTime, 1.0, Clamp(1 - (Age - GrowTime - HoldTime) / ShrinkTime, 0, 1))
FlashFade = Clamp(1 - Age / FlashTime, 0, 1)
RefDisc.SetRelativeScale3D( the scale rule BP_Explosion uses for Disc, from Step 0 (expected (S, 0.02, S)) )
LayerGlow, LayerBody, LayerFire.SetRelativeScale3D( (S, S, 1) )
LayerFlash.SetRelativeScale3D( (S * 0.7, S * 0.7, 1) )
for LayerGlow, LayerBody, LayerFire:  SetScalarParameterValueOnMaterials("T", T) and ("Fade", Fade)   // Glow has no T: ignored
LayerFlash:                           SetScalarParameterValueOnMaterials("Fade", FlashFade)
UpdateEmbers(T)
```

**Event `BeginPlay`**:
```
Life = GrowTime + HoldTime + ShrinkTime
PalCore = Select(Scheme == 0, OrangeCore, CyanCore); PalMid = Select(Scheme == 0, OrangeMid, CyanMid); PalEdge = Select(Scheme == 0, OrangeEdge, CyanEdge)
RefDisc.SetVisibility(Look == 0, false)
LayerBody.SetVisibility(Look == 1, false)
LayerFire.SetVisibility(Look >= 2, false)
LayerFlash.SetVisibility(Look >= 2, false)
LayerGlow.SetVisibility(Look >= 1, false)
EmberGroup.SetVisibility(Look == 3, true)       // propagate to children: TRUE
PushPalette(LayerBody), PushPalette(LayerFire), PushPalette(LayerFlash), PushPalette(LayerGlow), PushPalette(Ember1) ... PushPalette(Ember8)
LayerFire.SetScalarParameterValueOnMaterials("Seed", RandomFloatInRange(0.0, 100.0))
UpdateLooks()
```
The reference disc keeps its own material and gets no palette.

**Event `Tick(DeltaSeconds)`**: `Age = Age + DeltaSeconds`; `Branch(Age >= Life)`: True → `DestroyActor`; False → `UpdateLooks()`.

### 3.6 Geometry notes (for the report)
- All layers are centred on the actor, so the explosion is centred on the spawn point.
- Depth order (larger Y is nearer to the camera): glow 89, disc/fire/reference 90, flash 91, embers 92. The real explosion is at 90 as well, so previews draw over missiles (up to 84.5) and under the target marker (106 and up), as real explosions do.
- Because all materials are additive, overlapping previews add up, which is intended.

### 3.7 Tuning knobs (for the human; list them in the report as a table)
| Where | Knob | Effect |
|---|---|---|
| `MI_DD_ExpDisc` | `Intensity`, `Softness`, `CoreFalloff` | brightness; how soft the rim is; how small the white-hot core is |
| `MI_DD_ExpGlow` | `GlowIntensity`, `GlowPower` | glow brightness; how tight the glow is |
| `MI_DD_ExpFire` | `Intensity`, `Turb`, `Cool`, `Flow`, `Rag` | brightness; how lumpy; how fast it cools to red; how fast the turbulence moves; how ragged the outline is |
| `MI_DD_ExpFlash` | `FlashIntensity`, `FlashPower` | flash brightness; flash size |
| `MI_DD_ExpEmber` | `EmberIntensity` | spark brightness |
| `BP_ExplosionPreview` defaults | `Orange*`, `Cyan*`, `FlashTime` | the palettes; flash duration |

### 3.8 `BP_DDPlayerController`: additions
Three new functions (written once, read back):
- `SpawnExplosionPreview(Look: integer, Scheme: integer, X: float, Z: float)`: `SpawnActor` of class `BP_ExplosionPreview` at the location `(X, 0, Z)`, collision handling **AlwaysSpawn**, with the exposed pins `Look` and `Scheme` wired from the arguments.
- `SpawnPreviewAtCrosshair(Look: integer, Scheme: integer)`: `BreakVector(CrosshairWorldPos)` then `SpawnExplosionPreview(Look, Scheme, X, Z)`.
- `SpawnPreviewGrid()`: six calls of `SpawnExplosionPreview(Look, Scheme, X, Z)` with literal arguments: (1, 0, -500, 1150), (2, 0, 0, 1150), (3, 0, 500, 1150), (1, 1, -500, 650), (2, 1, 0, 650), (3, 1, 500, 650).

Eight new key events (`create_node`, only `Pressed` connected), each calling one function with **literal, explicitly wired arguments**:

| Key | Call |
|---|---|
| `Zero` | `SpawnPreviewAtCrosshair(0, 0)` |
| `One` | `SpawnPreviewAtCrosshair(1, 0)` |
| `Two` | `SpawnPreviewAtCrosshair(2, 0)` |
| `Three` | `SpawnPreviewAtCrosshair(3, 0)` |
| `Four` | `SpawnPreviewAtCrosshair(1, 1)` |
| `Five` | `SpawnPreviewAtCrosshair(2, 1)` |
| `Six` | `SpawnPreviewAtCrosshair(3, 1)` |
| `Seven` | `SpawnPreviewGrid()` |

---

## 4. What this phase changes in earlier specs

| Earlier text | Now |
|---|---|
| The only explosion look is the plain disc of `BP_Explosion` | Unchanged for the game. Six preview looks exist beside it, reachable only by the debug keys `0` to `7`. |
| Debug keys: `H`, `R`, `E`, `K`, `X`, `V`, `N`, `B`, `P`, `Delete`, `M`, `F`, `G`, `S` | Plus `Zero` to `Seven` (removed in 9g with the others). |
| 9c.3 frozen list | `BP_DDPlayerController` is unfrozen for the additions of 3.8 only. |

---

## 5. Tooling notes

1. Materials: use the material tools used in Phase 9b and 9c for `M_DD_Halo` and `M_DD_Tail`. Set the blend mode, shading model and two-sided flag on each material and read them back. Compile each material and read the compile result; any error in a material is a stop-and-fix item.
2. Components on a Blueprint: `ActorTools.add_component` (engine meshes by path), then `set_parent_component` (for the embers under `EmberGroup`); **after each re-parent read back the parent and the relative transform** and correct it if the tool changed it.
3. `SetScalarParameterValueOnMaterials` and `SetVectorParameterValueOnMaterials` are nodes of mesh components (they create the dynamic instances themselves). If they cannot be found with `find_node_types`, **stop and report**; do not invent another way.
4. `Look`/`Scheme` must be both **Instance Editable** and **Expose on Spawn** so the spawn node shows the two pins; compile the preview Blueprint before building the controller functions.
5. `StaticMeshTools.get_bounds` measures an engine mesh. `CastShadow` (not `bCastShadow`) is the shadow property.
6. If a tool reports that a property cannot be read or set, say so; do not guess.

---

## 6. Implementation steps

### Step 0 — Read and measure (no changes)
1. Measure `/Engine/BasicShapes/Plane` and `/Engine/BasicShapes/Sphere` (size at scale 1, bounds centre).
2. `BP_Explosion`: components of `Disc` (mesh, material, location, scale, flags); read the tick logic completely and record (a) the exact rule that computes the radius from `Age`, `GrowTime`, `HoldTime`, `ShrinkTime` and `MaxRadius` (or `PeakRadius`), (b) the exact scale it sets on `Disc` from the radius. Read `MaxRadius`/`PeakRadius` and the three times from where they come from (`DA_DDConfig`, `PDA_DDConfig`) and record the values.
3. `MI_DD_Explosion`: parent material, blend mode, and every parameter with its value (record the colour parameter's name and value).
4. `BP_DDPlayerController`: list all key events (expected: Android_Back, B, Delete, E, F, G, H, K, LeftMouseButton, M, N, P, R, S, V, X; **none of Zero to Seven**); confirm `CrosshairWorldPos` exists, its type is a vector, and where it is updated (also while the title screen is shown). Read the existing key-event wiring pattern (for example `F`) to copy it.
5. Confirm none of the asset names of section 3 exist yet.

**Check:** the report section for Step 0 is complete.

### Step 1 — Materials
Create the five materials and the five instances of 3.4. Read back for each material: blend mode, shading model, two-sided, the parameter names and defaults, and the compile result; and **state, for each material, why the emissive is exactly zero for r >= 1** (the factor that makes it zero).

**Check:** all ten assets exist and compile clean; parameters as specified; no Noise, Custom or texture node in any material. **Save All.**

### Step 2 — `BP_ExplosionPreview`
1. Create the Blueprint with the variables of 3.2 and the components of 3.3 (Plane and Sphere sizes from Step 0). Read back every component (parent, transform, mesh, material, `bVisible`, `bHiddenInGame`, `Mobility`, collision, `CastShadow`).
2. Write the functions `PushPalette`, `UpdateEmber`, `UpdateEmbers`, `UpdateLooks` and the events `BeginPlay` and `Tick` (3.5). Read each back.
3. Compile.

**Check:** compiles clean; components and variables read back as specified; `Look` and `Scheme` are Instance Editable and Expose on Spawn; the graphs read back as specified; every call pin is listed (A9). **Save All.**

### Step 3 — `BP_DDPlayerController`
Read the controller again first.
1. Write the three functions of 3.8 and read them back.
2. Create the eight key events with `create_node`, connect only `Pressed`, wire the calls with the literal arguments of 3.8. Read each back.
3. Compile.

**Check:** the controller compiles clean; every existing key event and function reads back exactly as before; the new pieces read back as specified; every call pin is listed (A9). **Save All.**

### Step 4 — Run
1. **Save All.**
2. `StartPIE`. The title screen appears at once. **Wait about 20 seconds**; nobody presses anything, so no preview and no game starts. **No screenshots.**
3. `GetLogEntries` (filter `[DD]`), then `StopPIE`.
4. Fill in the report (section 8).

---

## 7. Acceptance tests

### A. Static checks
- [ ] **A1** New assets: five materials, five instances, `BP_ExplosionPreview`. Changed: `BP_DDPlayerController`. Nothing else: no existing material or instance, no `BP_Explosion`, nothing under `Spike`, no Project Settings, input settings or `.ini` change. `L_Game` unchanged, and no temporary actor was ever placed.
- [ ] **A2** All new and changed assets compile with 0 errors and 0 warnings.
- [ ] **A3** Step 0 findings are in the report: mesh measurements, the radius rule and the `Disc` scale rule of `BP_Explosion`, the values of the config entries, `MI_DD_Explosion`'s parameters, the controller's key events, `CrosshairWorldPos`.
- [ ] **A4** Materials: blend mode Additive, Unlit, two-sided, parameters and defaults as in 3.4; for each material the reason why its emissive is exactly zero for r >= 1; no Noise, Custom or texture node.
- [ ] **A5** `BP_ExplosionPreview` variables as in 3.2 (defaults, flags) and components as in 3.3.
- [ ] **A6** The preview graphs read back as specified in 3.5; the radius rule in `UpdateLooks` is the one of `BP_Explosion` (recorded), and the timing defaults equal the config values.
- [ ] **A7** The controller: three new functions and eight new key events as in 3.8; all existing key events and functions unchanged.
- [ ] **A8** **Visibility audit:** a table of every new component with `bVisible`, `bHiddenInGame`, `Mobility`, collision and `CastShadow`: `bHiddenInGame` false, Movable, `NoCollision`, `CastShadow` false.
- [ ] **A9** **Pin audit (rule 13):** a table of every input pin of every call in the new graphs and key events (target, name/value pins, new visibility, propagate, spawn pins `Look`/`Scheme`, collision handling, literal arguments), each with where it is wired from or its value.
- [ ] **A10** **Unchanged (rule 3):** a table that shows, for each item of rule 3, that it reads exactly as before, including that nothing in the new graphs references `ActiveExplosions`, the game mode or the score.
- [ ] **A11** The tuning table of 3.7 is in the report.

### B. Log checks (agent; the one 20 s run, nobody presses anything)
- [ ] **B1** The startup lines as in the 9c.3 run: camera, GameMode, config (`Width=2250.0 Height=1800.0`), bases 3, cities 6, PlayerController, view target, crosshair, HUD, `[DD][Area] Viewport=...`.
- [ ] **B2** Exactly one of `[DD][Save] Loaded high=<n>` or `[DD][Save] No file high=<n>`, followed by exactly one `[DD][Flow] Title high=<n>`.
- [ ] **B3** During the 20 s wait: no `Spawner started`, no `[DD][Wave]`, `[DD][Enemy]`, `[DD][Flyer]`, `[DD][Score]`, `[DD][Damage]`, `[DD][Explosion]`, `[DD][Input] Fire request`, `[DD][Aim]`, `[DD][Game]`, and no `[DD][Flow]` line except the one `Title` line.
- [ ] **B4** No line containing `ERROR`, `Accessed None`, `Divide by zero` or `Blueprint Runtime Error`.

### D. Interactive checks (human, PIE, desktop) — all NEEDS-HUMAN
List every item as NEEDS-HUMAN in the report; do not run or claim them.
- [ ] **D1** Keys `0` to `6` each spawn the stated look at the mouse crosshair; `7` spawns the 3 x 2 grid; nothing happens on other number keys.
- [ ] **D2** **Size and timing:** every look reaches the same size as the reference (key `0`) and as a real explosion; nothing is drawn outside the reference circle (embers included); grow, hold and fade take 1.3 s in total; the preview then disappears completely.
- [ ] **D3** **The six looks:** A is a glowing disc with a white-hot core; B is a turbulent fireball with a short white flash that cools toward red and fades; C is B with eight sparks flying out and shrinking away; the orange set is orange, the cyan-white set is cyan-white. (Which one is best is the human's decision.)
- [ ] **D4** **No gameplay effect:** a preview does not kill missiles, does not score, does not hit flyers, does not change ammo; real explosions look and behave as before while previews are on screen.
- [ ] **D5** Several previews at once (key `7`, then key `1` repeatedly) run without a hitch and leave nothing behind.
- [ ] **D6** The tuning knobs of 3.7 work: changing a value on a material instance or on the preview Blueprint changes the next preview as described.
- [ ] **D7** No errors in the play log (`Blueprint Runtime Error`) after using the keys.

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped; the editor is on `L_Game`; nothing under `Spike` was touched.

---

## 8. Report format (the agent's final message)

```
## Phase 9d.1 report (explosion look previews)
**Engine version:** x.y.z
**MCP tools used:** (list)

### Step 0 findings
(mesh measurements; BP_Explosion radius rule and Disc scale rule; config values; MI_DD_Explosion parameters; the controller's key events and CrosshairWorldPos)

### Materials (Step 1)
(per material: settings, parameters with defaults, compile result, why emissive is zero for r >= 1; the five instances with their parameter values)

### BP_ExplosionPreview (Step 2)
(variables; component table with flags; the graphs as read back)

### BP_DDPlayerController (Step 3)
(the three functions and eight key events as read back)

### Pin audit (rule 13)
(table: graph | node | pin | wired from / value)

### Hands-off audit (rule 3)
(table: item | unchanged? | how it was checked)

### Tuning knobs (3.7)

### Acceptance results
A1 PASS/FAIL ... through E1 PASS/FAIL/NEEDS-HUMAN

### Log excerpt
(all [DD] lines of the run, with timestamps)

### Tool problems (errors, retries, editor restarts, writes that read back empty)
### Deviations
### Open questions / risks
```

---

## 9. What this phase hands to later phases

| Provided | How a later phase uses it |
|---|---|
| Six looks to compare, tunable by hand | The human picks one look and one colour scheme per kind of explosion. |
| Materials and layers that follow the real explosion's size and timing | Phase 9d.2 wires the winner into `BP_Explosion` (and may use the other scheme for enemy impacts, using the existing `bPlayerMade` flag), then removes the unused looks and the debug keys. |

**Known leftovers that stay (removed in 9g or 9d.2):** `BP_ExplosionPreview`, its materials and the keys `Zero` to `Seven`, the hidden cubes of `BP_Flyer`, the hidden `Dot` of `BP_TapMarker` and of the missiles, the old `UpdateSmartTail` (uncalled), `StopSpawner`, `bDirectorSilent`, the two disconnected spawn chains in `SpawnRandomEnemy`, the disconnected older fire chain in `LaunchAt`, HUD debug lines 2 to 4, `BP_PlayfieldDebug`, the debug keys, the unused `MI_DD_Enemy` and `MI_DD_Abm`, the `Spike` folder.

---

## 10. Common pitfalls

1. **Taking screenshots or placing temporary actors.** Not allowed in this phase.
2. **Letting a look draw beyond radius 160.** Every material is zero at r >= 1, the flash is 0.7 of the radius, and the embers stay under 143 units from the centre.
3. **Registering a preview as a real explosion** (adding it to `ActiveExplosions`, calling the game mode): it would kill missiles and change the game.
4. **A pin left at its default:** a key call with Look or Scheme at the default 0 shows the wrong look; the spawn node needs `Look` and `Scheme` wired and collision handling AlwaysSpawn.
5. **A radius curve that differs from the real explosion.** Use exactly the rule of `BP_Explosion` (Step 0).
6. **Hidden layers:** `BeginPlay` must show exactly the layers of the chosen look (rule 14); the embers are shown through the group with Propagate TRUE.
7. **Noise, Custom or texture nodes in a material** (rule 15).
8. **Changing `MI_DD_Explosion`** or any existing material: the reference disc must stay what the game uses.
9. **Rewriting an existing controller function or event graph** (rule 11). Only new functions and new key events.
10. **A key that is already bound.** Step 0 lists the key events; stop and report on a clash.
11. **Marking D items PASS.** You cannot press keys or see a preview. They are NEEDS-HUMAN.
12. **Touching `Spike`, `BP_Explosion` or any other frozen asset**, or cleaning up what this document does not list.
