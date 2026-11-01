# Phase 2 — Bases, Ammo and ABMs

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Depends on** | Phase 1.1 accepted (touch still NEEDS-DEVICE) |
| **Phase goal** | Three missile bases, each with 10 anti-ballistic missiles (ABMs). A tap or click launches an ABM from the **nearest base that still has ammo**. The ABM flies in a straight line at constant speed to the tapped point. The tap marker stays at that point until the ABM arrives, then both disappear. Each base shows its remaining ammo as a number under it. |

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **No bars, no aspect caps.** The whole window is the playfield (Phase 1.1). Do not add any.
3. **Existing logic must survive** unless a step says to change or delete it. Read each Blueprint's current graphs (`read_graph_dsl`) before editing.
4. **Compile after every Blueprint change.** Every Blueprint ends with no errors and no warnings.
5. **Run each step's "Check"** before moving on.
6. **You cannot press keys, click, touch, or resize in PIE.** The self-test (Step 8) exercises base selection, ammo, and a full ABM flight with synthetic inputs. Interactive checks (group D) are **NEEDS-HUMAN**; touch is **NEEDS-DEVICE**.
7. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
8. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after each test. **Save before loading a level.**
9. **Read sections 4 and 5 before building graphs.**

---

## 2. Scope

### In scope
- Config: one new value (`BaseLaunchOffsetZ`).
- `BP_MissileBase`: placeholder body, ammo count, ammo number under it, `SetAmmoDisplay`.
- `BP_ABM`: straight-line, constant-speed rocket that arrives exactly on the target.
- `BP_DDGameMode`: spawns the three bases at startup; `RefillBases`; `GetBaseName`; flight statistics for tests.
- `BP_DDPlayerController`: nearest-base selection, `LaunchAt`, new `RequestFire` body, debug key `R`.
- `BP_TapMarker`: stays until dismissed (the Phase 2 behavior).
- `BP_DDHUD`: title line only (no ammo on the HUD).
- Self-test extended to 15 tests.

### Out of scope (do NOT create)
- Explosions, damage, or anything that happens **at** arrival except removing the ABM and its marker (Phase 3 hooks in there).
- Enemy missiles, cities as actors (cities stay debug markers), base destruction, game over.
- Restocking ammo between waves (wave phase). `R` is a debug refill only.
- ABM trails, sounds, particles, imported art, ammo icons (art pass).
- Any UMG.

---

## 3. Design (decided; do not change)

### 3.1 Bases
- Three bases, spawned **at runtime** by `BP_DDGameMode` (not placed in the level), at the config positions: Left `BaseLeftX` (−1008), Center `BaseCenterX` (0), Right `BaseRightX` (1008), each at `Y = PlayPlaneY` and `Z = GroundTopZ` (100).
- Base identity is an integer: **0 = Left, 1 = Center, 2 = Right**. Names for logs: `LEFT`, `CENTER`, `RIGHT`.
- Each base starts with `AbmStartAmmo` (10).
- The **launch point** of a base is `base location + (0, 0, BaseLaunchOffsetZ)` = (X, 0, 170).
- Placeholder body: one cube (100 wide × 40 deep × 70 tall), steel gray-blue (distinct from the cyan debug markers).
- **Ammo display:** the remaining ammo is a **plain number** (white) drawn under the base, inside the brown ground strip. All of that goes through **one function on the base, `SetAmmoDisplay(Count)`**. The art pass will replace this number with one icon per missile left by changing only the body of `SetAmmoDisplay`. Nothing else may write to the ammo text.

### 3.2 Base selection (the rule)
When a shot is requested at `Target` (already clamped to the visible area):
1. Candidates are bases with `Ammo > 0`.
2. The winner is the candidate with the **smallest straight-line distance from its launch point to `Target`**.
3. Differences of 0.5 units or less count as a **tie**. Ties are resolved in this order: **Center, then Left, then Right**.
4. If no base has ammo, **nothing launches** and **no marker appears**.

### 3.3 ABM
- Spawned at the launch point of the chosen base. Flies in a **straight line at constant speed** to `Target`; no acceleration, no curve.
- Speed: Center base `AbmSpeedCenter` (3000 units/s), Left and Right bases `AbmSpeedSide` (1200 units/s).
- Movement is **time-based** (`DeltaSeconds`). Each tick moves `Speed × DeltaSeconds`; if that step reaches or passes the remaining distance, the ABM **snaps exactly to `Target`** and arrives (no overshoot).
- ABMs may fly through the whole visible area, including the extra width beside the outer bases.
- Several ABMs can be in flight at once.
- **Arrival:** record statistics on the game mode, dismiss the marker, log, and destroy the ABM. A comment marks the place where Phase 3 adds the explosion.

