# Phase 1.1 — Full-Screen Playfield and Tap Input (patch)

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Depends on** | Phase 1 accepted (crosshair, `RequestFire`, self-test) |
| **Phase goal** | (1) The playfield fills the **entire window at any aspect ratio**, with no bars. (2) The base/city layout is proportional to height and always fits. (3) One-finger tap and mouse click fire at a screen point. (4) A marker shows each shot. |

This patch **supersedes** these parts of Phase 1: the fixed 4:3 letterboxed field, the `1`/`2`/`3` keys, the base index on `RequestFire`, the red debug point, and the self-test (rewritten).

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **No bars, no aspect caps.** Do not add letterboxing, pillarboxing, a maximum aspect ratio, or any aspect clamp. The camera must use the whole window.
3. **Existing logic must survive** unless a step says to change or delete it. Read each Blueprint's current graphs (`read_graph_dsl`) before editing.
4. **Compile after every Blueprint change.** Every Blueprint ends with no errors and no warnings.
5. **Run each step's "Check"** before moving on.
6. **You cannot press keys, click, touch, or resize in PIE.** The self-test (Step 8) exercises the logic with synthetic inputs, including six aspect ratios. Interactive checks (group D) are **NEEDS-HUMAN**; touch is **NEEDS-DEVICE**.
7. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
8. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after each test. **Save before loading a level.**
9. **Read section 5 (Tooling notes) before building graphs.**

---

## 2. Scope

### In scope
- Config changes (reference height, fit aspect, proportional layout).
- A runtime **visible area** computed from the window aspect (in `BP_DDGameMode`).
- Camera, debug overlay, ground, crosshair clamp, and HUD updated to use it.
- Tap (touch) and click fire with a debounce; `1`/`2`/`3` removed.
- `BP_TapMarker` (a real actor, not a debug draw).
- Rewritten self-test.

### Out of scope (do NOT create)
- Bases, ABMs, ammo, explosions, enemies, cities as actors (Phase 2 onward).
- Multi-touch, drag, or hold behavior.
- UMG, sounds, particles, imported art.
- Packaging and Android project settings (human notes, section 10).

---

## 3. Design (decided; do not change)

### 3.1 The visible area uses the whole window
- **Reference height:** 1800 units (`PlayfieldHeight`). The layout is defined against it.
- **Window aspect:** `Aspect = ViewportW / ViewportH`, never clamped.
- **If `Aspect >= FitAspect` (1.25):** visible height = 1800, visible width = `1800 × Aspect`. Extra width beyond the layout is playable space.
- **If `Aspect < FitAspect`:** visible width stays `1800 × FitAspect` = 2250 and visible height grows to `2250 / Aspect` (more sky above; the ground stays at the bottom). The layout then looks proportionally smaller, but nothing is cut off and nothing is barred.
- The visible area is always centered on X = 0, with Z from 0 up to the visible height.

| Window aspect | Visible width × height | Notes |
|---|---|---|
| 0.8 (portrait-ish) | 2250 × 2812.5 | more sky |
| 1.0 | 2250 × 2250 | more sky |
| 1.25 (5:4) | 2250 × 1800 | the layout's tightest fit |
| 1.333 (4:3) | 2400 × 1800 | |
| 1.6 (16:10) | 2880 × 1800 | |
| 1.778 (16:9) | 3200 × 1800 | |
| 2.222 (20:9) | 4000 × 1800 | |
| 4.0 | 7200 × 1800 | wide playable space beside the bases |

### 3.2 The layout is proportional to height and fits
Nine slots in a row at equal spacing: `B c c c B c c c B`.
`Spacing = LayoutSpacingFraction × PlayfieldHeight = 0.14 × 1800 = 252`.

| Object | Slot | X |
|---|---|---|
| Base Left | −4 | −1008 |
| Cities | −3, −2, −1 | −756, −504, −252 |
| Base Center | 0 | 0 |
| Cities | +1, +2, +3 | 252, 504, 756 |
| Base Right | +4 | 1008 |

The outer base's far edge is at `1008 + 60 = 1068`. At the fit limit (5:4) the visible half-width is `1125`, so **the whole layout is on screen at every aspect ratio** (57 units of margin at the tightest). Layout math would allow about 1.19:1; 1.25 is used to keep a visible margin.

