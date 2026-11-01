# Phase 1 — Crosshair and Input

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Depends on** | Phase 0 accepted (see section 3) |
| **Phase goal** | The mouse controls a crosshair on the play plane. A left click and the `1`/`2`/`3` keys raise "fire requests" that are logged and shown on the HUD. No missiles yet. |

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Out-of-scope items are in section 2.
2. **Names are exact and case-sensitive.** Asset, variable, function, and folder names must match this document character for character.
3. **Phase 0 logic must survive.** Before editing any existing Blueprint, read its current graphs. Extend them; do not rewrite or delete Phase 0 nodes unless a step says so.
4. **Compile after every Blueprint change.** Every Blueprint must end with status "compiled, no errors, no warnings".
5. **Run the "Check" line at the end of each step** before moving to the next.
6. **You cannot press keys or move the mouse in PIE.** Do not try. Phase 1 therefore includes a built-in **self-test** (Step 6) that exercises the screen-to-world math, the clamp, and the fire path using synthetic inputs, and logs PASS/FAIL. Interactive checks (section 6, group D) are marked **NEEDS-HUMAN**.
7. **If something cannot be done** (a tool does not support an operation, a property has a different name, an asset path does not exist):
   - If this document names a **Fallback**, use it and record it under *Deviations*.
   - Otherwise stop that item, do not invent an alternative, continue with steps that do not depend on it, and record it under *Deviations*.
8. **Only touch** `/Game/DomeDefense/` and the project config entries named in this document.
9. **Do not commit to source control.** The human commits after reviewing.
10. **Do not leave the editor in Play-In-Editor (PIE).** Stop PIE after each test.
11. **Read section 5 (Tooling notes) before you build any graph.** It records what failed in Phase 0.
12. Values given as `(X, Y, Z)` are Unreal world units (cm) in the world coordinate system of section 4.1.

---

## 2. Scope

### In scope
- Phase 0 patch (two small fixes).
- One new config value: `CrosshairMinZ`.
- `BP_Crosshair`: a white plus-shaped crosshair built from two cubes.
- `MI_DD_Crosshair`: its material instance.
- `BP_DDPlayerController` additions: hidden OS cursor, mouse-to-play-plane conversion, clamping, crosshair movement, fire requests (left click and keys `1`, `2`, `3`), and the self-test.
- `BP_DDHUD` additions: live crosshair coordinates and last fire request.

### Out of scope (do NOT create)
- Bases, ABMs, explosions, enemy missiles, cities as gameplay actors.
- Ammo, scoring, waves, game-over.
- Enhanced Input assets (Input Actions, Mapping Contexts). Use direct key event nodes only.
- UMG widgets, sounds, particle effects, textures.
- Any change to the level's placed actors (the crosshair is **spawned at runtime**).
- Any other keys than `H` (Phase 0), left mouse button, `1`, `2`, `3`.

---

## 3. Prerequisites

- Phase 0 is accepted: `L_Game` opens, and Phase 0 PIE logs show the seven `[DD]` startup lines with no `ERROR`.
- The editor is open on `DomeDefense` and the MCP connection works.
- The human has turned off **Editor Preferences → General → Performance → "Use Less CPU when in Background"**. (In Phase 0 the first PIE ran throttled at max tick rate 3.) If you see "max tick rate 3" in the PIE log, mention it in *Open questions*.

---

## 4. Conventions (Phase 0 as built, plus Phase 1 additions)

### 4.1 Coordinate system and units (unchanged)

| Item | Decision |
|---|---|
| Units | 1 Unreal unit = 1 cm. Speeds in units/second, durations in seconds. |
| Play plane | XZ plane at `Y = PlayPlaneY` (0). |
| Screen right / up | **+X** / **+Z** |
| Depth | Y. Gameplay actors are placed at Y = 0 and never move in Y. |
| Camera | Orthographic at (0, 3000, 1200), looking toward −Y (Yaw −90). Playfield 3200 × 2400 (4:3), letterboxed. |
| Time | Time-based movement only. |

### 4.2 Conventions as built in Phase 0 (these replace the Phase 0 text where they differ)