### 3.4 Tap marker
- One marker per accepted shot, spawned at `Target`, with **no timer** (`bAutoExpire = false` is now the class default).
- It disappears when its ABM arrives. Safety net: the ABM also dismisses its marker if it is destroyed for any other reason.
- No ammo → no marker.

### 3.5 Input (unchanged from Phase 1.1)
One finger only, fire on touch-down of `Touch1`; mouse left click fires identically; 0.05 s debounce; taps below the ground line are moved up to the ground line; the mouse crosshair exists on desktop only. **New desktop debug key: `R` refills every base to full ammo** (testing aid; the game has no restock yet).

### 3.6 Layering (Y offset of visual components)
| Layer | Y offset |
|---|---|
| Ground | 0 |
| Bases (body) and ammo number | +60 |
| Cities (later) | +60 |
| Enemies (later) | +70 |
| **ABMs** | **+80** |
| Explosions (later) | +90 |
| Crosshair | +100 |
| Tap marker | +110 |

---

## 4. Conventions as built (these replace earlier text where they differ)

| Topic | As built |
|---|---|
| Camera | With **Constrain Aspect Ratio off**, this engine treats **Ortho Width as the vertical extent**: `OrthoWidth = VisibleMaxZ − VisibleMinZ` (1800 on normal screens, larger on screens narrower than 5:4). The Phase 1.1 text that says "width" was wrong; the log line `Area applied OrthoWidth=<n>` prints the **height**. Do not change `ApplyArea`. |
| Placed-actor overrides | A **placed** actor's instance override beats the Blueprint default (the level camera had its own Constrain Aspect Ratio override). After changing a class default that matters for a **placed** actor, read the value back **on the placed instance**. Phase 2 spawns everything at runtime, so this should not bite here. |
| Visible area | `BP_DDGameMode` holds `VisibleMinX/MaxX/MinZ/MaxZ` and `CurrentAspect`, computed by `RecomputeVisibleArea(W, H)`. Visible height is 1800 on screens at least 5:4 wide; narrower screens get a wider-than-1800 height with the width fixed at 2250. **All flight bounds and clamps use the visible area**, not the orange "guaranteed zone" in config. |
| Reading values | A value read in a later node is evaluated **when that node runs**, not where it sits in the exec chain. **Store a value in a variable (or a local) before you change what it depends on.** In the self-test, compute each sample's result into its own variable before running the next sample. |
| Config | Config functions and `GetConfig` are **not Pure**: use execution wires. Access pattern: `Get Game Mode → Cast To BP_DDGameMode → GetConfig`, once in `BeginPlay`, cached. |
| Logging | Only through `DDLog(Message, Category, bIsError)`; it is a **static** call (recreate the node from `find_node_types` if the compile error says "Target must have a connection"). Coordinates in logs are rounded integers. Float and vector formats from the engine look like `3200.0` and `X=0.000 Y=3000.000 Z=1200.000`. |
| Self-test | `bSelfTest` (default true) starts a chain from `BeginPlay`; `ReportTest(...)` logs expected/actual pairs and counts passes in `TestPassCount`; `RunSelfTest` is a function; latent waits live in event graphs. |
| Fire path | `RequestFire(ScreenX, ScreenY)`: debounce 0.05 s → `ScreenToPlane` → `ClampToPlayfield` → (**new**) `LaunchAt(Target)`. |
| Phase 1.1 status | Accepted. Touch is untested (NEEDS-DEVICE). |
| Ammo icons | Planned for the art pass; see 3.1. |

---

## 5. Tooling notes

1. **Do not guess node IDs.** Use `find_node_types`. Call `get_graph_dsl_docs` once.
2. **Key events cannot be written as DSL.** Use `create_node` + `connect_pins`. Known IDs: `Input|KeyboardEvents|H`. Find the one for `R` with `find_node_types` (query "R").
3. **Latent nodes (`Delay`) only work in event graphs**, not in functions. Set Delay durations **explicitly**.
4. **Combining results of non-Pure function calls:** a Boolean AND cannot wrap calls that need exec wires. Call each function in sequence, store each result in its own variable, then combine the variables.
5. **Sequence pins** are `then_0`, `then_1`. **No nodes after a Branch/return** in the same chain.
6. **Spawn Actor From Class:** set **Collision Handling Override = Always Spawn**. After spawning, call the actor's `Init...` function in the same exec chain (do not rely on Expose-on-Spawn).
7. **Arrays:** use `Add`, `Clear`, `Get (a copy)` by index, `Length`, and `IsValid` on elements.
8. **Properties not settable through ObjectTools:** `bIsPureFunc`, `BlueprintType`, `ActorLabel`. For text render settings use `set_properties` with the component's real property names (`Text`, `WorldSize`, `HorizontalAlignment`, `VerticalAlignment`, `TextRenderColor`) and **read each one back**.
9. **PIE tools:** `StartPIE`, `StopPIE`, `IsPIERunning`, `GetLogEntries`, `CaptureViewport`, `CaptureEditorImage`. Use `CaptureEditorImage` for the play view.
10. **"max tick rate 3"** appears when the editor window is not focused. The self-test tolerances in this phase allow for slow frames.