### 3.3 Two areas (keep them apart)
| Name | Where | Meaning |
|---|---|---|
| **Guaranteed zone** | Config: `PlayfieldMinX/MaxX` = ±1125, `PlayfieldMinZ/MaxZ` = 0 and 1800 | Always on screen at every aspect ratio. The layout lives inside it. `GetPlayfieldWidth()` = 2250, `GetPlayfieldHeight()` = 1800, `GetPlayfieldCenter()` = (0, 0, 900). |
| **Visible area** | `BP_DDGameMode` variables `VisibleMinX/MaxX/MinZ/MaxZ` | What the camera shows right now. Used for the camera, the clamp, the ground line, and (later) spawning and flight bounds. |

### 3.4 Ground
One wide cube, **20000 units wide**, centered at X = 0. It is wider than any plausible visible area (aspect up to 11:1), so it always spans the window edge to edge with no runtime resizing. Its top stays at Z = 100.

### 3.5 Input
- **One finger only.** Fire on touch-down (Pressed) of finger `Touch1`. Ignore every other finger. Ignore Released and Moved.
- A **mouse left click** (Pressed) behaves identically.
- **No keys fire.** (`H` still toggles the debug overlay.)
- A tap below the ground line is **moved up to the ground line** (`CrosshairMinZ`); it still fires.
- **Debounce 0.05 s:** a fire request within 0.05 s of an accepted one is dropped. This protects against a platform delivering both a touch event and an emulated mouse event for one tap.
- The mouse-following crosshair is a **desktop-only aid**: not spawned on Android or iOS.

### 3.6 Tap marker
- The marker is a **real actor** (`BP_TapMarker`), so it also shows in shipped builds.
- **Final behavior (Phase 2):** the marker stays at the target until the rocket (ABM) reaches it, then disappears. Each shot has its own marker.
- **Stand-in for this phase (no rockets exist yet):** the marker auto-expires after 1.0 s. `BP_TapMarker` has a `bAutoExpire` flag (default true) and a `Dismiss()` function. Phase 2 sets `bAutoExpire = false` and calls `Dismiss()` on arrival.

### 3.7 Layering addition
| Layer | Visual component Y offset |
|---|---|
| Tap marker | **+110** |

(Others as in Phase 1: ground 0, cities and bases +60, enemies +70, ABMs +80, explosions +90, crosshair +100.)

---

## 4. Conventions that still apply
- Screen right = +X, up = +Z, play plane Y = 0, camera at +Y looking toward −Y (Yaw −90).
- Logging only through `DDLog(Message, Category, bIsError)`. Coordinates in logs are **rounded integers**.
- Config functions and `GetConfig` are **not Pure**: use execution wires. Config access: `Get Game Mode → Cast To BP_DDGameMode → GetConfig`, once in `BeginPlay`, cached in a variable.
- Base identity (Phase 2): integer index, 0 = Left, 1 = Center, 2 = Right.
- `DDLog` is a **static** library call. If a compile error says "Target must have a connection", recreate the node from `find_node_types`.
- Gameplay feedback must not use debug-draw nodes (stripped from shipping builds).

---

## 5. Tooling notes

1. **Do not guess node IDs.** Use `find_node_types`. Call `get_graph_dsl_docs` once.
2. **Key, mouse-button and touch events cannot be written as DSL** (top-level forms must be `event` or `fn`). Use `create_node` + `connect_pins`. Node IDs that worked in Phase 1: `Input|KeyboardEvents|H` (letters) and `Input|KeyboardEvents|1` (digits). Find the left-mouse and touch IDs with `find_node_types` (queries "Left Mouse Button" and "Touch").
3. **Store a value in a variable before you change it.** A value read in a later node is evaluated when that node runs, not where it sits in the exec chain. Phase 1's first fire-path test failed this way. Use a variable (for example `FireBefore`) assigned before the call.
4. **Latent nodes (`Delay`) only work in event graphs**, not in functions. Set Delay durations **explicitly** (the default is 0.2 s).
5. **Sequence pins** are `then_0`, `then_1`. **No nodes after a Branch/return** in the same chain.
6. **Properties not settable through ObjectTools:** `bIsPureFunc`, `BlueprintType`, `ActorLabel`. Do not try to set Pure flags.
7. **PIE tools:** `StartPIE`, `StopPIE`, `IsPIERunning`, `GetLogEntries`, `CaptureViewport`, `CaptureEditorImage`. `CaptureViewport` showed a perspective view in Phase 1; use `CaptureEditorImage` for the play view.
8. **"max tick rate 3"** appears in the PIE log when the editor window is not focused. The self-test is written to cope with it.