| Topic | As built |
|---|---|
| Base identity | **No enum.** `E_DDBaseId` was dropped (the toolset cannot author enums). Bases are an **integer index: 0 = Left, 1 = Center, 2 = Right**. `-1` means "automatic / nearest base" (Phase 2 decides). |
| Config access | `PDA_DDConfig` functions and `BP_DDGameMode.GetConfig` are **not Pure** (the toolset cannot set the Pure flag). Every call needs an **execution wire**. Pattern: `Get Game Mode → Cast To BP_DDGameMode → GetConfig`, once in `BeginPlay`, stored in a variable. |
| Comment boxes | Not required (the toolset cannot create them). Ignore the Phase 0 comment-box rule. |
| Level | `L_Game` was derived from `/Game/Maps/Map` with template actors removed. The level contains engine actors (World Settings and so on) in addition to the four Phase 0 actors. |
| Camera exposure | The engine exposes `AutoExposureMinBrightness` / `AutoExposureMaxBrightness` (both 1). Bloom and vignette are 0. |
| Log number formats | Floats print like `3200.0`; vectors print like `X=0.000 Y=3000.000 Z=1200.000`. **All Phase 1 coordinate logs use rounded integers** to avoid format ambiguity. |

### 4.3 Logging (unchanged)
All debug text goes through `BFL_DDDebug.DDLog(Message, Category, bIsError)`. Output: `[DD][<Category>] <Message>` or `[DD][<Category>] ERROR: <Message>`. Phase 1 adds categories `Input` and `Test`.

`DDLog` is a **static** Blueprint library function. If a compile error says "Target must have a connection" or "could not find a function named DDLog", the node was created as an instance call. Delete it and recreate it using `find_node_types` (see section 5).

### 4.4 Input scheme (new, locked here)

| Input | Action |
|---|---|
| Mouse move | Crosshair follows the mouse, clamped to the playfield. |
| Left Mouse Button (pressed) | `RequestFire(-1)` (automatic base) |
| `1` (pressed) | `RequestFire(0)` (Left base) |
| `2` (pressed) | `RequestFire(1)` (Center base) |
| `3` (pressed) | `RequestFire(2)` (Right base) |
| `H` (pressed) | Toggle debug overlay (Phase 0). |

Reserved for later (do not use): `E` spawn enemy (Phase 4), `K` kill all enemies (Phase 4), `N` next wave (Phase 6).

### 4.5 Visual layering (new; applies to later phases)
Camera is at +Y, so a **larger Y is closer to the camera**. The actor stays at Y = 0; the offset is applied to its **visual components** (relative location Y).

| Layer | Visual component Y offset |
|---|---|
| Ground | 0 (cube front face is at +50) |
| Debug overlay | 0 (drawn by debug lines) |
| Cities, bases | +60 |
| Enemy missiles and trails | +70 |
| ABMs and trails | +80 |
| Explosions | +90 |
| Crosshair | **+100** |

### 4.6 Self-test convention (new; applies to later phases)
Because the agent cannot inject input, each phase adds a self-test that logs `[DD][Test] <name>: PASS|FAIL ...` and a final `[DD][Test] SelfTest done: n/m PASS`. A failing line is logged with `ERROR` (via `bIsError`).

---

## 5. Tooling notes (learned from the Phase 0 log)

Read these before you build graphs.

1. **Do not guess node IDs.** In Phase 0 the DSL form `Variables|Default|Get<VarName>` failed (`The node could not be created`) for both config variables and `bShowOverlay`. Call `BlueprintTools.get_graph_dsl_docs` once, and use `find_node_types` to discover exact node IDs before writing a graph.
2. **Key and mouse-button events cannot be written as DSL.** Top-level DSL forms must start with `event` or `fn`. Create key-event nodes with `BlueprintTools.create_node` (find the node type with `find_node_types`, for example query "Left Mouse Button", "One", "H"), then wire them with `connect_pins`. This is how the `H` key was done in Phase 0.
3. **Sequence node pin names** are `then_0`, `then_1`, not "Then 0".
4. **"Unreachable code after branch/return"** warnings: do not place nodes after a `Branch` or `return` in the same exec chain. Put the remaining logic inside the branch body that needs it.
5. **`set_node_pos` fails** on comment nodes. You do not need comment nodes.
6. **Properties that cannot be read or set through `ObjectTools`:** `bIsPureFunc`, `BlueprintType`, `ActorLabel`. Use the Actor toolset for actor labels. Do not try to set the Pure flag.
7. **Blueprint Function Libraries cannot be created** by `BlueprintTools.create`. You do not need one in Phase 1.
8. **Save before loading a level.** `SceneTools.load_level` fails if the current level has unsaved changes.
9. **PIE tools that exist:** `StartPIE`, `StopPIE`, `IsPIERunning`, `GetLogEntries`, `CaptureViewport`, `CaptureEditorImage`. Use them in this order: StartPIE → wait about 3 seconds → GetLogEntries (filter `[DD]`) → CaptureViewport → StopPIE.
10. **Compile each Blueprint** with `compile_blueprint` after writing graphs and read the result. Do not rely on the write call alone.