---

## 6. Implementation steps

### Step 1 — Config

Add to `PDA_DDConfig` (Instance Editable, category `ABM`):

| Variable | Type | Default |
|---|---|---|
| `BaseLaunchOffsetZ` | float | `70.0` |

Read the value back on `DA_DDConfig`; set it on the instance if it does not show `70.0`. All other values stay.

**Check:** `DA_DDConfig.BaseLaunchOffsetZ = 70.0`; `PDA_DDConfig` compiles clean.

---

### Step 2 — Material instances

In `/Game/DomeDefense/Materials/` (parent `M_DD_Unlit`):
- `MI_DD_Base`: `Color = (0.60, 0.65, 0.75, 1)` (steel gray-blue; this replaces the reserved cyan so bases differ from the cyan debug markers).
- `MI_DD_Abm`: `Color = (0.30, 0.70, 1.00, 1)`.

**Check:** both exist; parent is `M_DD_Unlit`; colors read back.

---

### Step 3 — `BP_MissileBase`

Blueprint class (parent `Actor`) in `/Game/DomeDefense/Actors/`.

**Components** (root is a `Scene` component `Root`):

| Name | Type | Settings |
|---|---|---|
| `Body` | StaticMeshComponent | mesh `/Engine/BasicShapes/Cube`; relative location (0, **60**, 35); relative scale (1.0, 0.4, 0.7); material `MI_DD_Base`; NoCollision |
| `AmmoText` | TextRenderComponent | relative location (0, **60**, −50); relative rotation **(Roll 0, Pitch 0, Yaw 90)**; Text `10`; World Size `60`; Horizontal alignment Center; Vertical alignment Center; Text Render Color white |

The ground strip spans Z 0 to 100 and the base actor sits at Z = 100, so a relative Z of −50 puts the number in the middle of the strip. Yaw 90 turns the text to face the camera (+Y).

**Class defaults:** Start With Tick Enabled = **false**.

**Variables** (not Instance Editable): `BaseIndex` (integer, 0), `Ammo` (integer, 0), `LaunchPos` (vector, 0,0,0).

**Functions** (non-Pure):
- `SetAmmoDisplay` (input `Count` integer): `AmmoText.SetText(ToText(Count))`. **This is the only place that writes the ammo text.**
- `SetAmmo` (input `NewAmmo` integer): `Ammo = Max(NewAmmo, 0)`; `SetAmmoDisplay(Ammo)`.
- `ConsumeAmmo` (no inputs): if `Ammo > 0` → `SetAmmo(Ammo - 1)`.
- `InitBase` (inputs `Index` integer, `StartAmmo` integer, `LaunchOffsetZ` float): `BaseIndex = Index`; `LaunchPos = GetActorLocation + (0, 0, LaunchOffsetZ)`; `SetAmmo(StartAmmo)`.
- `GetAmmoDisplayText` (output `Text` string): returns `AmmoText.GetText().ToString()` (for the self-test).

**Check:** compiles clean. Read back `AmmoText`: text `10`, world size 60, both alignments Center, rotation yaw 90.

**Fallback (text render problems):** if the number is invisible or black in the play view (the scene has no lights), or cannot be shown, keep `SetAmmoDisplay` and `AmmoText` but make `SetAmmoDisplay` also store the value in `Ammo`, and have `BP_DDHUD` draw each base's `Ammo` with `Draw Text` at the **projected screen position** of `base location + (0,0,−50)` (`Project World to Screen`). Record it under *Deviations*. If the text is mirrored or edge-on, try Yaw −90 first and record the final value.

---

### Step 4 — `BP_ABM`

Blueprint class (parent `Actor`) in `/Game/DomeDefense/Actors/`.

**Components:** `Root` (Scene); `Dot` StaticMeshComponent: mesh `/Engine/BasicShapes/Sphere`, relative location (0, **80**, 0), relative scale (0.25, 0.25, 0.25), material `MI_DD_Abm`, NoCollision.

**Class defaults:** **Start With Tick Enabled = true.**

**Variables** (not Instance Editable): `Target` (vector), `Speed` (float), `Marker` (reference `BP_TapMarker`), `FromBase` (integer), `LaunchTime` (float), `bActive` (boolean, false), `GM` (reference `BP_DDGameMode`).

**Event BeginPlay:** `GM = Get Game Mode → Cast To BP_DDGameMode`.