---

## 6. Implementation steps

### Step 1 — Config (`PDA_DDConfig` / `DA_DDConfig`)

**1a. Add variables** to `PDA_DDConfig` (Instance Editable):

| Category | Variable | Type | Default |
|---|---|---|---|
| Playfield | `FitAspect` | float | `1.25` |
| Layout | `LayoutSpacingFraction` | float | `0.14` |

**1b. Change values** (on the class defaults **and** read back on `DA_DDConfig`; set on the instance if it still shows the old value):

| Variable | Old | **New** |
|---|---|---|
| `PlayfieldMinX` | −1600 | **−1125** |
| `PlayfieldMaxX` | 1600 | **1125** |
| `PlayfieldMaxZ` | 2400 | **1800** |
| `BaseLeftX` | −1400 | **−1008** |
| `BaseRightX` | 1400 | **1008** |
| `CityX` | [−1050, −750, −450, 450, 750, 1050] | **[−756, −504, −252, 252, 504, 756]** |

Everything else stays (`PlayfieldMinZ` 0, `BaseCenterX` 0, `GroundTopZ` 100, `CameraDistanceY` 3000, `CrosshairMinZ` 100, and so on).

**Check:** `GetPlayfieldWidth()` = 2250, `GetPlayfieldHeight()` = 1800, `GetPlayfieldCenter()` = (0, 0, 900). `PDA_DDConfig` compiles clean.

---

### Step 2 — Level actors

| Actor | Change |
|---|---|
| `Ground` | Scale **(200, 1, 1)** (20000 wide). Location stays (0, 0, 50). Collision stays NoCollision. |
| `PlayfieldCamera` | Location **(0, 3000, 900)**. Rotation stays (0, 0, −90). |

Add no actors. Save the level.

**Check:** read back both transforms.

---

### Step 3 — Marker material and `BP_TapMarker`

**3a.** Material Instance `MI_DD_TapMarker` (parent `M_DD_Unlit`) in `/Game/DomeDefense/Materials/`, `Color = (1.0, 0.85, 0.2, 1)`.

**3b.** Blueprint `BP_TapMarker` (parent `Actor`) in `/Game/DomeDefense/Actors/`:
- Root: `Scene` component `Root`.
- Component `Dot`: StaticMeshComponent, mesh `/Engine/BasicShapes/Sphere`, relative location (0, **110**, 0), relative scale (0.5, 0.5, 0.5), material `MI_DD_TapMarker`, NoCollision (read back the collision-enabled flag, not only the preset).
- Variables: `bAutoExpire` (boolean, Instance Editable, default **true**), `Life` (float, Instance Editable, default **1.0**).
- Function `Dismiss` (no inputs, non-Pure): `Destroy Actor`.
- `Event BeginPlay`: `Branch(bAutoExpire)` True → `Set Life Span(Life)`.
- **Start With Tick Enabled = false.**

**Check:** compiles clean.

---

### Step 4 — `BP_DDGameMode`

Add variables (not Instance Editable):

| Variable | Type | Default |
|---|---|---|
| `VisibleMinX` | float | `-1600` |
| `VisibleMaxX` | float | `1600` |
| `VisibleMinZ` | float | `0` |
| `VisibleMaxZ` | float | `1800` |
| `CurrentAspect` | float | `1.777778` |

(The defaults describe 16:9, so actors whose `BeginPlay` runs before the first update still get sensible values.)

Add function **`RecomputeVisibleArea`** (inputs `W` integer, `H` integer; no outputs; non-Pure):
```
If W <= 0 OR H <= 0 → return
Aspect = float(W) / float(H)
VW = Config.GetPlayfieldHeight() * Max(Aspect, Config.FitAspect)
VH = VW / Aspect
CX = Config.GetPlayfieldCenter().X
VisibleMinX = CX - VW / 2
VisibleMaxX = CX + VW / 2
VisibleMinZ = Config.PlayfieldMinZ
VisibleMaxZ = Config.PlayfieldMinZ + VH
CurrentAspect = Aspect
```
Change nothing else. The existing `Config OK:` line keeps its code; its numbers become `Width=2250.0 Height=1800.0 GroundTopZ=100.0 Cities=6`.

**Check:** compiles clean; the five variables show the defaults above.

---

### Step 5 — `BP_PlayfieldCamera`

Read the current graph. Add variables `GM` (`BP_DDGameMode` reference) and `Cfg` (`PDA_DDConfig` reference).