---

## 6. Implementation steps

### Step 1 — Phase 0 patch

**1a. `BP_PlayfieldDebug.ToggleOverlay` logs the old value.** Fix: set `bShowOverlay = NOT bShowOverlay` **first**, then call `DDLog("Overlay=" + (bShowOverlay ? "ON" : "OFF"), "Debug", false)`. Expected: the first `H` press logs `Overlay=OFF`.

**1b. `Ground` collision.** On the level actor `Ground`, set the collision so that **Collision Enabled = NoCollision** (preset `NoCollision`). In Phase 0 the preset name was set but the enabled flag stayed `QueryAndPhysics`. Read the property back to verify.

**Check:** both properties read back correctly; `BP_PlayfieldDebug` compiles clean.

---

### Step 2 — Config: `CrosshairMinZ`

Add to `PDA_DDConfig` (Instance Editable, category `Crosshair`):

| Variable | Type | Default |
|---|---|---|
| `CrosshairMinZ` | float | `100.0` |

Compile. Open `DA_DDConfig` and read back `CrosshairMinZ`. If it is not `100.0`, set it to `100.0` on the instance.

**Check:** `DA_DDConfig.CrosshairMinZ = 100.0`; `PDA_DDConfig` compiles clean; the existing 25 variables and their values are unchanged.

---

### Step 3 — Material instance `MI_DD_Crosshair`

Create a Material Instance `MI_DD_Crosshair` (parent `M_DD_Unlit`) in `/Game/DomeDefense/Materials/`. Override `Color` to `(R=1, G=1, B=1, A=1)`.

**Check:** exists; parent is `M_DD_Unlit`; `Color` reads back as white.

---

### Step 4 — `BP_Crosshair`

Create a Blueprint class `BP_Crosshair` in `/Game/DomeDefense/Actors/` with parent **`Actor`**.

**Components** (root is a `Scene` component named `Root`):

| Component name | Type | Mesh | Relative location | Relative scale | Material | Collision |
|---|---|---|---|---|---|---|
| `BarH` | StaticMeshComponent | `/Engine/BasicShapes/Cube` | (0, **100**, 0) | (1.2, 0.06, 0.06) | `MI_DD_Crosshair` | NoCollision |
| `BarV` | StaticMeshComponent | `/Engine/BasicShapes/Cube` | (0, **100**, 0) | (0.06, 0.06, 1.2) | `MI_DD_Crosshair` | NoCollision |

Result: a white plus sign, 120 units long and 6 units thick, 100 units in front of the play plane (layering table, section 4.5).

**Class defaults:** **Start With Tick Enabled = false.** No variables, no graph logic.

**Check:** compiles clean; both bars have NoCollision (read back **Collision Enabled**, not only the preset name).

---

### Step 5 — `BP_DDPlayerController` additions

Read the existing graphs first. Keep the Phase 0 `BeginPlay` logic and the `H` key handling.

**5a. Class defaults**
- **Start With Tick Enabled = true.**
- **Show Mouse Cursor = true.**
- **Default Mouse Cursor = None** and **Current Mouse Cursor = None** (hides the OS cursor but keeps mouse tracking).
- Keep **Auto Manage Active Camera Target = false** (Phase 0).
- **Fallback:** if the cursor properties cannot be set or the OS cursor is still visible in PIE, leave the OS cursor visible and record it under *Deviations*.

**5b. New variables** (not Instance Editable unless stated)

| Variable | Type | Default |
|---|---|---|
| `Cfg` | object reference `PDA_DDConfig` | none |
| `Crosshair` | object reference `BP_Crosshair` | none |
| `CrosshairWorldPos` | vector | (0, 0, 0) |
| `LastFirePos` | vector | (0, 0, 0) |
| `FireCount` | integer | 0 |
| `TestPassCount` | integer | 0 |
| `bSelfTest` | boolean, **Instance Editable** | **true** |

