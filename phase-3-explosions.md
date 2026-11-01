# Phase 3 — Explosions

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Depends on** | Phase 2 (agent checks passed 15/15; Phase 2's own human checks, its D1–D8, and touch are still open and are not repeated here) |
| **Phase goal** | When an ABM reaches its target, an **explosion** appears there: a flat disc that grows, holds, shrinks and disappears. Several explosions can exist at once. Each explosion knows its current radius and can say whether a point is inside it (Phase 4 uses that to kill enemy missiles). The phase also cleans up the self-test code left by Phase 2. |

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **No bars, no aspect caps.** The whole window is the playfield. Do not add any.
3. **Existing logic must survive** unless a step says to change or delete it. Read each Blueprint's current graphs (`read_graph_dsl`) before editing.
4. **Compile after every Blueprint change.** Every Blueprint ends with no errors and no warnings.
5. **Run each step's "Check"** before moving on.
6. **You cannot press keys, click, touch, or resize in PIE.** The self-test checks explosions with **synthetic time steps** (it calls `Advance` by hand), so it does not depend on the frame rate. One live check lets a real ABM arrive. Interactive checks (group D) are **NEEDS-HUMAN**; touch is **NEEDS-DEVICE**.
7. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
8. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after each test. **Save before loading a level.**
9. **Read sections 4 and 5 before building graphs.**
10. **Do not change any config value.** This phase adds none.

---

## 2. Scope

### In scope
- Housekeeping in `BP_DDPlayerController` (delete dead self-test nodes, rename test variables).
- Material instance `MI_DD_Explosion`.
- `BP_Explosion`: life cycle, radius, point query, clean-up.
- `BP_DDGameMode`: `SpawnExplosion` and explosion bookkeeping.
- `BP_ABM.OnArrived`: spawn the explosion (the Phase 3 hook).
- Self-test extended from 15 to **19** tests.
- `BP_DDHUD`: title line only.

### Out of scope (do NOT create)
- Enemy missiles, killing anything, scoring, chain reactions.
- Damage to bases or cities.
- Sounds, particles, trails, flashing colors, screen shake, imported art (art pass).
- Explosions caused by anything except an ABM arriving.
- Any UMG. Any new config value. Any new key.

---

## 3. Design (decided; do not change)

### 3.1 Life of an explosion
Values come from `DA_DDConfig` (already present): `ExplosionMaxRadius` **M** (160), `ExplosionGrowTime` **G** (0.4 s), `ExplosionHoldTime` **H** (0.4 s), `ExplosionShrinkTime` **S** (0.5 s). Total time **TT = G + H + S** (1.3 s). The radius depends only on the explosion's **age** `t` (seconds since it started):

| Stage | Age `t` | Radius |
|---|---|---|
| Grow | `0 <= t < G` | `M * t / G` |
| Hold | `G <= t < G + H` | `M` |
| Shrink | `G + H <= t < TT` | `M * (1 - (t - G - H) / S)` |
| End | `t >= TT` | the explosion ends and its actor is destroyed |

The profile is linear. Age is the sum of the `DeltaSeconds` values the explosion has been ticked with. **All behavior goes through one function, `Advance(Dt)`**: the real tick calls `Advance(DeltaSeconds)`, and the self-test calls it with chosen steps.

### 3.2 Where and when
- An explosion starts at the **exact arrival point** of an ABM (`Target`, already clamped to the visible area), in the same moment the ABM and its marker are removed.
- It is created only through **`BP_DDGameMode.SpawnExplosion(Pos, bLog)`**. Phase 4 will use the same function for explosions of destroyed enemies.
- Explosions near the ground may overlap the ground strip, the bases and their numbers. That is accepted.

### 3.3 Collision circle
- The **collision circle is the current radius** at every moment (also while growing and shrinking). It is a circle in the play plane: only X and Z count, **Y is ignored**.
- `ContainsPoint(P)` is true only while the explosion is active, its radius is greater than 0, and `(P.X - X)^2 + (P.Z - Z)^2 <= Radius^2`.
- Nothing in this phase reacts to the circle. It is for Phase 4.

### 3.4 Overlap and independence
Any number of explosions can exist together. Each has its own age and radius. Explosions do not affect each other. The game mode keeps the list of active ones in `ActiveExplosions`.

### 3.5 Visual (placeholder, isolated)
- One flat disc: the engine **Sphere** mesh squashed in Y. Orange, unlit, opaque. All explosions use the same color.
- **Only `SetRadius` touches the disc.** The art pass will replace the disc with realistic effects by changing that one function (and the components); the radius logic stays.
- Scale formula: `Sc = Max(Radius, 0.5) / 50` (the Sphere mesh has radius 50 at scale 1); disc relative scale `(Sc, 0.02, Sc)`. The **0.02 in Y is mandatory**: a fat sphere would stick out in front of the crosshair and the tap markers and hide them.

### 3.6 Bookkeeping on the game mode
Counters and last-values used by tests (and by Phase 4): `ActiveExplosions`, `LastExplosion`, `LastExplosionPos`, `ExplosionCount` (spawned), `ExplosionsFinished` (ended by themselves), `LastExplosionPeak`, `LastExplosionLife`. An explosion destroyed from outside (the self-test does this) is **not** counted as finished.

### 3.7 Layering (Y offset of visual components)
| Layer | Y offset |
|---|---|
| Ground | 0 |
| Bases (body) and ammo number | +60 |
| Cities (later) | +60 |
| Enemies (later) | +70 |
| ABMs | +80 |
| **Explosions** | **+90** |
| Crosshair | +100 |
| Tap marker | +110 |

Explosions draw over the ground, bases, cities and ABMs, and under the crosshair and the tap markers.

### 3.8 Log lines
Category `Explosion`. Only real (non-test) explosions log; the self-test explosions are silent.
- Start: `[DD][Explosion] Start X=<int> Z=<int> max=<int>`
- End: `[DD][Explosion] End X=<int> Z=<int> peak=<int> life=<int>ms`

---

## 4. Conventions as built (these replace earlier text where they differ)

| Topic | As built |
|---|---|
| Camera | **Ortho Width is the vertical extent** (visible height). Do not change `ApplyArea`. |
| Placed-actor overrides | A **placed** actor's instance override beats the Blueprint default. This phase spawns everything at runtime. |
| Visible area | `BP_DDGameMode` holds `VisibleMinX/MaxX/MinZ/MaxZ`, computed by `RecomputeVisibleArea`. |
| Reading values | A value read in a later node is evaluated **when that node runs**. **Store a value in a variable (or a local) before you change what it depends on.** |
| Config | Config functions and `GetConfig` are **not Pure**: use execution wires. The game mode uses its cached `Config`; the controller uses its cached `Cfg`. |
| Logging | Only through the static `DDLog(Message, Category, bIsError)`. Coordinates in logs are rounded integers. |
| Self-test | `bSelfTest` (default true) starts a chain from `BeginPlay`. `RunSelfTest` runs T1–T6, T8 and T9 and calls `RunPhase2Tests` (T10–T13 and the new T7). `ReportTest` counts passes in `TestPassCount`. Flight test: `SelfTestFlightStart`, `SelfTestFlightMid`, `SelfTestFlightEnd`. |
| Flight time | `GM.LastFlightTime` is `(now - LaunchTime) + remaining distance / speed`, taken before the final snap, so it stays correct when the editor runs at about 3 fps. Leave it. |
| Hooks | Comment boxes cannot be created. A "hook" is described by the **node after which** the new call goes. |
| Text of the HUD title | One space on each side of every pipe character. |
| Slow editor | "max tick rate 3" appears when the editor window is not focused. Do not tighten the tolerances given below. |

---

## 5. Tooling notes

1. **Do not guess node IDs.** Use `find_node_types`. Call `get_graph_dsl_docs` once.
2. **Latent nodes (`Delay`) only work in event graphs**, not in functions. Set Delay durations **explicitly**.
3. **Non-Pure functions need exec wires.** A Boolean AND cannot wrap calls that need exec wires: call each function in sequence, store each result in its own variable, then combine the variables.
4. **Sequence pins** are `then_0`, `then_1`. **No nodes after a Branch/return** in the same chain. Where the pseudo-code shows an `If` followed by more steps, either put the `If` inside a small helper function, or connect both branch ends to the **same** next exec pin (an exec input accepts several wires).
5. **Spawn Actor From Class:** Collision Handling Override = **Always Spawn**. After spawning, call the actor's `Init...` function in the same exec chain.
6. **`Destroy Actor` runs `EndPlay` immediately.** In same-frame tests do **not** use `IsValid` on a just-destroyed actor; use `Length(GM.ActiveExplosions)` instead. After at least one frame has passed, `IsValid` is fine.
7. **`Set Actor Tick Enabled(false)`** stops an actor from ticking. The self-test uses it on its own explosions so that only `Advance` moves them.
8. **Properties not settable through ObjectTools:** `bIsPureFunc`, `BlueprintType`, `ActorLabel`. Read every property you set back.
9. **Renaming a variable** must update all its references. Use the Blueprint tool's rename operation if one exists (check the tool list). **Never delete and re-create a variable to rename it.**
10. **PIE tools:** `StartPIE`, `StopPIE`, `IsPIERunning`, `GetLogEntries`, `CaptureViewport`, `CaptureEditorImage`. Use `CaptureEditorImage` for the play view.

---

## 6. Implementation steps

### Step 1 — Housekeeping in `BP_DDPlayerController`

**1a. Dead nodes.** Open the `RunSelfTest` graph. It still contains the old Phase 1.1 **T7** nodes, which are not connected to anything. Delete **only** nodes that are (a) part of that old T7 block and (b) not connected to the execution chain. Do not delete anything that is wired.

**1b. Rename these controller variables** (all exist only for the self-test):

| Old | New |
|---|---|
| `C1`, `C2`, `C3`, `C4`, `C5`, `C6` | `TestC1`, `TestC2`, `TestC3`, `TestC4`, `TestC5`, `TestC6` |
| `A1`, `A2` | `TestA1`, `TestA2` |
| `T1Text`, `T2Text` | `TestT1Text`, `TestT2Text` |
| `MBefore`, `ABefore` | `TestMBefore`, `TestABefore` |

Rename only these twelve. Keep their types and defaults.

**Fallback:** if the tool has no rename that updates references, skip 1b, do not touch the variables, and record it under *Deviations*. If dead nodes cannot be deleted, skip 1a and record it.

**Check:** `RunSelfTest` has no unconnected nodes (or the deviation is recorded). No variable with an old name remains (or the deviation is recorded). The controller compiles clean.

---

### Step 2 — Material and config check

**2a.** Create Material Instance `MI_DD_Explosion` (parent `M_DD_Unlit`) in `/Game/DomeDefense/Materials/`: `Color = (1.0, 0.55, 0.10, 1.0)` (orange). It is **not** the yellow of the Phase 0 palette, on purpose: the tap marker is yellow and would vanish against a yellow disc.

**2b.** Read these four values back from `DA_DDConfig` and record them in the report: `ExplosionMaxRadius`, `ExplosionGrowTime`, `ExplosionHoldTime`, `ExplosionShrinkTime` (expected 160, 0.4, 0.4, 0.5). **Do not change them.** If any of them is missing or is 0 or less, stop and record it under *Deviations* (the tests divide by these values).

**Check:** `MI_DD_Explosion` exists, parent is `M_DD_Unlit`, color reads back. Config values recorded.

---

### Step 3 — Explosion and its game-mode support

Do the sub-steps **in this order** (the two Blueprints refer to each other).

#### 3a. Empty shell
Create Blueprint class `BP_Explosion` (parent `Actor`) in `/Game/DomeDefense/Actors/`. Compile. Nothing else yet.

#### 3b. `BP_DDGameMode`: new variables (not Instance Editable)

| Variable | Type | Default |
|---|---|---|
| `ActiveExplosions` | array of `BP_Explosion` reference | empty |
| `LastExplosion` | `BP_Explosion` reference | none |
| `LastExplosionPos` | vector | (0,0,0) |
| `ExplosionCount` | integer | 0 |
| `ExplosionsFinished` | integer | 0 |
| `LastExplosionPeak` | float | 0 |
| `LastExplosionLife` | float | 0 |

Compile.

#### 3c. `BP_Explosion`: full content

**Components:** `Root` (Scene); `Disc` StaticMeshComponent: mesh `/Engine/BasicShapes/Sphere`, relative location (0, **90**, 0), relative scale (0.01, 0.02, 0.01), material `MI_DD_Explosion`, NoCollision (read back the collision-enabled flag, not only the preset).

**Class defaults:** **Start With Tick Enabled = true.**

**Variables** (not Instance Editable):

| Variable | Type | Default |
|---|---|---|
| `MaxRadius`, `GrowTime`, `HoldTime`, `ShrinkTime`, `TotalTime` | float | 0 |
| `Age` | float | 0 |
| `Radius` | float | 0 |
| `PeakRadius` | float | 0 |
| `bActive` | boolean | false |
| `bLogEvents` | boolean | false |
| `GM` | `BP_DDGameMode` reference | none |

**Event BeginPlay:** `GM = Get Game Mode → Cast To BP_DDGameMode`.

**Functions** (all non-Pure):

**`Init`** (inputs `Pos` vector, `MaxR`, `G`, `H`, `S` floats, `bLog` boolean):
```
SetActorLocation(Pos)
MaxRadius = MaxR ; GrowTime = G ; HoldTime = H ; ShrinkTime = S
TotalTime = G + H + S
Age = 0 ; PeakRadius = 0 ; bLogEvents = bLog
SetRadius(0)
bActive = true
```

**`RadiusAtAge`** (input `T` float; output `R` float) — the profile of section 3.1. Exactly this order:
```
TT = Max(T, 0)
If TT >= TotalTime → return 0
If TT < GrowTime → return MaxRadius * (TT / GrowTime)
If TT < GrowTime + HoldTime → return MaxRadius
return MaxRadius * (1 - (TT - GrowTime - HoldTime) / ShrinkTime)
```
(Every division is safe: reaching a line with `/ GrowTime` means `GrowTime > TT >= 0`; reaching the last line means `ShrinkTime > 0`.)

**`SetRadius`** (input `NewRadius` float) — **the only function that touches `Disc`**:
```
Radius = Max(NewRadius, 0)
PeakRadius = Max(PeakRadius, Radius)
Sc = Max(Radius, 0.5) / 50
Disc.SetRelativeScale3D((Sc, 0.02, Sc))
```

**`Advance`** (input `Dt` float):
```
If NOT bActive → return
Age = Age + Dt
If Age >= TotalTime → Finish()
Else → SetRadius(RadiusAtAge(Age))
```

**`Finish`** (no inputs):
```
bActive = false
If IsValid(GM):
    GM.ExplosionsFinished = GM.ExplosionsFinished + 1
    GM.LastExplosionPeak = PeakRadius
    GM.LastExplosionLife = Age
If bLogEvents:
    DDLog("End X=" + Round(GetActorLocation.X) + " Z=" + Round(GetActorLocation.Z)
          + " peak=" + Round(PeakRadius) + " life=" + Round(Age * 1000) + "ms", "Explosion", false)
Destroy Actor
```

**`ContainsPoint`** (input `P` vector; output `bInside` boolean) — section 3.3:
```
Loc = GetActorLocation
dx = P.X - Loc.X ; dz = P.Z - Loc.Z
bInside = bActive AND (Radius > 0) AND ((dx * dx + dz * dz) <= (Radius * Radius))
```

**Event Tick(DeltaSeconds):** `Advance(DeltaSeconds)`.

**Event EndPlay:** `If IsValid(GM) → GM.ActiveExplosions.RemoveItem(Self)`.

**Check:** compiles clean. Read back `Disc` location Y = 90, material `MI_DD_Explosion`, collision disabled.

#### 3d. `BP_DDGameMode`: `SpawnExplosion`

Function (non-Pure), inputs `Pos` vector and `bLog` boolean, output `Explosion` (`BP_Explosion` reference):
```
E = Spawn Actor From Class(BP_Explosion, Location = Pos, Collision = Always Spawn)
E.Init(Pos, Config.ExplosionMaxRadius, Config.ExplosionGrowTime, Config.ExplosionHoldTime, Config.ExplosionShrinkTime, bLog)
ActiveExplosions.Add(E)
LastExplosion = E ; LastExplosionPos = Pos
ExplosionCount = ExplosionCount + 1
If bLog → DDLog("Start X=" + Round(Pos.X) + " Z=" + Round(Pos.Z) + " max=" + Round(Config.ExplosionMaxRadius), "Explosion", false)
return E
```

**Check:** compiles clean; `BP_Explosion` still compiles clean.

---

### Step 4 — `BP_ABM.OnArrived`: the hook

Read the current `OnArrived`. Add one call: **`GM.SpawnExplosion(Target, true)`**, placed **after the `Arrived` log and before `Destroy Actor`**, guarded by `IsValid(GM)`. Keep the statistics, the marker dismissal and the `Arrived` log exactly as they are.

If the graph structure makes that place impossible (no nodes after a Branch), put the call inside the existing `IsValid(GM)` branch, after the statistics lines, and record it under *Deviations*. The order of the `Arrived` and `Start` log lines is **not** checked.

**Check:** compiles clean; the call is reachable only from `OnArrived`.

---

### Step 5 — `BP_DDPlayerController`: tests

Read the existing graphs first. Keep everything else.

**5a. New variables** (not Instance Editable)

| Variable | Type | Default |
|---|---|---|
| `TestExplosion` | `BP_Explosion` reference | none |
| `TestActiveBefore` | integer | 0 |
| `TestLiveCountBefore` | integer | 0 |
| `TestLiveFinishedBefore` | integer | 0 |
| `TestLiveActiveBefore` | integer | 0 |

**5b. New function `RunPhase3Tests`** (non-Pure, no inputs). Store each sample in its own variable or local before using it. `Y = Cfg.PlayPlaneY`; `M, G, H, S` are `Cfg.ExplosionMaxRadius`, `Cfg.ExplosionGrowTime`, `Cfg.ExplosionHoldTime`, `Cfg.ExplosionShrinkTime`; `TT = G + H + S`. All tolerances are 0.5 units.

```
T16 explosion profile
    cBefore = GM.ExplosionCount ; TestActiveBefore = Length(GM.ActiveExplosions)
    E = GM.SpawnExplosion((0, Y, 900), false) ; TestExplosion = E
    E.SetActorTickEnabled(false)
    r1 = E.RadiusAtAge(0)                 expected 0
    r2 = E.RadiusAtAge(G * 0.5)           expected M * 0.5
    r3 = E.RadiusAtAge(G)                 expected M
    r4 = E.RadiusAtAge(G + H * 0.5)       expected M
    r5 = E.RadiusAtAge(G + H)             expected M
    r6 = E.RadiusAtAge(G + H + S * 0.5)   expected M * 0.5
    r7 = E.RadiusAtAge(TT)                expected 0
    r8 = E.RadiusAtAge(TT + 5)            expected 0
    ok = all eight samples within 0.5 of expected
         AND GM.ExplosionCount == cBefore + 1
         AND Length(GM.ActiveExplosions) == TestActiveBefore + 1
    If ok → TestPassCount++
    DDLog("T16 explosion profile: " + (ok ? "PASS" : "FAIL") + " max=" + Round(M), "Test", NOT ok)

T17 point query and finish          (uses TestExplosion, centered at (0, Y, 900))
    E = TestExplosion
    E.Advance(G + H * 0.5)                // now in the hold stage
    rad = E.Radius
    i1 = E.ContainsPoint((M - 1, Y, 900))                 // inside, X axis
    o1 = E.ContainsPoint((M + 1, Y, 900))                 // outside, X axis
    i2 = E.ContainsPoint((0, Y, 900 + M - 1))             // inside, Z axis
    o2 = E.ContainsPoint((0, Y, 900 + M + 1))             // outside, Z axis
    i3 = E.ContainsPoint((0, Y + 500, 900))               // Y is ignored: still inside
    i4 = E.ContainsPoint((0.70 * M, Y, 900 + 0.70 * M))   // diagonal, distance 0.990 M: inside
    o4 = E.ContainsPoint((0.72 * M, Y, 900 + 0.72 * M))   // diagonal, distance 1.018 M: outside (a circle, not a square)
    fBefore = GM.ExplosionsFinished
    E.Advance(TT)                         // pushes the age past the end → Finish
    ok = |rad - M| <= 0.5 AND i1 AND i2 AND i3 AND i4 AND NOT o1 AND NOT o2 AND NOT o4
         AND GM.ExplosionsFinished == fBefore + 1
         AND Abs(GM.LastExplosionPeak - M) <= 0.5
         AND Length(GM.ActiveExplosions) == TestActiveBefore
    If ok → TestPassCount++
    DDLog("T17 point query and finish: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)

T18 overlapping explosions are independent
    EA = GM.SpawnExplosion((-50, Y, 900), false) ; EA.SetActorTickEnabled(false)
    EB = GM.SpawnExplosion(( 50, Y, 900), false) ; EB.SetActorTickEnabled(false)
    EA.Advance(G + H * 0.5)               // A is at full size
    EB.Advance(G * 0.5)                   // B is half size
    ra = EA.Radius ; rb = EB.Radius
    aInB = EB.ContainsPoint((-50, Y, 900))     // B (radius M/2) does not reach A's center: distance 100 > M/2 when M = 160
    bInA = EA.ContainsPoint(( 50, Y, 900))     // A (radius M) reaches B's center: distance 100 <= M when M = 160
    lenMid = Length(GM.ActiveExplosions)
    Destroy Actor (EA) ; Destroy Actor (EB)
    lenEnd = Length(GM.ActiveExplosions)
    ok = |ra - M| <= 0.5 AND |rb - M * 0.5| <= 0.5 AND bInA AND NOT aInB
         AND lenMid == TestActiveBefore + 2 AND lenEnd == TestActiveBefore
    If ok → TestPassCount++
    DDLog("T18 overlapping explosions: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)
```
(T18's two point checks assume the default `M = 160`. If `M` differs, replace the two centers' offset of 50 by `0.3 * M` so that `0.6 * M` lies between `M / 2` and `M`, and record it.)

**5c. Changes to the flight test**

- **`SelfTestFlightStart`:** add these three lines at the **top** (before the refill and the launch):
  ```
  TestLiveCountBefore = GM.ExplosionCount
  TestLiveFinishedBefore = GM.ExplosionsFinished
  TestLiveActiveBefore = Length(GM.ActiveExplosions)
  ```
- **`SelfTestFlightEnd`:** keep T15 exactly. **Remove its last three lines** (`GM.RefillBases(false) ; FireCount = 0 ; ...` and the `SelfTest done` log). They move to the new function below.

**5d. New function `SelfTestExplosionEnd`** (non-Pure, no inputs; runs about 5 s after launch, long after the explosion should have ended even at 3 fps):
```
E = GM.LastExplosion
TT = Cfg.ExplosionGrowTime + Cfg.ExplosionHoldTime + Cfg.ExplosionShrinkTime
spawnedOK  = (GM.ExplosionCount == TestLiveCountBefore + 1)
posOK      = Abs(GM.LastExplosionPos.X - FlightTarget.X) <= 0.5 AND Abs(GM.LastExplosionPos.Z - FlightTarget.Z) <= 0.5
finishedOK = (GM.ExplosionsFinished == TestLiveFinishedBefore + 1)
goneOK     = NOT IsValid(E) AND Length(GM.ActiveExplosions) == TestLiveActiveBefore
peakOK     = GM.LastExplosionPeak >= Cfg.ExplosionMaxRadius * 0.5 AND GM.LastExplosionPeak <= Cfg.ExplosionMaxRadius + 0.5
lifeOK     = GM.LastExplosionLife >= TT - 0.001 AND GM.LastExplosionLife <= TT + 0.8
ok = spawnedOK AND posOK AND finishedOK AND goneOK AND peakOK AND lifeOK
If ok → TestPassCount++
DDLog("T19 explosion lifecycle: " + (ok ? "PASS" : "FAIL") + " peak=" + Round(GM.LastExplosionPeak)
      + " life=" + Round(GM.LastExplosionLife * 1000) + "ms", "Test", NOT ok)
GM.RefillBases(false) ; FireCount = 0 ; LastFirePos = (0,0,0) ; LastFireTime = -1000
DDLog("SelfTest done: " + TestPassCount + "/19 PASS", "Test", TestPassCount != 19)
```
(The peak and life windows are wide on purpose: at 3 fps the explosion gets only a few ticks, and the peak is then below the full radius only if no tick lands in the hold stage; half the radius is always reached.)

**5e. `Event BeginPlay`: change the self-test chain** to:
`Branch(bSelfTest)` True → `Delay(0.5)` → `UpdateArea(Get Viewport Size)` → `Delay(0.5)` → `RunSelfTest()` → `RunPhase3Tests()` → `SelfTestFlightStart()` → `Delay(0.5)` → `SelfTestFlightMid()` → `Delay(2.5)` → `SelfTestFlightEnd()` → `Delay(2.0)` → `SelfTestExplosionEnd()`.

**Check:** compiles clean; `SelfTestFlightEnd` has no refill and no `SelfTest done` line; the chain matches 5e; nothing else in the controller changed.

---

### Step 6 — `BP_DDHUD`

Change **line 1 only** to: `"DOME DEFENSE | PHASE 3 | EXPLOSIONS"`. Keep lines 2 to 4 as they are.

**Check:** compiles clean.

---

### Step 7 — Save and run

1. **Save All** (save the level before any level load).
2. `StartPIE`. The self-test now runs for about **7 seconds**. Wait about **12 seconds**.
3. About 3 seconds after `StartPIE`, take `CaptureEditorImage` about once per second for 3 shots (the live explosion appears at about 3 to 4 s and lives about 1.3 s). Take one more capture after the self-test finishes.
4. `GetLogEntries` (filter `[DD]`), then `StopPIE`.
5. Fill in the report (section 8).

---

## 7. Acceptance tests

### A. Static checks
- [ ] **A1** New assets: `BP_Explosion`, `MI_DD_Explosion`. No others.
- [ ] **A2** Every changed Blueprint compiles with 0 errors and 0 warnings.
- [ ] **A3** The four explosion config values are recorded and unchanged; no other config value changed.
- [ ] **A4** `BP_Explosion` has `Init`, `RadiusAtAge`, `SetRadius`, `Advance`, `Finish`, `ContainsPoint`, and the events BeginPlay, Tick, EndPlay. `Disc` is touched **only** inside `SetRadius` and in the component defaults (search the graphs).
- [ ] **A5** `BP_DDGameMode` has `SpawnExplosion` and the seven new variables. `SpawnExplosion` is called only from `BP_ABM.OnArrived` and from the controller's test functions (search the graphs).
- [ ] **A6** `BP_ABM.OnArrived` still has the statistics, the marker dismissal and the `Arrived` log, and now calls `SpawnExplosion` before `Destroy Actor`.
- [ ] **A7** Step 1 done (or the deviation recorded): no dead nodes in `RunSelfTest`; no variable with an old name.
- [ ] **A8** No actors were added to the level. No `Draw Debug` calls. No new keys.

### B. Log checks (agent)
Order is not guaranteed. Wait about 12 seconds.

- [ ] **B1** The Phase 0, 1.1 and 2 startup lines still appear (GameMode, Config OK `Width=2250.0 Height=1800.0 ...`, Camera, PlayerController, View target, Crosshair spawned, HUD, PlayfieldDebug, `[DD][Area] Viewport=...`) and `Bases spawned n=3`.
- [ ] **B2** `T1` to `T6`, `T8` to `T15` are PASS exactly as in Phase 2, including `Fire request base=CENTER ... ammo=9`, `Fire request base=LEFT X=1000 Z=1700 n=1 ammo=9`, `Arrived base=LEFT X=1000 Z=1700`, and `T15 arrival: PASS`.
- [ ] **B3** `T16 explosion profile: PASS max=160` (or the config value).
- [ ] **B4** `T17 point query and finish: PASS`
- [ ] **B5** `T18 overlapping explosions: PASS`
- [ ] **B6** Exactly **one** `[DD][Explosion] Start X=1000 Z=1700 max=160` line in the whole play window.
- [ ] **B7** Exactly **one** `[DD][Explosion] End X=1000 Z=1700 peak=<n> life=<n>ms` line, with `peak` between 80 and 160 and `life` between 1300 and 2100 (for the default config; in general `TT*1000` up to `TT*1000 + 800`).
- [ ] **B8** `T19 explosion lifecycle: PASS peak=<n> life=<n>ms`
- [ ] **B9** `[DD][Test] SelfTest done: 19/19 PASS`
- [ ] **B10** No line containing `ERROR`, `Accessed None`, or `Blueprint Runtime Error`.

### C. Visual checks (editor image of the play view; NEEDS-HUMAN if you cannot capture it)
- [ ] **C1** Phase 2 visuals unchanged: black background, no bars, yellow and orange frames, brown ground edge to edge, green city markers, three gray-blue bases with numbers. At rest after the test the numbers read `10`, and no explosion, ABM or marker is on screen.
- [ ] **C2** In at least one capture, an **orange disc** is visible at the top right (the test target at X 1000, Z 1700), round, with no other shape left behind. If no capture caught it, mark **NEEDS-HUMAN**; do **not** claim it.
- [ ] **C3** HUD line 1 reads `DOME DEFENSE | PHASE 3 | EXPLOSIONS`.

### D. Interactive checks (human, PIE; click inside the viewport first; `Shift+F1` frees the mouse, `Esc` stops PIE; **wait about 7 seconds after Play so the self-test is finished**)
- [ ] **D1** Click in the playfield. The marker stays at the click until the ABM arrives. At the moment of arrival the marker disappears and an **orange disc** appears at exactly the click point. It grows for about 0.4 s, stays full for about 0.4 s, shrinks for about 0.5 s, then vanishes. Nothing is left behind.
- [ ] **D2** At full size the disc's **diameter is about 18% of the screen height** (radius 160 of a visible height of 1800, which holds for windows at least 5:4 wide; narrower windows show a taller area, so the disc looks relatively smaller). It is **round** in every window shape (wide, 5:4, nearly square).
- [ ] **D3** Click rapidly in nearby spots: several discs exist at the same time, overlap, and each one grows and shrinks on its own schedule. No flicker where they overlap.
- [ ] **D4** Click low, near the ground: the disc may cover the ground strip, the bases and their numbers (accepted). The disc is drawn **over** the ground, bases and ABM dots, and **under** the crosshair and the yellow tap markers.
- [ ] **D5** Base selection, ammo numbers, `R` refill and `H` overlay still work as in Phase 2. The crosshair still follows the mouse.

### D-DEVICE. Touch (carried from Phase 1.1; still open)
- [ ] **T-1** A tap launches from the nearest base, and its explosion appears at the tap point.
- [ ] **T-2** A second finger is ignored.
- [ ] **T-3** One tap makes exactly one launch and one explosion.
- [ ] **T-4** No on-screen joysticks.

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped.

---

## 8. Report format (the agent's final message)

```
## Phase 3 report
**Engine version:** x.y.z
**MCP tools used:** (list)

### Assets created or changed
(table: path | type | change | compiled clean)

### Config values read
(ExplosionMaxRadius, ExplosionGrowTime, ExplosionHoldTime, ExplosionShrinkTime)

### Acceptance results
A1 PASS/FAIL ... through E1 PASS/FAIL/NEEDS-HUMAN/NEEDS-DEVICE

### Log excerpt
(all [DD] lines from PIE, including every [DD][Test], [DD][ABM], [DD][Input] and [DD][Explosion] line)

### Screenshot notes
(what the disc looked like, when each capture was taken)

### Deviations
### Open questions / risks
```

---

## 9. What this hands to Phase 4

| Provided | How Phase 4 uses it |
|---|---|
| `BP_DDGameMode.ActiveExplosions` | The list an enemy checks each frame. |
| `BP_Explosion.ContainsPoint(P)` | The kill test (circle in X and Z). Also `Radius`, `PeakRadius`, `Age`. |
| `BP_DDGameMode.SpawnExplosion(Pos, bLog)` | A destroyed enemy creates its own explosion with the same function (chain reactions). |
| `ExplosionCount`, `ExplosionsFinished`, `LastExplosion*` | Test hooks. |
| `BP_Explosion.SetRadius` | The single place to replace the disc with real effects in the art pass. |
| Self-test chain (sync deterministic tests, then latent live tests) | Phase 4 adds enemy tests the same way. |

---

## 10. Notes for the human (not agent steps)

1. **Look and tuning are placeholders.** Size and timing live in `DA_DDConfig` (`ExplosionMaxRadius`, `ExplosionGrowTime`, `ExplosionHoldTime`, `ExplosionShrinkTime`). Change them there; the tests read the same values.
2. **The self-test now takes about 7 seconds** at the start of every play session and refills the bases at its end. Do not click during that time. You said it will be dropped later; it must not ship in the Android build.
3. **The art pass:** replace the disc inside `SetRadius` (and its components). Keep `Radius`, `ContainsPoint`, `Advance` and the life profile, because Phase 4 depends on them.
4. **Orange, not yellow:** the explosion color differs from the Phase 0 palette so that yellow tap markers stay visible over it.
5. **Touch is still untested.** Run the D-DEVICE checks on a device (or with "Use Mouse for Touch") before closing this phase.
6. **Commit** the accepted state (Phase 2 plus these changes) to the repository when you are happy.

---

## 11. Common pitfalls

1. A fat sphere. The disc's Y scale must stay `0.02`, or it hides the crosshair and the tap markers.
2. Scaling the **actor** instead of the `Disc` component. Only `SetRadius` scales, and only the disc.
3. Test explosions left ticking. Call `SetActorTickEnabled(false)` on them right after spawning, so only `Advance` moves them.
4. Using `IsValid` on an actor in the same frame you destroyed it. Use `Length(GM.ActiveExplosions)`.
5. Reading `GM.ExplosionsFinished` (or any counter) **after** something that changes it. Store the value first.
6. Test explosions that log. They must pass `bLog = false`, or B6 and B7 (exactly one `Start` and one `End`) fail.
7. Calling `Delay` inside a function. Latent waits live in the event graph chain.
8. Changing the live test's tolerances. They are loose because the editor runs at about 3 fps when it is not focused.
9. Counting an explosion that the self-test destroys by hand as "finished". Only `Finish` increments `ExplosionsFinished`.