**Function `Init`** (inputs `StartPos` vector, `TargetPos` vector, `FlightSpeed` float, `MarkerRef` reference `BP_TapMarker`, `BaseIdx` integer; non-Pure):
```
Target = TargetPos ; Speed = FlightSpeed ; Marker = MarkerRef ; FromBase = BaseIdx
SetActorLocation(StartPos)
LaunchTime = Get Game Time In Seconds
bActive = true
```

**Function `OnArrived`** (no inputs; non-Pure):
```
SetActorLocation(Target)
If IsValid(GM):
    GM.LastFlightTime = (Get Game Time In Seconds) - LaunchTime
    GM.LastArrivalPos = Target
    GM.ArrivalCount   = GM.ArrivalCount + 1
If IsValid(Marker) → Marker.Dismiss()
DDLog("Arrived base=" + GM.GetBaseName(FromBase) + " X=" + Round(Target.X) + " Z=" + Round(Target.Z), "ABM", false)
// PHASE 3 HOOK: spawn the explosion at Target here, before Destroy Actor.
Destroy Actor
```

**Event Tick(DeltaSeconds):**
```
If NOT bActive → return
Remaining = Distance(GetActorLocation, Target)
Step = Speed * DeltaSeconds
If Step >= Remaining → bActive = false ; OnArrived()
Else → Dir = (Target - GetActorLocation) / Remaining ; SetActorLocation(GetActorLocation + Dir * Step)
```

**Event EndPlay:** if `IsValid(Marker)` → `Marker.Dismiss()` (safety net).

**Check:** compiles clean.

---

### Step 5 — `BP_TapMarker`

Open `BP_TapMarker`. Set the class default **`bAutoExpire = false`**. Change nothing else (`Life`, `Dismiss`, the `Dot`).

**Check:** `bAutoExpire` reads back `false`; compiles clean.

---

### Step 6 — `BP_DDGameMode`

Read the existing graphs. Add variables (not Instance Editable):

| Variable | Type | Default |
|---|---|---|
| `Bases` | array of `BP_MissileBase` reference | empty |
| `LastFlightTime` | float | 0 |
| `LastArrivalPos` | vector | (0,0,0) |
| `ArrivalCount` | integer | 0 |

Add functions (non-Pure):

**`GetBaseName`** (input `Index` integer, output `Name` string): `-1 → "AUTO"`, `0 → "LEFT"`, `1 → "CENTER"`, `2 → "RIGHT"`, otherwise `"INVALID"`.

**`SpawnBases`** (no inputs). Three explicit blocks (no loop):
```
Bases.Clear()
Block for (Index 0, X = Config.BaseLeftX), then (1, Config.BaseCenterX), then (2, Config.BaseRightX):
    B = Spawn Actor From Class(BP_MissileBase, Location = (X, Config.PlayPlaneY, Config.GroundTopZ), Collision = Always Spawn)
    B.InitBase(Index, Config.AbmStartAmmo, Config.BaseLaunchOffsetZ)
    Bases.Add(B)
DDLog("Bases spawned n=" + Length(Bases), "Core", false)
```

**`RefillBases`** (input `bLog` boolean): for each of `Bases[0]`, `Bases[1]`, `Bases[2]` that is valid → `SetAmmo(Config.AbmStartAmmo)`; if `bLog` → `DDLog("Bases refilled", "Core", false)`.

**`Event BeginPlay` addition:** after the valid-config branch's existing logic, call `SpawnBases()`.

**Check:** compiles clean. The Phase 0 and 1.1 `BeginPlay` logs still exist in the graph.

---

### Step 7 — `BP_DDPlayerController`

Read the existing graphs first. Keep everything from Phase 1.1 (area updates, crosshair, `ScreenToPlane`, `ClampToPlayfield`, `ReportTest`, `RunSelfTest` structure, debounce, `H`).

**7a. New variables** (not Instance Editable)

| Variable | Type | Default |
|---|---|---|
| `SelBest` | integer | -1 |
| `SelBestDist` | float | 1000000 |
| `LastAbm` | reference `BP_ABM` | none |
| `LastMarker` | reference `BP_TapMarker` | none |
| `FlightTarget` | vector | (0,0,0) |
| `FlightExpected` | float | 0 |
| `FlightArrivalsBefore` | integer | 0 |
| `bFlightLaunched` | boolean | false |

**7b. New functions** (non-Pure)

**`ConsiderBase`** (inputs `Index` integer, `Target` vector):
```
B = GM.Bases[Index]
If IsValid(B) AND B.Ammo > 0:
    d = Distance(B.LaunchPos, Target)
    If d < SelBestDist - 0.5 → SelBest = Index ; SelBestDist = d
```

**`SelectBase`** (input `Target` vector; output `BaseIndex` integer):
```
SelBest = -1 ; SelBestDist = 1000000
ConsiderBase(1, Target)      // Center first: wins ties
ConsiderBase(0, Target)      // then Left
ConsiderBase(2, Target)      // then Right
return SelBest
```