(`PlayfieldDebugActor` from Phase 0 stays.)

**5c. New functions** (all non-Pure; call them on execution wires)

**`ScreenToPlane`**
- Inputs: `ScreenX` (float), `ScreenY` (float).
- Outputs: `bSuccess` (boolean), `WorldPos` (vector).
- Logic:
```
ok = Deproject Screen Position To World(ScreenX, ScreenY) → WorldLocation (Loc), WorldDirection (Dir)
If NOT ok, or Abs(Dir.Y) < 0.0001 → bSuccess = false ; WorldPos = (0,0,0) ; return
t = (Cfg.PlayPlaneY - Loc.Y) / Dir.Y
P = Loc + Dir * t
WorldPos = (P.X, Cfg.PlayPlaneY, P.Z) ; bSuccess = true
```
Use this ray-plane formula. Do **not** use `Loc.X`/`Loc.Z` directly and do **not** use a bounded line-plane node: for an orthographic camera, the returned origin may be far from the plane.

**`ClampToPlayfield`**
- Input: `InPos` (vector). Output: `OutPos` (vector).
- Logic: `OutPos.X = Clamp(InPos.X, Cfg.PlayfieldMinX, Cfg.PlayfieldMaxX)`; `OutPos.Y = Cfg.PlayPlaneY`; `OutPos.Z = Clamp(InPos.Z, Cfg.CrosshairMinZ, Cfg.PlayfieldMaxZ)`.

**`RequestFire`**
- Input: `BaseIndex` (integer).
- Logic:
```
FireCount = FireCount + 1
LastFirePos = CrosshairWorldPos
BaseName = (BaseIndex == -1) ? "AUTO" : (BaseIndex == 0) ? "LEFT" : (BaseIndex == 1) ? "CENTER" : (BaseIndex == 2) ? "RIGHT" : "INVALID"
DDLog("Fire request base=" + BaseName + " X=" + Round(LastFirePos.X) + " Z=" + Round(LastFirePos.Z) + " n=" + FireCount, "Input", false)
Draw Debug Point(Position = LastFirePos, Size = 24, Color = Red (1,0,0), Persistent = false, Duration = 1.0)
```

**`ReportTest`**
- Inputs: `TestName` (string), `bPass` (boolean), `ExpectedX`, `ExpectedZ`, `ActualX`, `ActualZ` (floats).
- Logic:
```
If bPass → TestPassCount = TestPassCount + 1
DDLog(TestName + ": " + (bPass ? "PASS" : "FAIL") + " expected=(" + Round(ExpectedX) + "," + Round(ExpectedZ) + ") actual=(" + Round(ActualX) + "," + Round(ActualZ) + ")", "Test", NOT bPass)
```

**`RunSelfTest`** (no inputs)
```
(W, H) = Get Viewport Size   (convert to float)
If W <= 0 or H <= 0 → DDLog("Viewport size invalid", "Test", true) ; return
Aspect = Cfg.GetPlayfieldWidth() / Cfg.GetPlayfieldHeight()
If (W / H) >= Aspect → Hc = H ; Wc = H * Aspect
Else                 → Wc = W ; Hc = W / Aspect
C  = Cfg.GetPlayfieldCenter()
PW = Cfg.GetPlayfieldWidth()  ; PH = Cfg.GetPlayfieldHeight()
TestPassCount = 0
DDLog("SelfTest start viewport=" + Round(W) + "x" + Round(H), "Test", false)

T1  (sx, sy) = (W/2, H/2)                       expected = (C.X,        C.Z)
T2  (sx, sy) = (W/2 + Wc/4, H/2 - Hc/4)         expected = (C.X + PW/4, C.Z + PH/4)
T3  (sx, sy) = (W/2 - Wc/4, H/2 + Hc/4)         expected = (C.X - PW/4, C.Z - PH/4)
    For each: (ok, P) = ScreenToPlane(sx, sy)
              bPass = ok AND Abs(P.X - expectedX) <= 3 AND Abs(P.Z - expectedZ) <= 3
              ReportTest("T1 deproject center" / "T2 deproject upper-right" / "T3 deproject lower-left", bPass, expectedX, expectedZ, P.X, P.Z)

T4  Q = ClampToPlayfield((5000, 0, -300))   expected = (Cfg.PlayfieldMaxX, Cfg.CrosshairMinZ)
    bPass = Abs(Q.X - expectedX) <= 0.5 AND Abs(Q.Z - expectedZ) <= 0.5 ; ReportTest("T4 clamp low-right", ...)

T5  Q = ClampToPlayfield((-5000, 0, 9000))  expected = (Cfg.PlayfieldMinX, Cfg.PlayfieldMaxZ)
    ReportTest("T5 clamp high-left", ...)

T6  Before = FireCount ; RequestFire(-1)
    bPass = (FireCount == Before + 1)
    If bPass → TestPassCount = TestPassCount + 1
    DDLog("T6 fire path: " + (bPass ? "PASS" : "FAIL"), "Test", NOT bPass)
    FireCount = 0 ; LastFirePos = (0,0,0)       // so the HUD starts clean

DDLog("SelfTest done: " + TestPassCount + "/6 PASS", "Test", TestPassCount != 6)
```
With the Phase 0 config these resolve to expected points (0, 1200), (800, 1800), (-800, 600), then (1600, 100) and (-1600, 2400).