**Class defaults:** **Constrain Aspect Ratio = false** (this changes Phase 0, and it is what removes bars: the camera now uses the window's own aspect and `Ortho Width` defines the horizontal extent). Orthographic, near/far clip and exposure settings as in Phase 0. Tick stays disabled.
**Fallback:** if Constrain Aspect Ratio cannot be turned off, leave it on, call `Set Aspect Ratio(GM.CurrentAspect)` in `ApplyArea`, and report any bars seen under *Deviations*.

Add function **`ApplyArea`** (no inputs; non-Pure):
```
VW = GM.VisibleMaxX - GM.VisibleMinX
Set Actor Location( (Cfg.GetPlayfieldCenter().X,
                     Cfg.PlayPlaneY + Cfg.CameraDistanceY,
                     (GM.VisibleMinZ + GM.VisibleMaxZ) / 2) )
CameraComponent.Set Ortho Width(VW)
DDLog("Area applied OrthoWidth=" + Round(VW) + " Aspect*100=" + Round(GM.CurrentAspect * 100), "Camera", false)
```
**Event BeginPlay (replace the body after the config fetch):** cache `GM` (`Get Game Mode → Cast To BP_DDGameMode`) and `Cfg` (`GM.GetConfig`); if either is invalid → `DDLog("No config", "Camera", true)` and stop. Otherwise: `Set Actor Rotation(Roll 0, Pitch 0, Yaw −90)`, call `ApplyArea()`, then `DDLog("Configured Loc=" + <actor location> + " OrthoWidth=" + <ortho width>, "Camera", false)`.

**Check:** compiles clean.

---

### Step 6 — `BP_PlayfieldDebug`

Add a variable `GM` (`BP_DDGameMode` reference), cached in `BeginPlay` next to `Cfg`. Replace the `Event Tick` drawing with this list (all `Duration = 0.0`, every point at `Y = Cfg.PlayPlaneY`). Let `G = Cfg.GroundTopZ`, `C = Cfg.GetPlayfieldCenter()`.

| # | Element | Geometry | Color | Thickness |
|---|---|---|---|---|
| 1 | **Visible area** frame | rectangle `GM.VisibleMinX..VisibleMaxX` × `GM.VisibleMinZ..VisibleMaxZ` | Yellow (1,1,0) | 6 |
| 2 | **Guaranteed zone** frame | rectangle `Cfg.PlayfieldMinX..MaxX` × `Cfg.PlayfieldMinZ..MaxZ` | Orange (1,0.5,0) | 3 |
| 3 | Ground line | `(GM.VisibleMinX, G)` → `(GM.VisibleMaxX, G)` | White | 3 |
| 4 | X axis gizmo | `C` → `C + (300,0,0)` | Red | 6 |
| 5 | Z axis gizmo | `C` → `C + (0,0,300)` | Blue (0,0.4,1) | 6 |
| 6 | 3 base markers | Draw Debug Box at `(BaseX, Y, G+50)`, extent `(50,5,50)`, for `BaseLeftX`, `BaseCenterX`, `BaseRightX` | Cyan | 4 |
| 7 | 6 city markers | Draw Debug Box at `(CityX[i], Y, G+40)`, extent `(60,5,40)` | Green | 4 |

Keep `bShowOverlay`, `ToggleOverlay` (flip first, then log), and the `BeginPlay` log.

**Check:** compiles clean.

---

### Step 7 — `BP_DDPlayerController`

Read the existing graphs first.

**7a. Delete** the key events for `1`, `2`, `3`, the Phase 1 `RequestFire(BaseIndex)` body, and the `Draw Debug Point` call. **Keep** `H`, `ScreenToPlane`, `ReportTest`, the input-mode setup, and the crosshair follow.

**7b. New variables**

| Variable | Type | Default |
|---|---|---|
| `GM` | reference `BP_DDGameMode` | none |
| `PlayfieldCamera` | reference `BP_PlayfieldCamera` | none |
| `LastViewportW`, `LastViewportH` | integer | 0 |
| `LastFireTime` | float | `-1000.0` |
| `FireBefore` | integer | 0 (used by the self-test) |
| `bDesktopCrosshair` | boolean, Instance Editable | `true` |

**7c. `Event BeginPlay` changes** (inside the valid-config branch, after `Cfg` is set)
- `GM` = the cast game mode (reuse the cast already made for `Cfg`).
- `PlayfieldCamera = Get Actor Of Class(BP_PlayfieldCamera)`.
- Make the crosshair spawn conditional: `bUseCrosshair = bDesktopCrosshair AND NOT (Get Platform Name == "Android" OR Get Platform Name == "IOS")`. If true, spawn `BP_Crosshair` at `Cfg.GetPlayfieldCenter()` as in Phase 1 and log `Crosshair spawned`. If false, skip and log `Crosshair disabled on this platform`.
- Self-test chain: `Branch(bSelfTest)` True → `Delay(0.5)` → `UpdateArea(Get Viewport Size)` → `Delay(0.5)` → `RunSelfTest()`.

**7d. Functions** (all non-Pure)

**`UpdateArea`** (inputs `W`, `H` integers)
```
If W <= 0 OR H <= 0 → return
GM.RecomputeVisibleArea(W, H)
If IsValid(PlayfieldCamera) → PlayfieldCamera.ApplyArea()
LastViewportW = W ; LastViewportH = H
DDLog("Viewport=" + W + "x" + H + " VisibleWidth=" + Round(GM.VisibleMaxX - GM.VisibleMinX) + " VisibleHeight=" + Round(GM.VisibleMaxZ - GM.VisibleMinZ), "Area", false)
```

**`ClampToPlayfield`** (changed: it now clamps to the **visible area**)
```
OutPos.X = Clamp(InPos.X, GM.VisibleMinX, GM.VisibleMaxX)
OutPos.Y = Cfg.PlayPlaneY
OutPos.Z = Clamp(InPos.Z, Cfg.CrosshairMinZ, GM.VisibleMaxZ)
```

**`RequestFire`** (inputs `ScreenX`, `ScreenY` floats; replaces the old signature)
```
Now = Get Game Time In Seconds
If (Now - LastFireTime) < 0.05 → return
(ok, P) = ScreenToPlane(ScreenX, ScreenY)
If NOT ok → return
LastFireTime = Now
Target = ClampToPlayfield(P)
FireCount = FireCount + 1
LastFirePos = Target
Spawn Actor From Class(BP_TapMarker, Location = Target, Collision Handling = Always Spawn)
DDLog("Fire request X=" + Round(Target.X) + " Z=" + Round(Target.Z) + " n=" + FireCount, "Input", false)
```

**7e. `Event Tick`: add at the top** (always runs, before the crosshair update):
```
(W, H) = Get Viewport Size
If W > 0 AND H > 0 AND (W != LastViewportW OR H != LastViewportH) → UpdateArea(W, H)
```
Then the Phase 1 crosshair update, unchanged.

**7f. Input events** (create with `create_node`; **Pressed** pin only)
- **Left Mouse Button:** `Get Mouse Position → (MX, MY)`; if it returned true → `RequestFire(MX, MY)`.
- **Touch** (the input-touch event node): from **Pressed**: `Branch(Finger Index == Touch1)` True → `RequestFire(Location.X, Location.Y)`. Ignore Released and Moved. Ignore all other finger indices.
- `H` stays. **No other keys.**

**7g. `RunSelfTest`** (function, no inputs)
```
(W, H) = Get Viewport Size  (as floats)
If W <= 0 OR H <= 0 → DDLog("Viewport size invalid", "Test", true) ; return
TestPassCount = 0
DDLog("SelfTest start viewport=" + Round(W) + "x" + Round(H), "Test", false)

// T1: aspect logic at six synthetic window sizes (no resize needed)
SampleOK(w,h):  GM.RecomputeVisibleArea(w,h)
                a = w / h ; expVW = Cfg.GetPlayfieldHeight() * Max(a, Cfg.FitAspect) ; expVH = expVW / a
                return Abs((GM.VisibleMaxX - GM.VisibleMinX) - expVW) <= 0.5
                   AND Abs((GM.VisibleMaxZ - GM.VisibleMinZ) - expVH) <= 0.5
pass1 = SampleOK(1600,1200) AND SampleOK(1920,1080) AND SampleOK(2400,1080)
        AND SampleOK(4000,1000) AND SampleOK(1000,1000) AND SampleOK(800,1000)
        (expected width x height: 2400x1800, 3200x1800, 4000x1800, 7200x1800, 2250x2250, 2250x2812.5)
GM.RecomputeVisibleArea(Round(W), Round(H))          // restore the real state
If pass1 → TestPassCount++
DDLog("T1 aspect logic: " + (pass1 ? "PASS" : "FAIL") + " (6 samples)", "Test", NOT pass1)

// Real state for T2-T4 (the whole window is the playfield, so pixels map linearly)
VW = GM.VisibleMaxX - GM.VisibleMinX ; VH = GM.VisibleMaxZ - GM.VisibleMinZ
CX = (GM.VisibleMinX + GM.VisibleMaxX)/2 ; CZ = (GM.VisibleMinZ + GM.VisibleMaxZ)/2
Tol = Max(3, 2 * VW / W)                              // two pixels in world units

T2  (sx,sy) = (W/2,        H/2)          expected = (CX,        CZ)
T3  (sx,sy) = (W/2 + W/4,  H/2 - H/4)    expected = (CX + VW/4, CZ + VH/4)
T4  (sx,sy) = (W/2 - W/4,  H/2 + H/4)    expected = (CX - VW/4, CZ - VH/4)
    each: (ok,P) = ScreenToPlane(sx,sy) ; bPass = ok AND Abs(P.X-expX) <= Tol AND Abs(P.Z-expZ) <= Tol
          ReportTest("T2 deproject center" / "T3 deproject upper-right" / "T4 deproject lower-left", bPass, expX, expZ, P.X, P.Z)

T5  Q = ClampToPlayfield((50000, 0, -300))     expected = (GM.VisibleMaxX, Cfg.CrosshairMinZ)   ReportTest("T5 clamp low-right", ...)
T6  Q = ClampToPlayfield((-50000, 0, 50000))   expected = (GM.VisibleMinX, GM.VisibleMaxZ)      ReportTest("T6 clamp high-left", ...)
    (tolerance 0.5 for T5 and T6)

T7  LastFireTime = -1000 ; FireBefore = FireCount ; RequestFire(W/2, H/2)
    bPass = (FireCount == FireBefore + 1) AND IsValid(Get Actor Of Class(BP_TapMarker))
    If bPass → TestPassCount++
    DDLog("T7 fire path: " + (bPass ? "PASS" : "FAIL"), "Test", NOT bPass)
    For each actor in Get All Actors Of Class(BP_TapMarker) → Dismiss()
    FireCount = 0 ; LastFirePos = (0,0,0) ; LastFireTime = -1000

T8  half = Cfg.GetPlayfieldWidth() / 2 ; outer = Max(Abs(Cfg.BaseLeftX), Abs(Cfg.BaseRightX)) + 60
    bPass = (Abs(Cfg.GetPlayfieldWidth() - Cfg.GetPlayfieldHeight() * Cfg.FitAspect) <= 0.5) AND (outer <= half)
    If bPass → TestPassCount++
    DDLog("T8 layout fits: " + (bPass ? "PASS" : "FAIL") + " outerEdge=" + Round(outer) + " limit=" + Round(half), "Test", NOT bPass)

T9  s = Cfg.LayoutSpacingFraction * Cfg.GetPlayfieldHeight()
    bPass = |BaseLeftX + 4s| <= 0.5 AND |BaseCenterX| <= 0.5 AND |BaseRightX - 4s| <= 0.5
            AND |CityX[k] - slot[k]*s| <= 0.5 for k = 0..5, slot = [-3,-2,-1,1,2,3]
    If bPass → TestPassCount++
    DDLog("T9 layout proportional: " + (bPass ? "PASS" : "FAIL") + " spacing=" + Round(s), "Test", NOT bPass)

DDLog("SelfTest done: " + TestPassCount + "/9 PASS", "Test", TestPassCount != 9)
```

**Check:** compiles clean; no `1`/`2`/`3` key events remain; the `H` handler still exists; no `Draw Debug Point` remains.

---

### Step 8 — `BP_DDHUD`

Keep lines 1 to 3 from Phase 1 (line 2 only if `OwnerPC.Crosshair` is valid). Add **line 4** (guard: `OwnerPC` and `OwnerPC.GM` valid):

`Draw Text("AREA  W=" + Round(GM.VisibleMaxX - GM.VisibleMinX) + "  H=" + Round(GM.VisibleMaxZ - GM.VisibleMinZ) + "  ASPECT*100=" + Round(GM.CurrentAspect * 100), Color (0.8, 1.0, 0.8), X=20, Y=100, Scale=1.2)`

**Check:** compiles clean.

---

### Step 9 — Save and run

1. **Save All** (save the level before anything that loads a level).
2. `StartPIE`, wait **about 4 seconds** (the self-test starts after about 1 second), `GetLogEntries` (filter `[DD]`), `CaptureEditorImage`, `StopPIE`.
3. Fill in the report (section 8).

---

## 7. Acceptance tests

### A. Static checks
- [ ] **A1** New assets: `BP_TapMarker`, `MI_DD_TapMarker`. No others.
- [ ] **A2** Every changed Blueprint compiles with 0 errors and 0 warnings.
- [ ] **A3** `DA_DDConfig` values match Step 1 (including `FitAspect` and `LayoutSpacingFraction`). There is **no** `MaxAspect` or `MinAspect` variable.
- [ ] **A4** `Ground` scale is (200, 1, 1); `PlayfieldCamera` location is (0, 3000, 900); the camera component's Constrain Aspect Ratio reads back **false**.
- [ ] **A5** `BP_DDPlayerController` has no `1`/`2`/`3` key events and no `Draw Debug Point`; it has the left-mouse and touch events.
- [ ] **A6** No actors were added to the level.

### B. Log checks (agent)
Order is not guaranteed. Wait about 4 seconds.

- [ ] **B1** The seven Phase 0 lines still appear. `Config OK:` reads `Width=2250.0 Height=1800.0 GroundTopZ=100.0 Cities=6`. `[DD][Camera] Configured ...` no longer prints an aspect.
- [ ] **B2** `[DD][Camera] Area applied OrthoWidth=<int> Aspect*100=<int>` appears at least once.
- [ ] **B3** `[DD][Area] Viewport=<W>x<H> VisibleWidth=<int> VisibleHeight=<int>`, with `VisibleWidth = round(1800 × max(W/H, 1.25))` and `VisibleHeight = round(VisibleWidth / (W/H))`.
- [ ] **B4** `[DD][Input] Crosshair spawned` (desktop).
- [ ] **B5** `T1 aspect logic: PASS (6 samples)`.
- [ ] **B6** `T2`, `T3`, `T4` deproject lines are PASS.
- [ ] **B7** `T5 clamp low-right: PASS` and `T6 clamp high-left: PASS`.
- [ ] **B8** `[DD][Input] Fire request X=<int> Z=<int> n=1` (from T7) and `T7 fire path: PASS`.
- [ ] **B9** `T8 layout fits: PASS outerEdge=1068 limit=1125`.
- [ ] **B10** `T9 layout proportional: PASS spacing=252`.
- [ ] **B11** `[DD][Test] SelfTest done: 9/9 PASS`.
- [ ] **B12** No line containing `ERROR`, `Accessed None`, or `Blueprint Runtime Error`.

### C. Visual checks (editor image of the play view; NEEDS-HUMAN if you cannot capture it)
- [ ] **C1** Black background; **no black bars** on any side. A **yellow frame** hugs the window edges. An **orange frame** sits inside it, touching the top and bottom (at aspects of 1.25 or more) and inset from the left and right.
- [ ] **C2** The brown **ground strip spans edge to edge**, with a white line along its top edge.
- [ ] **C3** 3 cyan boxes (bases) and 6 green boxes (cities) in one row at equal spacing, all **inside the orange frame**, symmetric about the center. Red axis points right, blue axis points up, from the center.
- [ ] **C4** White crosshair plus at the center; HUD line 2 reads `CROSSHAIR X=0 Z=900` (at aspects of 1.25 or more); line 3 reads `LAST FIRE none`; line 4 reads `AREA W=<int> H=<int> ASPECT*100=<int>`.

### D. Interactive checks (human, PIE; click inside the viewport first; `Shift+F1` frees the mouse, `Esc` stops PIE)
- [ ] **D1** Mouse movement: the crosshair stays under the mouse tip (about 1 px offset is expected). At the top edge the HUD shows `Z` equal to the visible height; at the left and right edges `X = ±VisibleWidth/2`; at the bottom `Z=100`.
- [ ] **D2** Left click: log `Fire request ...`, a yellow-orange dot appears at the crosshair and disappears after about 1 s, HUD line 3 updates and `COUNT` goes up by 1. Keys `1`, `2`, `3` do nothing.
- [ ] **D3** **Resize the window** to about 5:4, 16:9, very wide (about 4:1), and nearly square / tall. Every time: **no bars**, the whole window is playfield, **both outer cyan boxes stay fully visible**, the ground spans edge to edge, HUD line 4 updates, and the crosshair stays under the mouse. In windows narrower than 5:4 there is extra sky above and the layout looks smaller.
- [ ] **D4** `H` hides and restores the overlay (both frames, white line, axes, markers). First press logs `Overlay=OFF`.
- [ ] **D5** The crosshair draws in front of the ground strip.

### D-DEVICE. Touch (Android device, or desktop with the project setting "Use Mouse for Touch" enabled)
- [ ] **T-1** Tapping anywhere fires: log line and marker at the tapped point; no mouse crosshair on the device.
- [ ] **T-2** A second finger held down at the same time is ignored.
- [ ] **T-3** No on-screen virtual joysticks are drawn (see section 10).
- [ ] **T-4** One tap produces exactly one fire request.

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped.

---

## 8. Report format (the agent's final message)

```
## Phase 1.1 report
**Engine version:** x.y.z
**MCP tools used:** (list)

### Assets created or changed
(table: path | type | change | compiled clean)

### Acceptance results
A1 PASS/FAIL ... through E1 PASS/FAIL/NEEDS-HUMAN/NEEDS-DEVICE

### Log excerpt
(all [DD] lines from PIE, including every [DD][Test], [DD][Area] and [DD][Camera] line)

### Screenshot notes
### Deviations
### Open questions / risks
```

---

## 9. What this hands to Phase 2

| Provided | How Phase 2 uses it |
|---|---|
| `BP_DDGameMode.VisibleMinX/MaxX/MinZ/MaxZ`, `CurrentAspect` | Bounds for ABM flight and enemy spawning, including the space beside the outer bases. |
| `BP_DDPlayerController.RequestFire(ScreenX, ScreenY)` | After `Target` is computed, Phase 2 picks the **nearest base with ammo** (straight-line distance; if empty, the next nearest; ties go to the center base), decrements ammo, and spawns an ABM that flies to `Target`. If no base has ammo, no ABM launches and **no marker is spawned** (move the marker spawn after base selection). Keep the log line and the debounce. |
| `BP_TapMarker` | Phase 2 sets `bAutoExpire = false` and calls `Dismiss()` when the ABM arrives. |
| Config layout (`BaseLeftX` ... `CityX`, `LayoutSpacingFraction`) | Where bases and cities stand. Do not hard-code positions. |
| Self-test convention | Phase 2 adds `[DD][Test]` lines for base selection, ammo and marker lifetime. |

---

## 10. Notes for the human (not agent steps)

1. **Android orientation:** Project Settings → Platforms → Android → Orientation = **Landscape**. Verify at packaging.
2. **Virtual joysticks:** Project Settings → Engine → Input → Mobile → **Default Touch Interface = None**, or Android draws on-screen joysticks over the game.
3. **Renderer lines** the agent wrote to `DefaultEngine.ini` (Lumen off) apply to the desktop renderer. Android uses the mobile renderer; revisit renderer settings before packaging, and again when realistic effects arrive.
4. **HUD text** uses fixed pixel sizes and will look tiny on a high-density phone. It will be reworked with the scoring phase.
5. **Touch is untested here.** The agent cannot verify it. Run the D-DEVICE checks on a device (or with "Use Mouse for Touch") before closing Phase 2.

---

## 11. Common pitfalls

1. **Black bars appear:** the camera's Constrain Aspect Ratio is still on. Turn it off (Step 5), or use the fallback and report it.
2. **Outer base cut off at some aspect:** layout values were not updated (Step 1b), or `PlayfieldMinX/MaxX` were left at ±1600.
3. **Narrow windows look wrong:** `RecomputeVisibleArea` must grow the **height** (`VH = VW / Aspect`) when the window is narrower than `FitAspect`; it must not shrink the width or clamp the aspect.
4. **T2–T4 fail by a few units only:** the tolerance must be two pixels in world units (`2 × VW / W`, at least 3).
5. **T7 fails:** `FireBefore` was read after the call, or `LastFireTime` was not reset to −1000 first (the debounce swallowed the request).
6. **Ground ends before the window edge:** scale was not set to 200 on X.
7. **One tap makes two markers:** the debounce is missing or `LastFireTime` is not being set.
8. **A second finger fires:** the `Finger Index == Touch1` branch is missing.
9. **Compile error "Target must have a connection" on `DDLog`:** recreate the node from `find_node_types`.
10. **Crosshair clamps to the old area:** `ClampToPlayfield` still reads `Cfg.PlayfieldMinX/MaxX` instead of `GM.VisibleMinX/MaxX`.