**`LaunchAt`** (input `Target` vector; output `bLaunched` boolean):
```
Idx = SelectBase(Target)
If Idx == -1 → DDLog("No ammo X=" + Round(Target.X) + " Z=" + Round(Target.Z), "Input", false) ; return false
Base = GM.Bases[Idx]
FireCount = FireCount + 1 ; LastFirePos = Target
Marker = Spawn Actor From Class(BP_TapMarker, Location = Target, Collision = Always Spawn) ; LastMarker = Marker
Speed = (Idx == 1) ? Cfg.AbmSpeedCenter : Cfg.AbmSpeedSide
Abm = Spawn Actor From Class(BP_ABM, Location = Base.LaunchPos, Collision = Always Spawn) ; LastAbm = Abm
Abm.Init(Base.LaunchPos, Target, Speed, Marker, Idx)
Base.ConsumeAmmo()
AmmoLeft = Base.Ammo                            // assigned AFTER ConsumeAmmo
DDLog("Fire request base=" + GM.GetBaseName(Idx) + " X=" + Round(Target.X) + " Z=" + Round(Target.Z)
      + " n=" + FireCount + " ammo=" + AmmoLeft, "Input", false)
return true
```

**`RequestFire`** (change the body; inputs `ScreenX`, `ScreenY` floats unchanged):
```
Now = Get Game Time In Seconds
If (Now - LastFireTime) < 0.05 → return
(ok, P) = ScreenToPlane(ScreenX, ScreenY)
If NOT ok → return
LastFireTime = Now
LaunchAt(ClampToPlayfield(P))
```
(Remove the old marker spawn, `FireCount`/`LastFirePos` updates, and log from `RequestFire`; they live in `LaunchAt` now.)

**7c. Key `R`** (create with `create_node`, **Pressed** pin only): `GM.RefillBases(true)`. No other keys. `H` stays.

**7d. Self-test helpers** (non-Pure)

**`SelectCase`** (inputs `A0`, `A1`, `A2` integers, `TX`, `TZ` floats, `Expected` integer; output `bPass` boolean):
```
GM.Bases[0].SetAmmo(A0) ; GM.Bases[1].SetAmmo(A1) ; GM.Bases[2].SetAmmo(A2)
R = SelectBase((TX, Cfg.PlayPlaneY, TZ))
bPass = (R == Expected)
```

**7e. Self-test: changes to `RunSelfTest`**

Keep T1 to T6, T8 and T9 exactly as in Phase 1.1. **Remove the final `SelfTest done` log and the Phase 1.1 T7 block from this function**, then add the following in this order, after T9:

```
T10 bases spawned
    ok = (Length(GM.Bases) == 3)
         AND for i in {0,1,2}: IsValid(Bases[i]) AND Abs(Bases[i].X - {BaseLeftX, BaseCenterX, BaseRightX}[i]) <= 0.5
                               AND Abs(Bases[i].Z - Cfg.GroundTopZ) <= 0.5
                               AND Bases[i].Ammo == Cfg.AbmStartAmmo
                               AND Abs(Bases[i].LaunchPos.Z - (Cfg.GroundTopZ + Cfg.BaseLaunchOffsetZ)) <= 0.5
    If ok → TestPassCount++
    DDLog("T10 bases spawned: " + (ok ? "PASS" : "FAIL") + " ammo=" + Cfg.AbmStartAmmo, "Test", NOT ok)

T11 base selection (store each case in its own variable c1..c6, then combine)
    c1 = SelectCase(10,10,10, -700,  500, 0)      // nearest Left
    c2 = SelectCase(10,10,10,  700,  500, 2)      // nearest Right
    c3 = SelectCase(10,10,10,    0, 1200, 1)      // nearest Center
    c4 = SelectCase(10, 0,10,    0,  500, 0)      // center empty; Left/Right tie -> Left
    c5 = SelectCase( 0, 0,10, -700,  500, 2)      // only Right has ammo
    c6 = SelectCase( 0, 0, 0,    0,  500, -1)     // nothing left
    GM.RefillBases(false)
    ok = c1 AND c2 AND c3 AND c4 AND c5 AND c6
    If ok → TestPassCount++
    DDLog("T11 base selection: " + (ok ? "PASS" : "FAIL") + " (6 cases)", "Test", NOT ok)

T12 ammo accounting
    B = GM.Bases[0]
    B.SetAmmo(10) ; B.ConsumeAmmo() ; a1 = B.Ammo ; t1 = B.GetAmmoDisplayText()
    B.SetAmmo(0)  ; B.ConsumeAmmo() ; a2 = B.Ammo ; t2 = B.GetAmmoDisplayText()
    GM.RefillBases(false)
    ok = (a1 == 9) AND (t1 == "9") AND (a2 == 0) AND (t2 == "0")
    If ok → TestPassCount++
    DDLog("T12 ammo accounting: " + (ok ? "PASS" : "FAIL") + " shown=" + t1, "Test", NOT ok)

T13 no ammo
    SetAmmo(0) on all three bases
    before = FireCount ; mBefore = Length(Get All Actors Of Class(BP_TapMarker)) ; aBefore = Length(Get All Actors Of Class(BP_ABM))
    r = LaunchAt((0, Cfg.PlayPlaneY, 900))
    ok = (r == false) AND (FireCount == before)
         AND Length(Get All Actors Of Class(BP_TapMarker)) == mBefore AND Length(Get All Actors Of Class(BP_ABM)) == aBefore
    GM.RefillBases(false)
    If ok → TestPassCount++
    DDLog("T13 no ammo: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)

T7 fire path (now last in this function; replaces the Phase 1.1 T7)
    LastFireTime = -1000 ; FireBefore = FireCount ; RequestFire(W/2, H/2)
    ok = (FireCount == FireBefore + 1) AND IsValid(Get Actor Of Class(BP_TapMarker)) AND IsValid(Get Actor Of Class(BP_ABM))
    If ok → TestPassCount++
    DDLog("T7 fire path: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)
    For each actor in Get All Actors Of Class(BP_ABM)      → Destroy Actor
    For each actor in Get All Actors Of Class(BP_TapMarker) → Dismiss()
    GM.RefillBases(false) ; FireCount = 0 ; LastFirePos = (0,0,0) ; LastFireTime = -1000
```
(`ReportTest` is unchanged. `RunSelfTest` ends after T7; the summary line moves to `SelfTestFlightEnd`.)