**5d. `Event BeginPlay` additions** (append after the existing Phase 0 logic; put the remainder inside the valid-config branch, per tooling note 4):
```
Cfg = Get Game Mode → Cast To BP_DDGameMode → GetConfig
Branch(IsValid(Cfg))
  False → DDLog("No config", "Input", true)
  True  →
    Set Input Mode Game And UI(In Widget To Focus = none, In Mouse Lock Mode = Lock Always, Hide Cursor During Capture = false)
    C = Cfg.GetPlayfieldCenter()
    Crosshair = Spawn Actor From Class(BP_Crosshair, Location = C, Rotation = (0,0,0), Scale = (1,1,1),
                                       Collision Handling Override = Always Spawn, Owner = self)
    CrosshairWorldPos = C
    DDLog("Crosshair spawned", "Input", false)
    Branch(bSelfTest) True → Delay(0.5) → RunSelfTest()
```

**5e. `Event Tick`**
```
If NOT IsValid(Crosshair) OR NOT IsValid(Cfg) → return
got = Get Mouse Position → (MX, MY)
If got:
    (ok, P) = ScreenToPlane(MX, MY)
    If ok → CrosshairWorldPos = ClampToPlayfield(P)
Crosshair.SetActorLocation(CrosshairWorldPos)      // always, even when the mouse is unavailable
```

**5f. Input events** (create with `create_node`, see tooling note 2; use the **Pressed** exec pin only)

| Key event node | Action |
|---|---|
| Left Mouse Button | `RequestFire(-1)` |
| One (`1`) | `RequestFire(0)` |
| Two (`2`) | `RequestFire(1)` |
| Three (`3`) | `RequestFire(2)` |

The existing `H` handling stays as it is.

**Check:** compiles clean; all functions listed above exist with the exact names; the Phase 0 log lines and `H` handling still exist in the graph.

---

### Step 6 — `BP_DDHUD` additions

Read the existing graphs first.

**Variable:** `OwnerPC` — object reference `BP_DDPlayerController` (not editable).

**Event BeginPlay (append):** `OwnerPC = Cast To BP_DDPlayerController(Get Owning Player Controller)`. Keep the Phase 0 `DDLog("HUD BeginPlay OK", ...)`.

**Event ReceiveDrawHUD (replace the single Phase 0 line with these):**
```
Line 1 (always):  Draw Text("DOME DEFENSE | PHASE 1 | CROSSHAIR", White, X=20, Y=20, Font=None, Scale=1.5, ScalePosition=false)

If NOT IsValid(OwnerPC): OwnerPC = Cast To BP_DDPlayerController(Get Owning Player Controller)
If IsValid(OwnerPC):
  Line 2: Draw Text("CROSSHAIR X=" + Round(OwnerPC.CrosshairWorldPos.X) + " Z=" + Round(OwnerPC.CrosshairWorldPos.Z),
                    Color (0.8, 1.0, 1.0), X=20, Y=50, Scale=1.2)
  Line 3: If OwnerPC.FireCount == 0 → Draw Text("LAST FIRE none", Color (1.0, 0.8, 0.8), X=20, Y=75, Scale=1.2)
          Else → Draw Text("LAST FIRE X=" + Round(OwnerPC.LastFirePos.X) + " Z=" + Round(OwnerPC.LastFirePos.Z) + " COUNT=" + OwnerPC.FireCount,
                           Color (1.0, 0.8, 0.8), X=20, Y=75, Scale=1.2)
```
`Round` converts a float to an integer; append it to the string through an integer-to-string conversion.

