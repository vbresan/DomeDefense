# Phase 0 — Foundations

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.x (use the version the project is already on; do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Phase goal** | An empty but fully wired project: map, camera, playfield, config asset, logging, and framework classes. No gameplay yet. |

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Do not add features, assets, nodes, or "improvements" that are not listed. Out-of-scope items are listed in section 2.
2. **Names are exact and case-sensitive.** Asset names, variable names, function names, and folder names must match this document character for character.
3. **Compile after every Blueprint change.** Every Blueprint must end with status "compiled, no errors, no warnings".
4. **Run the "Check" line at the end of each step** before moving to the next step.
5. **If something cannot be done** (a tool does not support an operation, a property has a different name in this engine version, an asset path does not exist):
   - If this document names a **Fallback**, use it and record it in the report under *Deviations*.
   - Otherwise, **stop that item**, do not invent an alternative, continue with any steps that do not depend on it, and record it under *Deviations*.
6. **Only touch** `/Game/DomeDefense/` and the project config entries named in this document.
7. **Do not commit to source control.** The human commits after reviewing.
8. **Do not leave the editor in Play-In-Editor (PIE).** Stop PIE after each test.
9. When values are given as `(X, Y, Z)`, they are **Unreal world units (cm)** in the world coordinate system defined in section 4.1.

---

## 2. Scope

### In scope
- Folder structure, naming, and shared conventions.
- One map: `L_Game`.
- Orthographic playfield camera, auto-configured from the config asset.
- Config asset holding all tunable gameplay numbers (values only; nothing reads most of them yet).
- Logging helper.
- Framework classes: GameMode, PlayerController, HUD.
- Debug overlay that draws the playfield bounds, ground line, axes, and the planned base and city positions.
- One unlit material and one material instance (ground).

### Out of scope (do NOT create)
- Crosshair, mouse input, click handling, Enhanced Input assets.
- Missile bases, ABMs, explosions, enemy missiles, cities (as gameplay actors).
- Waves, scoring logic, game-over logic.
- UMG widgets, sounds, particle effects, textures, imported art.
- Custom collision channels or presets.
- Packaging or build settings.

---

## 3. Prerequisites (done by the human before the agent starts)

- A UE 5.x project named `DomeDefense` (display name Dome Defense), created from the **Games → Blank** template, **Blueprint** (not C++), Starter Content **off**, Ray Tracing **off**.
- The editor is open on that project and the MCP connection works.
- The agent records the exact engine version in its report.

---

## 4. Global conventions (apply to ALL phases)

### 4.1 Coordinate system and units

| Item | Decision |
|---|---|
| Units | 1 Unreal unit = 1 cm. All speeds are units per second. All durations are seconds. |
| Play plane | The **XZ plane at Y = `PlayPlaneY` (0)**. All gameplay happens on this plane. |
| Screen right | **+X** |
| Screen up | **+Z** |
| Depth | Y. Gameplay actors are spawned at Y = 0 and **never move in Y**. |
| Camera | Orthographic, positioned at **+Y** looking toward **−Y** (rotation Roll 0, Pitch 0, **Yaw −90**). |
| Time | Movement is time-based (use `DeltaSeconds`, Timelines, or Timers). Never frame-count-based. |

If a `ProjectileMovementComponent` is used in a later phase: enable **Constrain To Plane**, set **Plane Constraint Normal = (0, 1, 0)**, and enable **Snap To Plane At Spawn**.

### 4.2 Playfield geometry (the single source of truth is `DA_DDConfig`)

```
Z=2400 +------------------------------------------------------------+
       |                                                            |
       |                             ^ Z (blue)                     |
Z=1200 |                             +--> X (red)   <- center (0,1200)
       |                                                            |
       |                                                            |
Z=100  |-[B]---[c][c][c]-------------[B]-------------[c][c][c]---[B]-|  <- ground line
Z=0    +------------------------------------------------------------+
     X=-1600                        X=0                          X=1600
```
`[B]` = missile base, `[c]` = city. Ground occupies Z 0 to 100. Bases and cities stand on top of it at Z = 100.

| Object | X positions | Approx. horizontal position as % of playfield width |
|---|---|---|
| Base Left | -1400 | 6.25 % |
| Base Center | 0 | 50 % |
| Base Right | 1400 | 93.75 % |
| Cities (6) | -1050, -750, -450, 450, 750, 1050 | 17.2, 26.6, 35.9, 64.1, 73.4, 82.8 % |

Playfield size: width 3200, height 2400 (4:3).

### 4.3 Folder structure

```
/Game/DomeDefense/
  Maps/         L_Game
  Core/         BP_DDGameMode, BP_DDPlayerController, BP_DDHUD, BFL_DDDebug
  Data/         PDA_DDConfig, DA_DDConfig, E_DDBaseId
  Actors/       BP_PlayfieldCamera, BP_PlayfieldDebug
  Materials/    M_DD_Unlit, MI_DD_Ground
```
Later phases add to `Actors/` (and create sub-folders if needed). Do not create any other folders now.

### 4.4 Naming rules

| Kind | Prefix | Example |
|---|---|---|
| Blueprint class | `BP_` | `BP_PlayfieldCamera` |
| Core framework Blueprint | `BP_DD` | `BP_DDGameMode` |
| Blueprint function library | `BFL_` | `BFL_DDDebug` |
| Data asset class / instance | `PDA_` / `DA_` | `PDA_DDConfig` / `DA_DDConfig` |
| Enum | `E_` | `E_DDBaseId` |
| Material / instance | `M_` / `MI_` | `M_DD_Unlit` |
| Map | `L_` | `L_Game` |

Variables and functions use `PascalCase`. Booleans start with `b` (e.g. `bShowOverlay`). Level actor labels have no prefix (e.g. `PlayfieldCamera`).

### 4.5 Blueprint style rules

- No `Event Tick` unless this document says so. In Phase 0 only `BP_PlayfieldDebug` ticks.
- Cache references in variables on `BeginPlay`. Do not call `Get Actor Of Class` or `Cast` every frame.
- Put a **comment box** around each logical block of nodes in a graph, named after what it does.
- Keep logic in the Blueprint that owns the data. Prefer calling a function on a known reference over long event chains.

### 4.6 Config access convention

All tunable numbers live in `DA_DDConfig` (an instance of `PDA_DDConfig`).
`BP_DDGameMode` holds the reference and exposes `GetConfig`. Any other Blueprint gets config like this:

```
Get Game Mode → Cast To BP_DDGameMode → GetConfig → (PDA_DDConfig reference)
```
Do this once in `BeginPlay`, store the result in a variable, and re-use it.

### 4.7 Logging convention

All debug text goes through `BFL_DDDebug.DDLog(Message, Category, bIsError)`. Do not call `Print String` directly anywhere else. Output format:

```
[DD][<Category>] <Message>
[DD][<Category>] ERROR: <Message>      (when bIsError is true)
```

### 4.8 Other decisions locked in Phase 0

| Topic | Decision |
|---|---|
| GameMode parent | `GameModeBase` (not `GameMode`) |
| Pawn | **None.** `Default Pawn Class = None`. The player controller works without a pawn. |
| Input | Phase 0 uses **direct key event nodes** in `BP_DDPlayerController` for debug keys only. Game input is decided in Phase 1. |
| UI | Drawn by `BP_DDHUD` canvas functions (`Draw Text`, `Draw Rect`, ...). **No UMG** unless a later phase says so. |
| Collision | **No custom channels.** Overlaps use the built-in `OverlapAllDynamic` preset and filter by `Cast`. |
| Visuals | Primitive meshes with an **unlit** material. No lights. |
| Debug keys | `H` = toggle debug overlay (Phase 0). Reserved for later: `E` spawn enemy (Phase 4), `N` next wave (Phase 6), `K` kill all enemies (Phase 4). |

---

## 5. Implementation steps

### Step 1 — Create folders
Create the folders listed in section 4.3 (`Maps`, `Core`, `Data`, `Actors`, `Materials`).

**Check:** all five folders exist under `/Game/DomeDefense/`.

---

### Step 2 — Enum `E_DDBaseId`
Create a Blueprint Enumeration at `/Game/DomeDefense/Data/E_DDBaseId` with exactly these entries, in this order:

| Index | Name | Display name |
|---|---|---|
| 0 | `Left` | Left |
| 1 | `Center` | Center |
| 2 | `Right` | Right |

It is not used in Phase 0. Phase 2 uses it.

**Check:** enum exists with 3 entries in the order above.

---

### Step 3 — Config class and instance

**3a.** Create a Blueprint class `PDA_DDConfig` in `/Game/DomeDefense/Data/` with parent class **`PrimaryDataAsset`**.

**3b.** Add these variables. All are **Instance Editable** (visible in details panel). Set the listed **default values** in the class defaults.

| Category | Variable | Type | Default |
|---|---|---|---|
| Playfield | `PlayPlaneY` | float | `0.0` |
| Playfield | `PlayfieldMinX` | float | `-1600.0` |
| Playfield | `PlayfieldMaxX` | float | `1600.0` |
| Playfield | `PlayfieldMinZ` | float | `0.0` |
| Playfield | `PlayfieldMaxZ` | float | `2400.0` |
| Playfield | `GroundTopZ` | float | `100.0` |
| Playfield | `CameraDistanceY` | float | `3000.0` |
| Layout | `BaseLeftX` | float | `-1400.0` |
| Layout | `BaseCenterX` | float | `0.0` |
| Layout | `BaseRightX` | float | `1400.0` |
| Layout | `CityX` | array of float | `[-1050.0, -750.0, -450.0, 450.0, 750.0, 1050.0]` |
| ABM | `AbmStartAmmo` | integer | `10` |
| ABM | `AbmSpeedSide` | float | `1200.0` |
| ABM | `AbmSpeedCenter` | float | `3000.0` |
| Explosion | `ExplosionMaxRadius` | float | `160.0` |
| Explosion | `ExplosionGrowTime` | float | `0.4` |
| Explosion | `ExplosionHoldTime` | float | `0.4` |
| Explosion | `ExplosionShrinkTime` | float | `0.5` |
| Enemy | `EnemyBaseSpeed` | float | `130.0` |
| Scoring | `ScoreEnemyMissile` | integer | `25` |
| Scoring | `ScoreBomber` | integer | `100` |
| Scoring | `ScoreSmartBomb` | integer | `125` |
| Scoring | `ScoreUnusedAbm` | integer | `5` |
| Scoring | `ScoreSurvivingCity` | integer | `100` |
| Scoring | `MaxWaveMultiplier` | integer | `6` |

The ABM, Explosion, Enemy, and Scoring values are initial guesses. Later phases may tune them. Phase 0 only creates them.

**3c.** Add these **pure functions** to `PDA_DDConfig` (mark each as **Pure**):

| Function | Returns | Logic |
|---|---|---|
| `GetPlayfieldWidth` | float | `PlayfieldMaxX - PlayfieldMinX` |
| `GetPlayfieldHeight` | float | `PlayfieldMaxZ - PlayfieldMinZ` |
| `GetPlayfieldCenter` | vector | `X = (PlayfieldMinX + PlayfieldMaxX) / 2`, `Y = PlayPlaneY`, `Z = (PlayfieldMinZ + PlayfieldMaxZ) / 2` |

**3d.** Create a data asset instance of `PDA_DDConfig` named `DA_DDConfig` in `/Game/DomeDefense/Data/` (Content Browser → Add → Miscellaneous → Data Asset → pick `PDA_DDConfig`). Leave all values at their defaults.

**Check:** `DA_DDConfig` exists. `GetPlayfieldWidth` = 3200, `GetPlayfieldHeight` = 2400, `GetPlayfieldCenter` = (0, 0, 1200). `CityX` has 6 entries.

---

### Step 4 — Logging library `BFL_DDDebug`
Create a Blueprint Function Library `BFL_DDDebug` in `/Game/DomeDefense/Core/`.

Add one function `DDLog`:

| Input | Type | Default |
|---|---|---|
| `Message` | string | (empty) |
| `Category` | string | `Core` |
| `bIsError` | boolean | `false` |

Logic:

```
Prefix = bIsError ? "ERROR: " : ""
Full   = "[DD][" + Category + "] " + Prefix + Message
Print String(
    In String       = Full,
    Print To Screen = true,
    Print To Log    = true,
    Text Color      = bIsError ? (R=1, G=0, B=0, A=1) : (R=0, G=1, B=1, A=1),
    Duration        = 5.0)
```

**Check:** compiles with no warnings. Function is visible in the Blueprint action menu as `DDLog`.

---

### Step 5 — Materials

**5a.** Create a Material `M_DD_Unlit` in `/Game/DomeDefense/Materials/`:
- Shading Model: **Unlit**. Blend Mode: **Opaque**. Two Sided: **true**.
- One **Vector Parameter** node named `Color`, default `(1, 1, 1, 1)`, connected to **Emissive Color**.

**5b.** Create a Material Instance `MI_DD_Ground` (parent `M_DD_Unlit`) in the same folder. Override `Color` to `(R=0.55, G=0.30, B=0.05, A=1)`.

**Fallback (only if material graph editing is not possible):** skip 5a/5b, use `/Engine/BasicShapes/BasicShapeMaterial` on the ground mesh, add one `DirectionalLight` to the map named `TempLight`, and record it under *Deviations*.

**Reference palette for later phases (do not create now):**

| Instance | Color (R, G, B) |
|---|---|
| `MI_DD_City` | (0.1, 0.8, 0.2) |
| `MI_DD_Base` | (0.0, 0.8, 0.9) |
| `MI_DD_Abm` | (0.3, 0.7, 1.0) |
| `MI_DD_Explosion` | (1.0, 0.9, 0.3) |
| `MI_DD_Enemy` | (1.0, 0.2, 0.2) |
| `MI_DD_Crosshair` | (1.0, 1.0, 1.0) |

**Check:** `MI_DD_Ground` exists and its parent is `M_DD_Unlit`.

---

### Step 6 — `BP_DDGameMode`
Create a Blueprint class `BP_DDGameMode` in `/Game/DomeDefense/Core/` with parent **`GameModeBase`**.

**Class defaults:**
- `Default Pawn Class` = **None**
- `Player Controller Class` = `BP_DDPlayerController` (create the class first in Step 9 if the tool requires; otherwise set this after Step 9)
- `HUD Class` = `BP_DDHUD` (same note, set after Step 10)

**Variable:** `Config` — type: object reference to `PDA_DDConfig`, Instance Editable, **default value = `DA_DDConfig`**.

**Function:** `GetConfig` — pure, no inputs, returns `PDA_DDConfig` reference. Returns `Config`.

**Event BeginPlay:**

```
DDLog("GameMode BeginPlay OK", "Core", false)
Branch(IsValid(Config))
  False → DDLog("Config is not assigned", "Core", true) ; stop
  True  → DDLog("Config OK: Width=" + Config.GetPlayfieldWidth()
                + " Height=" + Config.GetPlayfieldHeight()
                + " GroundTopZ=" + Config.GroundTopZ
                + " Cities=" + Length(Config.CityX), "Core", false)
```

**Check:** compiles clean. Class defaults show `Config = DA_DDConfig`.

---

### Step 7 — `BP_PlayfieldCamera`
Create a Blueprint class `BP_PlayfieldCamera` in `/Game/DomeDefense/Actors/` with parent **`CameraActor`**.

**Class defaults (CameraComponent):**
- Projection Mode: **Orthographic**
- Ortho Width: `3200.0`
- Constrain Aspect Ratio: **true**; Aspect Ratio: `1.333333`
- Ortho Near Clip Plane: `1.0`; Ortho Far Clip Plane: `10000.0` (if this engine version exposes different near/far properties for orthographic, set the equivalents so the plane at Y = 0, 3000 units from the camera, is inside the range)
- Post Process Settings (enable the override checkbox for each): **Min EV100 = 1.0**, **Max EV100 = 1.0** (if the engine shows *Min Brightness / Max Brightness* instead, set both to `1.0`), **Bloom Intensity = 0**, **Vignette Intensity = 0**
- Actor: **Auto Activate For Player = Player 0**; **Start With Tick Enabled = false**

**Event BeginPlay:**

```
Cfg = Get Game Mode → Cast To BP_DDGameMode → GetConfig
If cast failed or Cfg not valid → DDLog("No config", "Camera", true) ; stop
Center = Cfg.GetPlayfieldCenter()
Set Actor Location And Rotation(
    Location = (Center.X, Cfg.PlayPlaneY + Cfg.CameraDistanceY, Center.Z),
    Rotation = Make Rotator(Roll=0, Pitch=0, Yaw=-90))
CameraComponent.Set Ortho Width(Cfg.GetPlayfieldWidth())
CameraComponent.Set Aspect Ratio(Cfg.GetPlayfieldWidth() / Cfg.GetPlayfieldHeight())
DDLog("Configured Loc=" + <X,Y,Z of actor location> + " OrthoWidth=" + <width> + " Aspect=" + <aspect>, "Camera", false)
```

**Camera exposure calibration:** in the PIE test (section 6) the colors must look like the colors in the palette (e.g. the green city markers are clearly saturated green, not white or dark). If they look blown out or dim, change Min EV100 and Max EV100 **together** by steps of 1 until correct, and record the final value in the report.

**Check:** compiles clean.

---

### Step 8 — `BP_PlayfieldDebug`
Create a Blueprint class `BP_PlayfieldDebug` in `/Game/DomeDefense/Actors/` with parent **`Actor`**. Root: a `Scene` component. **Start With Tick Enabled = true.** Does not need to be visible in game.

**Variables:**
- `Cfg` — object reference `PDA_DDConfig` (not editable)
- `bShowOverlay` — boolean, default `true`

**Function `ToggleOverlay`** (no inputs/outputs): `bShowOverlay = NOT bShowOverlay`, then `DDLog("Overlay=" + (bShowOverlay ? "ON" : "OFF"), "Debug", false)`.

**Event BeginPlay:**

```
Cfg = Get Game Mode → Cast To BP_DDGameMode → GetConfig
If invalid → DDLog("No config", "Debug", true) ; stop
DDLog("PlayfieldDebug ready (overlay=" + (bShowOverlay ? "ON" : "OFF") + ")", "Debug", false)
```

**Event Tick:** if `bShowOverlay` is false or `Cfg` is invalid, do nothing. Otherwise draw all of the following with **Duration = 0.0** (one frame). Every point has `Y = Cfg.PlayPlaneY`. Let `MinX, MaxX, MinZ, MaxZ, G = GroundTopZ`, and `C = Cfg.GetPlayfieldCenter()`.

| # | Element | Node | Geometry | Color (R,G,B) | Thickness |
|---|---|---|---|---|---|
| 1 | Playfield bounds | 4× Draw Debug Line | (MinX,MinZ)→(MaxX,MinZ), (MaxX,MinZ)→(MaxX,MaxZ), (MaxX,MaxZ)→(MinX,MaxZ), (MinX,MaxZ)→(MinX,MinZ) | Yellow (1,1,0) | 6 |
| 2 | Ground line | Draw Debug Line | (MinX, G)→(MaxX, G) | White (1,1,1) | 3 |
| 3 | X axis gizmo | Draw Debug Line | C → C + (400, 0, 0) | Red (1,0,0) | 6 |
| 4 | Z axis gizmo | Draw Debug Line | C → C + (0, 0, 400) | Blue (0,0.4,1) | 6 |
| 5 | 3 base markers | 3× Draw Debug Box | Center = (BaseX, Y, G + 50), Extent = (50, 5, 50), for BaseX in {BaseLeftX, BaseCenterX, BaseRightX} | Cyan (0,1,1) | 4 |
| 6 | 6 city markers | ForEach over `Cfg.CityX`, Draw Debug Box | Center = (CityX[i], Y, G + 40), Extent = (60, 5, 40) | Green (0,1,0) | 4 |

Draw Debug Box rotation = (0,0,0).

**Check:** compiles clean.

---

### Step 9 — `BP_DDPlayerController`
Create a Blueprint class `BP_DDPlayerController` in `/Game/DomeDefense/Core/` with parent **`PlayerController`**.

**Class defaults:** **Auto Manage Active Camera Target = false.**

**Variable:** `PlayfieldDebugActor` — object reference to `BP_PlayfieldDebug` (not editable).

**Event BeginPlay:**

```
DDLog("PlayerController BeginPlay OK", "Core", false)

Cam = Get Actor Of Class(BP_PlayfieldCamera)
Branch(IsValid(Cam))
  True  → Set View Target With Blend(New View Target = Cam, Blend Time = 0.0)
          DDLog("View target set to PlayfieldCamera", "Core", false)
  False → DDLog("No BP_PlayfieldCamera in level", "Core", true)

PlayfieldDebugActor = Get Actor Of Class(BP_PlayfieldDebug)
Branch(IsValid(PlayfieldDebugActor)) False → DDLog("No BP_PlayfieldDebug in level", "Core", true)
```

**Key event `H` (Pressed):** if `PlayfieldDebugActor` is valid → call `ToggleOverlay` on it.

Use the direct **keyboard key event node** for `H`. Do not create input mapping or Enhanced Input assets.

**Check:** compiles clean. Then go back to Step 6 and make sure `Player Controller Class = BP_DDPlayerController`.

---

### Step 10 — `BP_DDHUD`
Create a Blueprint class `BP_DDHUD` in `/Game/DomeDefense/Core/` with parent **`HUD`**.

- **Event BeginPlay:** `DDLog("HUD BeginPlay OK", "Core", false)`
- **Event ReceiveDrawHUD:** `Draw Text(Text = "DOME DEFENSE | PHASE 0 | FOUNDATIONS", Text Color = White, Screen X = 20, Screen Y = 20, Font = None/default, Scale = 1.5, Scale Position = false)`

**Check:** compiles clean. Then go back to Step 6 and make sure `HUD Class = BP_DDHUD`.

---

### Step 11 — Level `L_Game`

**11a.** Create a new level from the **Empty Level** template (no sky, no lights, no floor). Save as `/Game/DomeDefense/Maps/L_Game`.

**11b.** **World Settings → GameMode Override = `BP_DDGameMode`.**

**11c.** Place exactly these actors (and nothing else). Set the **Actor Label** exactly as shown.

| Label | Class / asset | Location (X, Y, Z) | Rotation (Roll, Pitch, Yaw) | Scale | Notes |
|---|---|---|---|---|---|
| `PlayfieldCamera` | `BP_PlayfieldCamera` | (0, 3000, 1200) | (0, 0, **-90**) | (1,1,1) | |
| `PlayfieldDebug` | `BP_PlayfieldDebug` | (0, 0, 0) | (0, 0, 0) | (1,1,1) | |
| `Ground` | Static Mesh Actor, mesh `/Engine/BasicShapes/Cube`, material slot 0 = `MI_DD_Ground` | (0, 0, 50) | (0, 0, 0) | **(32, 1, 1)** | Mobility: Static. Collision preset: **NoCollision**. Result: spans X ±1600, Z 0 to 100. |
| `PlayerStart` | `PlayerStart` | (0, 0, 500) | (0, 0, 0) | (1,1,1) | Only prevents a "no player start" warning. No pawn is spawned. |

**11d.** **Project Settings → Maps & Modes:** Editor Startup Map = `L_Game`, Game Default Map = `L_Game`, Default GameMode = `BP_DDGameMode`. (If the tooling cannot change project settings, record it under *Deviations*; the World Settings override from 11b is sufficient to run.)

**Check:** the level contains exactly the 4 actors above. Opening the project loads `L_Game`.

---

### Step 12 — Optional rendering settings
Only if the tooling can edit `Config/DefaultEngine.ini`. These save GPU time and avoid Lumen artifacts with an orthographic camera. **Do not restart the editor yourself.** Tell the human a restart is needed.

```ini
[/Script/Engine.RendererSettings]
r.DynamicGlobalIlluminationMethod=0
r.ReflectionMethod=0
```
If any of these cannot be set, skip it and mention it in the report. Nothing else in this document depends on them.

---

### Step 13 — Save
Run **Save All**. Confirm there are no unsaved assets or maps.

---

## 6. Acceptance tests

Run PIE using **Selected Viewport** (standalone is also fine) for at least 5 seconds. Press `H` once, wait 2 seconds, press `H` again, then stop PIE.

### A. Static checks
- [ ] **A1** All assets in section 4.3 exist at the exact paths and names.
- [ ] **A2** Every Blueprint (`PDA_DDConfig`, `BFL_DDDebug`, `BP_DDGameMode`, `BP_PlayfieldCamera`, `BP_PlayfieldDebug`, `BP_DDPlayerController`, `BP_DDHUD`) compiles with 0 errors and 0 warnings.
- [ ] **A3** `DA_DDConfig` values match the table in Step 3b.
- [ ] **A4** `BP_DDGameMode` has `Default Pawn Class = None`, `Player Controller Class = BP_DDPlayerController`, `HUD Class = BP_DDHUD`, `Config = DA_DDConfig`.
- [ ] **A5** `L_Game` contains exactly the 4 actors in Step 11c with the exact transforms. World Settings GameMode Override = `BP_DDGameMode`.

### B. Log checks (Output Log and/or on-screen text)
Line order is **not** guaranteed. Number formatting may differ (e.g. `3200.000000` instead of `3200`). Match on the names and the numeric values.

- [ ] **B1** `[DD][Core] GameMode BeginPlay OK`
- [ ] **B2** `[DD][Core] Config OK: Width=3200 Height=2400 GroundTopZ=100 Cities=6`
- [ ] **B3** `[DD][Camera] Configured Loc=(0,3000,1200) OrthoWidth=3200 Aspect=1.3333`
- [ ] **B4** `[DD][Core] PlayerController BeginPlay OK`
- [ ] **B5** `[DD][Core] View target set to PlayfieldCamera`
- [ ] **B6** `[DD][Core] HUD BeginPlay OK`
- [ ] **B7** `[DD][Debug] PlayfieldDebug ready (overlay=ON)`
- [ ] **B8** After pressing `H`: `[DD][Debug] Overlay=OFF`; after pressing `H` again: `[DD][Debug] Overlay=ON`
- [ ] **B9** **No** log line containing `ERROR`, `Accessed None`, `Blueprint Runtime Error`, or `Blueprint Compile` problems.

### C. Visual checks (screenshot of the PIE viewport)
If the agent cannot capture a screenshot, mark C as **"needs human confirmation"** in the report. Do not mark it as passed.

- [ ] **C1** Black background. Nothing visible except the items below.
- [ ] **C2** The **yellow rectangle** is visible on all four sides and its outer edge coincides with the edge of the rendered 4:3 area (no visible gap).
- [ ] **C3** A **brown ground strip** fills the bottom ~4.2 % of the playfield height (100 of 2400), with a thin **white line** along its top edge, spanning the full width.
- [ ] **C4** At the exact center of the playfield: a **red line pointing RIGHT** and a **blue line pointing UP**.
- [ ] **C5** **3 cyan boxes** sitting on the ground line at roughly 6 %, 50 %, and 94 % of the width.
- [ ] **C6** **6 green boxes** sitting on the ground line at roughly 17, 27, 36, 64, 73, and 83 % of the width (3 left of center, 3 right, mirror-symmetric).
- [ ] **C7** White HUD text "DOME DEFENSE | PHASE 0 | FOUNDATIONS" in the top-left corner of the window.
- [ ] **C8** Pressing `H` hides **all** of C2, C3's white line, C4, C5, C6. The brown ground and the HUD text stay. Pressing `H` again brings them back.
- [ ] **C9** Colors look right (green is clearly green, cyan is clearly cyan, nothing washed out). If you changed EV100 in Step 7, note the final value.
- [ ] **C10** Resizing the PIE window letterboxes or pillarboxes the playfield. It never stretches.

### D. Hygiene
- [ ] **D1** Save All done. No unsaved assets.
- [ ] **D2** PIE is stopped.
- [ ] **D3** No assets exist under `/Game/DomeDefense/` other than those listed in section 4.3, plus `Content/` default engine folders the editor created itself.

---

## 7. Report format (the agent's final message)

```
## Phase 0 report
**Engine version:** x.y.z
**MCP tools used:** (list)

### Assets created
(table: path | type | compiled clean yes/no)

### Acceptance results
A1 PASS/FAIL ... through D3 PASS/FAIL/NEEDS-HUMAN

### Log excerpt
(paste the [DD] lines from PIE)

### Calibration values
Min EV100 / Max EV100 final: ...

### Deviations
(every place you did not follow this document, why, and the fallback used)

### Open questions / risks
(anything ambiguous or surprising)
```

---

## 8. What Phase 0 hands to Phase 1

| Provided | How Phase 1 uses it |
|---|---|
| World coordinate system (screen right = +X, up = +Z, play plane Y = 0) | Mouse position is deprojected from the orthographic camera and intersected with the plane `Y = PlayPlaneY`. |
| `BP_DDPlayerController` | Phase 1 adds the cursor, crosshair spawning, and input here. |
| `DA_DDConfig` + `GetConfig` convention | Phase 1 clamps the crosshair to `PlayfieldMin/Max X/Z`. |
| `DDLog` | All Phase 1 debug output. |
| `BP_PlayfieldDebug` overlay | Visual proof that the crosshair lands on the right world position. |
| `H` debug key and reserved key list | Later phases add keys without conflicts. |

---

## 9. Common pitfalls (check these if a test fails)

1. **Black screen or wrong view:** the view target is not the camera. Check `Auto Manage Active Camera Target = false` on `BP_DDPlayerController`, the `Set View Target With Blend` call, and `Auto Activate For Player = Player 0` on the camera.
2. **Scene is mirrored (red line points LEFT):** camera yaw is +90 instead of **-90**, or the camera is placed at Y = -3000. It must be at **+3000** with Yaw **-90**.
3. **Nothing rendered but the background:** orthographic far clip plane is too small (the play plane is 3000 units from the camera).
4. **Yellow rectangle does not touch the viewport edges:** `Constrain Aspect Ratio` is off, the aspect ratio is not 4:3, or the camera Ortho Width was not set from the config.
5. **"Config is not assigned" error:** `BP_DDGameMode.Config` default was not set to `DA_DDConfig`, or the level's GameMode Override is not `BP_DDGameMode`.
6. **No HUD text:** `HUD Class` on `BP_DDGameMode` is not `BP_DDHUD`.
7. **Colors washed out or too dark:** exposure. See "Camera exposure calibration" in Step 7.
8. **Debug lines flicker or vanish:** `Duration` must be `0.0` and the draw calls must be inside `Event Tick`.
9. **`Get Actor Of Class` returns None:** the actor is missing from the level or has a different class. Check the Step 11c table.