**7f. New functions for the flight test** (non-Pure; no inputs)

**`SelfTestFlightStart`**
```
GM.RefillBases(false)
GM.Bases[1].SetAmmo(0) ; GM.Bases[2].SetAmmo(0)          // only the Left base can fire
FlightTarget = (1000, Cfg.PlayPlaneY, 1700)
FlightExpected = Distance(GM.Bases[0].LaunchPos, FlightTarget) / Cfg.AbmSpeedSide     // about 2.10 s
FlightArrivalsBefore = GM.ArrivalCount
bFlightLaunched = LaunchAt(FlightTarget)
```
**`SelfTestFlightMid`** (about 0.5 s after launch: the ABM must still be flying)
```
ok = bFlightLaunched AND IsValid(LastAbm) AND IsValid(LastMarker)
     AND GM.Bases[0].Ammo == Cfg.AbmStartAmmo - 1 AND GM.ArrivalCount == FlightArrivalsBefore
If ok → TestPassCount++
DDLog("T14 marker and ABM stay in flight: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)
```
**`SelfTestFlightEnd`** (about 3.0 s after launch)
```
arrived = (GM.ArrivalCount == FlightArrivalsBefore + 1)
posOK   = Distance(GM.LastArrivalPos, FlightTarget) <= 0.5
timeOK  = (GM.LastFlightTime >= FlightExpected - 0.05) AND (GM.LastFlightTime <= FlightExpected + 0.5)
goneOK  = NOT IsValid(LastAbm) AND NOT IsValid(LastMarker)
ok = arrived AND posOK AND timeOK AND goneOK
If ok → TestPassCount++
DDLog("T15 arrival: " + (ok ? "PASS" : "FAIL") + " flight=" + Round(GM.LastFlightTime * 1000) + "ms expected=" + Round(FlightExpected * 1000) + "ms", "Test", NOT ok)
GM.RefillBases(false) ; FireCount = 0 ; LastFirePos = (0,0,0) ; LastFireTime = -1000
DDLog("SelfTest done: " + TestPassCount + "/15 PASS", "Test", TestPassCount != 15)
```

**7g. `Event BeginPlay`: change the self-test chain** to:
`Branch(bSelfTest)` True → `Delay(0.5)` → `UpdateArea(Get Viewport Size)` → `Delay(0.5)` → `RunSelfTest()` → `SelfTestFlightStart()` → `Delay(0.5)` → `SelfTestFlightMid()` → `Delay(2.5)` → `SelfTestFlightEnd()`.

**Check:** compiles clean; no leftover marker spawn or `FireCount` update in `RequestFire`; `H` and `R` handlers exist; the chain matches 7g.

---

### Step 8 — `BP_DDHUD`

Change **line 1 only** to: `"DOME DEFENSE  |  PHASE 2  |  BASES AND ABMS"`. Keep lines 2 to 4 as they are. Do **not** add any ammo text to the HUD (unless you are using the Step 3 fallback).

**Check:** compiles clean.

---

### Step 9 — Save and run