**Check:** compiles clean.

---

### Step 7 — Save and run the self-test

1. **Save All** (the level must be saved before any level load).
2. `StartPIE`, wait about 3 seconds, read the `[DD]` log entries, call `CaptureViewport`, then `StopPIE`.
3. Fill in section 7.

---

## 7. Acceptance tests

### A. Static checks
- [ ] **A1** New assets exist: `/Game/DomeDefense/Actors/BP_Crosshair` and `/Game/DomeDefense/Materials/MI_DD_Crosshair`. No other new assets.
- [ ] **A2** Every Blueprint (`PDA_DDConfig`, `BP_Crosshair`, `BP_DDPlayerController`, `BP_DDHUD`, `BP_PlayfieldDebug`) compiles with 0 errors and 0 warnings.
- [ ] **A3** `DA_DDConfig.CrosshairMinZ = 100.0`; the other 25 values are unchanged.
- [ ] **A4** `Ground` has Collision Enabled = NoCollision (read back).
- [ ] **A5** `BP_DDPlayerController` has Start With Tick Enabled = true and Show Mouse Cursor = true; `BP_Crosshair` has tick disabled.
- [ ] **A6** The level's placed actors are unchanged from Phase 0 (the crosshair is not placed in the level).

### B. Log checks (agent can verify)
Line order is **not** guaranteed. Match on names and numbers. Wait at least 1 second after PIE start (the self-test begins after a 0.5 s delay).

- [ ] **B1** All seven Phase 0 lines still appear (GameMode, Config OK, Camera Configured, PlayerController, View target, HUD, PlayfieldDebug ready).
- [ ] **B2** `[DD][Input] Crosshair spawned`
- [ ] **B3** `[DD][Test] SelfTest start viewport=<W>x<H>`
- [ ] **B4** `[DD][Test] T1 deproject center: PASS expected=(0,1200) actual=(0,1200)` (actual within 3 units)
- [ ] **B5** `T2 deproject upper-right: PASS expected=(800,1800) ...`
- [ ] **B6** `T3 deproject lower-left: PASS expected=(-800,600) ...`
- [ ] **B7** `T4 clamp low-right: PASS expected=(1600,100) actual=(1600,100)`
- [ ] **B8** `T5 clamp high-left: PASS expected=(-1600,2400) actual=(-1600,2400)`
- [ ] **B9** `[DD][Input] Fire request base=AUTO X=<int> Z=<int> n=1` (logged by T6) and `T6 fire path: PASS`
- [ ] **B10** `[DD][Test] SelfTest done: 6/6 PASS`
- [ ] **B11** No log line containing `ERROR`, `Accessed None`, or `Blueprint Runtime Error`.

### C. Visual checks (screenshot of the PIE viewport, agent or human)
If you cannot capture a screenshot, mark C as NEEDS-HUMAN.

- [ ] **C1** Phase 0 visuals unchanged: black background, yellow bounds, brown ground with white line, red axis right, blue axis up, 3 cyan boxes, 6 green boxes.
- [ ] **C2** A white **plus-shaped crosshair** is visible, drawn in front of the other items. With no real mouse over the viewport it sits at the exact screen center.
- [ ] **C3** HUD line 1 reads `DOME DEFENSE | PHASE 1 | CROSSHAIR`; line 2 reads `CROSSHAIR X=0 Z=1200` (or the mouse position if a real mouse is over the viewport); line 3 reads `LAST FIRE none`.
- [ ] **C4** The OS mouse cursor is not drawn over the playfield (or the *Deviations* section says the fallback was used).

### D. Interactive checks (human, PIE; click inside the viewport once so it has focus; `Shift+F1` frees the mouse, `Esc` stops PIE)
- [ ] **D1** Moving the mouse moves the crosshair with no visible lag, and the crosshair center sits exactly under the mouse tip.
- [ ] **D2** Mouse to the top-left corner of the playfield: HUD shows `X=-1600 Z=2400`. Bottom-right: `X=1600 Z=100` (the crosshair stops on the ground line).
- [ ] **D3** Resize the window to a different aspect ratio (wide, then tall): the crosshair still sits under the mouse inside the 4:3 area, and stays clamped when the mouse is in the black bars.
- [ ] **D4** Left click: log `Fire request base=AUTO ...`, a **red dot** appears at the crosshair for about 1 second, HUD line 3 updates with `COUNT` increasing by 1 per click.
- [ ] **D5** Keys `1`, `2`, `3` log `base=LEFT`, `base=CENTER`, `base=RIGHT` respectively.
- [ ] **D6** `H` hides and restores the Phase 0 overlay (yellow bounds, white line, axes, markers). The **first** press now logs `Overlay=OFF`. The crosshair and HUD stay visible.
- [ ] **D7** Moving the crosshair over the ground strip: the crosshair is drawn in front of it (not hidden).

### E. Hygiene
- [ ] **E1** Save All done; no unsaved assets.
- [ ] **E2** PIE is stopped.

---

## 8. Report format (the agent's final message)

```
## Phase 1 report
**Engine version:** x.y.z
**MCP tools used:** (list)

### Assets created or changed
(table: path | type | change | compiled clean yes/no)

### Acceptance results
A1 PASS/FAIL ... through E2 PASS/FAIL/NEEDS-HUMAN

### Log excerpt
(paste the [DD] lines from PIE, including all [DD][Test] lines)

### Screenshot notes
(what you saw; where the crosshair was; HUD text as read)

### Deviations
(every place you did not follow this document, why, and the fallback used)

### Open questions / risks
(anything ambiguous or surprising)
```

---

## 9. What Phase 1 hands to Phase 2

| Provided | How Phase 2 uses it |
|---|---|
| `BP_DDPlayerController.CrosshairWorldPos` | The always-valid, clamped target point for every shot. |
| `BP_DDPlayerController.RequestFire(BaseIndex)` | Phase 2 extends the body to choose a base, check ammo, and spawn an ABM. Keep the existing log line and the `FireCount` / `LastFirePos` updates. |
| Base index convention (`-1` auto, `0` Left, `1` Center, `2` Right) | Used by `BP_MissileBase` and by the nearest-base selection. |
| `ScreenToPlane`, `ClampToPlayfield` | Reused whenever a screen point must become a play-plane point. |
| Layering table (section 4.5) | Visual Y offsets for bases, cities, ABMs, enemies, explosions. |
| Self-test convention (section 4.6) | Every later phase adds `[DD][Test]` lines the agent can verify without input. |
| Tooling notes (section 5) | Carried forward and extended each phase. |

---

## 10. Common pitfalls (check these if a test fails)

1. **T1 fails by a large amount or the crosshair is off-center:** the view target is not the playfield camera, `Constrain Aspect Ratio` was lost, or `ScreenToPlane` used `DeprojectMousePosition` instead of the passed-in coordinates.
2. **T2/T3 are mirrored (Z has the wrong sign):** screen Y grows downward. The expected Z for the upper-right point is `C.Z + PH/4`, not `C.Z − PH/4`.
3. **T2/T3 are off only in non-4:3 windows:** `Wc`/`Hc` (the letterboxed 4:3 area) were computed wrongly. If `W/H >= Aspect`, the height is the limiting side; otherwise the width is.
4. **`ScreenToPlane` fails (`bSuccess = false`):** a bounded line-plane node was used with a far-away ortho origin. Use the unbounded `t = (PlayPlaneY − Loc.Y) / Dir.Y` formula.
5. **Crosshair never appears:** it was placed at Y = 0 inside the ground cube (the visual bars need relative Y = +100), the material is missing, or the spawn failed because `Cfg` was invalid.
6. **Crosshair does not move in PIE:** `Start With Tick Enabled` is false on the controller, `Get Mouse Position` returns false because the viewport has no focus (click inside it once), or the input mode node was not called.
7. **Clicks do nothing:** the key-event node is wired to **Released** instead of **Pressed**, or the viewport has no focus.
8. **The OS cursor is still visible:** `Default Mouse Cursor` / `Current Mouse Cursor` were not set to None. Use the fallback in Step 5a.
9. **Compile error "Target must have a connection" on `DDLog`:** the node was created as an instance call. Recreate it from `find_node_types` as a static library call.
10. **HUD lines 2 and 3 are missing:** `OwnerPC` cast failed. The HUD retries the cast in `ReceiveDrawHUD`; check that `BP_DDGameMode.Player Controller Class` is `BP_DDPlayerController`.
11. **H toggle regresses:** a Phase 0 node was deleted while editing the controller. Compare with the pre-edit graph you read first.