1. **Save All** (save the level before any level load).
2. `StartPIE`, wait **about 8 seconds** (the self-test runs for about 4 s), `GetLogEntries` (filter `[DD]`), `CaptureEditorImage` (take one image at about 2 s and one after the self-test finishes), `StopPIE`.
3. Fill in the report (section 8).

---

## 7. Acceptance tests

### A. Static checks
- [ ] **A1** New assets: `BP_MissileBase`, `BP_ABM`, `MI_DD_Base`, `MI_DD_Abm`. No others.
- [ ] **A2** Every changed Blueprint compiles with 0 errors and 0 warnings.
- [ ] **A3** `DA_DDConfig.BaseLaunchOffsetZ = 70.0`; other values unchanged.
- [ ] **A4** `BP_TapMarker.bAutoExpire` default is `false`.
- [ ] **A5** `BP_MissileBase` has `SetAmmoDisplay`, and `AmmoText` is written only inside it (search the graphs).
- [ ] **A6** `BP_DDPlayerController` has `ConsiderBase`, `SelectBase`, `LaunchAt`, the new `RequestFire`, and key events `H` and `R`; no `Draw Debug` calls; no `1`/`2`/`3` keys.
- [ ] **A7** No actors were added to the level.

### B. Log checks (agent)
Order is not guaranteed. Wait about 8 seconds.

- [ ] **B1** The Phase 0 and Phase 1.1 startup lines still appear (GameMode, Config OK `Width=2250.0 Height=1800.0 ...`, Camera, PlayerController, View target, Crosshair spawned, HUD, PlayfieldDebug, `[DD][Area] Viewport=...`).
- [ ] **B2** `[DD][Core] Bases spawned n=3`
- [ ] **B3** `T1` to `T6`, `T8`, `T9` are PASS (unchanged from Phase 1.1).
- [ ] **B4** `T10 bases spawned: PASS ammo=10`
- [ ] **B5** `T11 base selection: PASS (6 cases)`
- [ ] **B6** `T12 ammo accounting: PASS shown=9`
- [ ] **B7** `T13 no ammo: PASS`, and a `[DD][Input] No ammo X=0 Z=900` line.
- [ ] **B8** `[DD][Input] Fire request base=CENTER X=<int> Z=<int> n=1 ammo=9` (from T7, at the screen center).
- [ ] **B9** `T7 fire path: PASS`
- [ ] **B10** `[DD][Input] Fire request base=LEFT X=1000 Z=1700 n=1 ammo=9` (from the flight test).
- [ ] **B11** `T14 marker and ABM stay in flight: PASS`
- [ ] **B12** `[DD][ABM] Arrived base=LEFT X=1000 Z=1700`
- [ ] **B13** `T15 arrival: PASS flight=<ms> expected=<ms>` with `expected` about 2100 ms.
- [ ] **B14** `[DD][Test] SelfTest done: 15/15 PASS`
- [ ] **B15** No line containing `ERROR`, `Accessed None`, or `Blueprint Runtime Error`.

### C. Visual checks (editor image of the play view; NEEDS-HUMAN if you cannot capture it)
- [ ] **C1** Phase 1.1 visuals unchanged: black background, no bars, yellow and orange frames, brown ground edge to edge with a white top line, 6 green city markers, axes.
- [ ] **C2** **Three steel gray-blue bases** stand on the ground line at X = −1008, 0 and +1008 (same places as the cyan debug boxes).
- [ ] **C3** Each base has a **white `10`** under it, inside the brown strip, readable (not mirrored, not black, not edge-on).
- [ ] **C4** No ABM or marker is on screen at rest. HUD line 1 reads `DOME DEFENSE | PHASE 2 | BASES AND ABMS`. The HUD shows no ammo numbers.

### D. Interactive checks (human, PIE; click inside the viewport first; `Shift+F1` frees the mouse, `Esc` stops PIE)
- [ ] **D1** Click on the left side of the playfield: the **Left** base's number goes from 10 to 9; a blue dot flies from the Left base to the click; the **yellow-orange marker stays at the click until the dot reaches it**, then both vanish.
- [ ] **D2** The Center base's dot is much faster than a side base's dot (about 2.5 times).
- [ ] **D3** Nearest base: a click near the left fires Left, near the right fires Right, near the middle fires Center. On a wide window, a click **outside** an outer base fires that outer base, and its dot flies through the extra space.
- [ ] **D4** Empty one base: click 10 times near the left. Its number reaches `0`. The next click near the left is fired by the **next nearest** base (Center), not Left.
- [ ] **D5** With every base at `0`, a click does nothing: no dot, no marker, and the log shows `No ammo`. Press `R`: all numbers return to `10`.
- [ ] **D6** Click quickly several times: several dots and markers exist at once, and **each marker disappears when its own dot arrives**, not all together.
- [ ] **D7** Resize the window (wide, 5:4, nearly square): bases and numbers stay together under the layout, the numbers stay inside the ground strip and readable, and dots still land on the click.
- [ ] **D8** `H` still toggles the debug overlay; the crosshair still follows the mouse.

### D-DEVICE. Touch (Android device, or desktop with "Use Mouse for Touch"; carried from Phase 1.1, still open)
- [ ] **T-1** A tap launches from the nearest base with a marker at the tap.
- [ ] **T-2** A second finger is ignored.
- [ ] **T-3** One tap makes exactly one launch.
- [ ] **T-4** No on-screen joysticks.

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped.

---

## 8. Report format (the agent's final message)

```
## Phase 2 report
**Engine version:** x.y.z
**MCP tools used:** (list)

### Assets created or changed
(table: path | type | change | compiled clean)

### Acceptance results
A1 PASS/FAIL ... through E1 PASS/FAIL/NEEDS-HUMAN/NEEDS-DEVICE

### Log excerpt
(all [DD] lines from PIE, including every [DD][Test], [DD][ABM] and [DD][Input] line)

### Screenshot notes
(what the bases and numbers looked like; text orientation used)

### Deviations
### Open questions / risks
```

---

## 9. What this hands to Phase 3

| Provided | How Phase 3 uses it |
|---|---|
| `BP_ABM.OnArrived` with the `PHASE 3 HOOK` comment | Spawn the explosion at `Target` there, before `Destroy Actor`. Keep the marker dismissal and statistics. |
| `BP_DDGameMode.ArrivalCount`, `LastArrivalPos`, `LastFlightTime` | Test hooks for explosion timing. |
| `BP_DDGameMode.Bases[0..2]` with `Ammo`, `LaunchPos`, `SetAmmo`, `ConsumeAmmo` | Enemy impacts (Phase 5) will zero a base's ammo through `SetAmmo(0)`. |
| `BP_MissileBase.SetAmmoDisplay(Count)` | The one function to rewrite for the ammo-icon art pass. |
| `RefillBases` | The wave phase calls it between waves. `R` is its debug twin. |
| Config `ExplosionMaxRadius` and the three timing values | Already in config for Phase 3. |
| Self-test chain pattern (sync tests, then a latent flight test) | Phase 3 adds the explosion timing test at the end of the chain. |

---

## 10. Notes for the human (not agent steps)

1. **Ammo numbers are a stand-in.** The art pass replaces them with one icon per missile left by editing only `SetAmmoDisplay`. Ten icons will need to fit the ground strip (for example two rows of five).
2. **Unchanged Android notes:** set orientation to Landscape; set Default Touch Interface to None; the renderer lines in `DefaultEngine.ini` are desktop-only; the HUD's fixed pixel sizes will look small on phones. The ammo numbers are world text, so they scale with the screen.
3. **Touch is still untested.** Run the D-DEVICE checks on a device before closing this phase.
4. **`R` is a desktop debug key.** There is no restock in the game until the wave phase.
5. If the agent used the HUD-projection fallback for the ammo numbers, they will not scale with screen height; revisit it in the art pass.

---

## 11. Common pitfalls

1. **A rocket launches from the wrong base:** `SelectBase` must consider Center first, then Left, then Right, and use strictly-less-than-by-0.5 to replace the current best. Distances are measured from the **launch point**, not the actor location.
2. **The marker vanishes after one second:** `bAutoExpire` is still `true` on the class default (Step 5).
3. **The marker stays forever:** `OnArrived` did not call `Dismiss`, or `bActive` was never set false and the ABM keeps re-arriving.
4. **The ABM overshoots or stops short:** the arrival test must be `Step >= Remaining`, with the snap to `Target` in `OnArrived`.
5. **The ABM speed depends on frame rate:** movement must use `DeltaSeconds`.
6. **T11 fails on the tie case (c4):** tie order is Center, Left, Right, and `ConsiderBase` replaces the best only if `d < SelBestDist − 0.5`.
7. **A self-test case sees the previous case's value:** results were read after the next case ran. Store each in its own variable first.
8. **The ammo number is black or invisible:** the scene has no lights. Use the Step 3 fallback.
9. **The ammo number is mirrored:** try Yaw −90 on `AmmoText`.
10. **Bases appear at the wrong height or inside the ground:** the base actor must spawn at `Z = GroundTopZ` (100), and the body's relative Z is +35.
11. **"Target must have a connection" on `DDLog`:** recreate the node from `find_node_types`.
12. **The flight test (T14) fails because the ABM already arrived:** the Left base must be the only one with ammo and the target must be far (about 2.1 s of flight). Check `SelfTestFlightStart`.
13. **T7 leaves ABMs or markers on screen:** the cleanup at the end of T7 must destroy every `BP_ABM` and dismiss every `BP_TapMarker`.
