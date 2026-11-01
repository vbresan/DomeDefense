# Phase 4 — Enemy Missiles and Automatic Spawner

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Depends on** | Phase 3 (agent checks passed 19/19; the human checks of Phases 2 and 3 and touch are still open and are not repeated here) |
| **Phase goal** | Enemy missiles appear at the top of the screen on their own, one every few seconds, and fly in a straight line at constant speed toward one of nine ground slots (three bases, six cities). An enemy that touches an explosion is destroyed and creates **its own explosion** (chain reactions). An enemy that reaches its slot is logged as an impact and removed. Nothing is damaged yet. |

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **No bars, no aspect caps.** The whole window is the playfield. Do not add any.
3. **Existing logic must survive** unless a step says to change or delete it. Read each Blueprint's current graphs (`read_graph_dsl`) before editing.
4. **Compile after every Blueprint change.** Every Blueprint ends with no errors and no warnings.
5. **Run each step's "Check"** before moving on.
6. **You cannot press keys, click, touch, or resize in PIE.** The self-test checks enemies with **synthetic time steps** (it calls `Advance` by hand) and with two short live segments. Interactive checks (group D) are **NEEDS-HUMAN**; touch is **NEEDS-DEVICE**.
7. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
8. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after each test. **Save before loading a level.**
9. **Read sections 4 and 5 before building graphs.**
10. **This phase adds exactly two config values** (Step 1). Change no other config value.

---

## 2. Scope

### In scope
- Config: `bEnemyAutoSpawn` and `EnemySpawnInterval`.
- Material instance `MI_DD_Enemy`.
- `BP_EnemyMissile`: flight, kill by explosion, impact.
- `BP_DDGameMode`: nine ground slots, spawn, random spawn, clear, automatic spawner, counters.
- `BP_DDPlayerController`: debug keys `E` and `K`, spawner start, self-test extended from 19 to **26** tests.
- `BP_DDHUD`: title line only.

### Out of scope (do NOT create)
- Damage to bases or cities, hit points, destroyed-city visuals, base destruction, game over (Phase 5).
- Waves, speed ramps, scoring, score display, bonus (Phase 6).
- MIRV splitting, bombers, smart bombs, enemy trails, sounds, particles, imported art.
- Any UMG. Any new key besides `E` and `K`. Any new config value besides the two above.

---

## 3. Design (decided; do not change)

### 3.1 The nine ground slots
Enemies aim at one of nine slots, numbered left to right. Positions come from config; **never hard-code X values**. Every slot is at `Y = PlayPlaneY` and `Z = GroundTopZ` (the top of the ground strip).

| Slot | Meaning | X from config | Name for logs |
|---|---|---|---|
| 0 | Left base | `BaseLeftX` | `BASE-LEFT` |
| 1 | City 1 | `CityX[0]` | `CITY1` |
| 2 | City 2 | `CityX[1]` | `CITY2` |
| 3 | City 3 | `CityX[2]` | `CITY3` |
| 4 | Center base | `BaseCenterX` | `BASE-CENTER` |
| 5 | City 4 | `CityX[3]` | `CITY4` |
| 6 | City 5 | `CityX[4]` | `CITY5` |
| 7 | City 6 | `CityX[5]` | `CITY6` |
| 8 | Right base | `BaseRightX` | `BASE-RIGHT` |

Any other index: position `(0, PlayPlaneY, GroundTopZ)` and name `INVALID`.

### 3.2 Spawning
- **Start point:** `X` random between `VisibleMinX` and `VisibleMaxX` (the **whole visible width**, including the extra space beside the outer bases), `Z = VisibleMaxZ` (the top edge of the visible area), `Y = PlayPlaneY`.
- **Target:** one slot chosen at random, uniformly from 0 to 8, at spawn time. (Phase 5 will exclude destroyed slots.)
- **Speed:** `EnemyBaseSpeed` from config (130 units/s; a vertical flight takes about 13 s).
- **Automatic spawner:** when running, it spawns **one random enemy every `EnemySpawnInterval` seconds** (the first one after one interval). It starts only when told to (section 3.6). `bEnemyAutoSpawn = false` turns it off completely.

### 3.3 Flight
Straight line, constant speed, time-based, like the ABM: each step moves `Speed * Dt`; if the step reaches or passes the remaining distance, the enemy **snaps exactly to its target** and impacts. No curve, no acceleration.

### 3.4 Kill and chain reaction
- Every step, **before moving**, the enemy asks every explosion in `GM.ActiveExplosions` `ContainsPoint(its own location)`. If any says yes, the enemy **dies**.
- A dying enemy: counts as killed, **spawns an explosion at its own location** (`GM.SpawnExplosion`, same size and timing as any explosion), and is destroyed. That explosion can kill more enemies (chain reaction).
- **Spawn the explosion only after the loop over `ActiveExplosions` has finished** (spawning adds to that array).
- Explosions do nothing else: ABMs, cities and bases are not affected.

### 3.5 Impact
An enemy that reaches its target: counts as an impact, logs, is destroyed. **No explosion, no damage, no visible effect** in this phase (Phase 5 decides).

### 3.6 Spawner control
- `StartSpawner` starts the loop (only if `bEnemyAutoSpawn` is true and it is not already running). `StopSpawner` stops it.
- **The spawner must not run during the self-test.** The controller starts it at the end of the self-test, or at the beginning if `bSelfTest` is false.
- Debug keys (desktop only): **`E`** spawns one random enemy now; **`K`** removes all enemies **without explosions**. `K` does not stop the spawner.

### 3.7 Visual (placeholder)
A red dot: the engine **Sphere**, relative scale **(0.4, 0.4, 0.4)** (40 units across), at relative `Y = 70`, material `MI_DD_Enemy`. No trail.

### 3.8 Layering (Y offset of visual components)
| Layer | Y offset |
|---|---|
| Ground | 0 |
| Bases (body) and ammo number | +60 |
| Cities (later) | +60 |
| **Enemies** | **+70** |
| ABMs | +80 |
| Explosions | +90 |
| Crosshair | +100 |
| Tap marker | +110 |

### 3.9 Log lines
Category `Enemy`. Only real enemies log; synthetic test enemies are silent (`bLog = false`). The two live test enemies log.
- `[DD][Enemy] Spawn slot=<NAME> fromX=<int> toX=<int>`
- `[DD][Enemy] Killed slot=<NAME> X=<int> Z=<int>` (slot = the slot it was aiming at)
- `[DD][Enemy] Impact slot=<NAME> X=<int> Z=<int>`
- `[DD][Enemy] Spawner started interval=<int>ms`
- `[DD][Enemy] Cleared n=<int>` (from `K`)

---

## 4. Conventions as built (these replace earlier text where they differ)

| Topic | As built |
|---|---|
| Camera | **Ortho Width is the vertical extent** (visible height). Do not change `ApplyArea`. |
| Visible area | `BP_DDGameMode` holds `VisibleMinX/MaxX/MinZ/MaxZ`. Spawn positions use the visible area, **not** the config playfield values. |
| Reading values | A value read in a later node is evaluated **when that node runs**. **Store a value in a variable (or a local) before you change what it depends on.** This also applies to **random nodes**: every read of `Random ... In Range` rolls a new number. |
| Config | Config functions and `GetConfig` are **not Pure**. The game mode uses its cached `Config`; the controller uses its cached `Cfg`. |
| Logging | Only through the static `DDLog(Message, Category, bIsError)`. Coordinates in logs are rounded integers. |
| Ticking | A Blueprint actor needs **Can Ever Tick = true** and **Start With Tick Enabled = true** for its Event Tick to run (found in Phase 3). Set both on `BP_EnemyMissile`. |
| Self-test | `bSelfTest` (default true) starts a chain from `BeginPlay`. `RunSelfTest` runs T1–T6, T8, T9 and calls `RunPhase2Tests`; `RunPhase3Tests` runs T16–T18; `SelfTestFlightStart/Mid/End` run T14–T15; `SelfTestExplosionEnd` runs T19. Sample values inside a test function are **function locals**. |
| Test variable names | Older test variables (`C1`–`C6`, `A1`, `A2`, `T1Text`, `T2Text`, `MBefore`, `ABefore`) keep their names (the tools cannot rename them). **Leave them.** New test variables use the prefix `Test`. |
| Hooks | Comment boxes cannot be created. A "hook" is described by the node **after which** the new call goes. |
| Text of the HUD title | One space on each side of every pipe character. |
| Slow editor | "max tick rate 3" appears when the editor window is not focused. Do not tighten the tolerances given below. |

---

## 5. Tooling notes

1. **Do not guess node IDs.** Use `find_node_types`. Call `get_graph_dsl_docs` once.
2. **Key events cannot be written as DSL.** Use `create_node` + `connect_pins`, **Pressed** pin only. Known ID pattern: `Input|KeyboardEvents|H`. Find the ones for `E` and `K` with `find_node_types`.
3. **Latent nodes (`Delay`) only work in event graphs**, not in functions. `StartSpawner` and `EnemySpawnLoop` are **custom events** in the event graph of `BP_DDGameMode`. A custom event can be called from another Blueprint through a cast reference.
4. **Non-Pure functions need exec wires.** Call each function in sequence, store each result in its own variable or local, then combine the variables.
5. **Sequence pins** are `then_0`, `then_1`. **No nodes after a Branch/return** in the same chain. Where the pseudo-code shows an `If` followed by more steps, either put the `If` inside a small helper function, or connect both branch ends to the **same** next exec pin.
6. **Spawn Actor From Class:** Collision Handling Override = **Always Spawn**. After spawning, call the actor's `Init` in the same exec chain.
7. **`Destroy Actor` runs `EndPlay` immediately.** In same-frame tests do **not** use `IsValid` on a just-destroyed actor; use `Length(GM.ActiveEnemies)` or `Length(GM.ActiveExplosions)`. After at least one frame has passed, `IsValid` is fine.
8. **`Set Actor Tick Enabled(false)`** keeps an actor still. Synthetic test enemies and explosions use it, so only `Advance` moves them. The two **live** test enemies keep ticking.
9. **Loops over an array that can change:** loop over a **copy** (`ClearEnemies`), or collect first and act afterwards (`IsInsideExplosion`).
10. **Properties not settable through ObjectTools:** `bIsPureFunc`, `BlueprintType`, `ActorLabel`. Read every property you set back.
11. **PIE tools:** `StartPIE`, `StopPIE`, `IsPIERunning`, `GetLogEntries`, `CaptureViewport`, `CaptureEditorImage`. Use `CaptureEditorImage` for the play view.

---

## 6. Implementation steps

### Step 1 — Config

Add to `PDA_DDConfig` (Instance Editable, category `Enemy`):

| Variable | Type | Default |
|---|---|---|
| `bEnemyAutoSpawn` | boolean | `true` |
| `EnemySpawnInterval` | float | `2.5` |

Read both back on `DA_DDConfig`; set them on the instance if they do not show the defaults. Also read back `EnemyBaseSpeed` (expected 130) and record it. **Do not change any other value.**

**Check:** `DA_DDConfig.bEnemyAutoSpawn = true`, `EnemySpawnInterval = 2.5`, `EnemyBaseSpeed` recorded; `PDA_DDConfig` compiles clean.

---

### Step 2 — Material

Create Material Instance `MI_DD_Enemy` (parent `M_DD_Unlit`) in `/Game/DomeDefense/Materials/`: `Color = (1.0, 0.2, 0.2, 1.0)`.

**Check:** exists; parent is `M_DD_Unlit`; color reads back.

---

### Step 3 — Enemy and its game-mode support

Do the sub-steps **in this order** (the two Blueprints refer to each other).

#### 3a. Empty shell
Create Blueprint class `BP_EnemyMissile` (parent `Actor`) in `/Game/DomeDefense/Actors/`. Compile. Nothing else yet.

#### 3b. `BP_DDGameMode`: new variables (not Instance Editable)

| Variable | Type | Default |
|---|---|---|
| `ActiveEnemies` | array of `BP_EnemyMissile` reference | empty |
| `EnemiesSpawned` | integer | 0 |
| `EnemiesKilled` | integer | 0 |
| `EnemyImpacts` | integer | 0 |
| `LastImpactPos` | vector | (0,0,0) |
| `LastImpactSlot` | integer | -1 |
| `LastKillPos` | vector | (0,0,0) |
| `bSpawnerRunning` | boolean | false |

Compile.

#### 3c. `BP_EnemyMissile`: full content

**Components:** `Root` (Scene); `Dot` StaticMeshComponent: mesh `/Engine/BasicShapes/Sphere`, relative location (0, **70**, 0), relative scale (0.4, 0.4, 0.4), material `MI_DD_Enemy`, NoCollision (read back the collision-enabled flag, not only the preset).

**Class defaults:** **Can Ever Tick = true** and **Start With Tick Enabled = true**.

**Variables** (not Instance Editable):

| Variable | Type | Default |
|---|---|---|
| `StartPos` | vector | (0,0,0) |
| `Target` | vector | (0,0,0) |
| `Speed` | float | 0 |
| `SlotIndex` | integer | 0 |
| `bActive` | boolean | false |
| `bLogEvents` | boolean | false |
| `GM` | `BP_DDGameMode` reference | none |

**Event BeginPlay:** `GM = Get Game Mode → Cast To BP_DDGameMode`.

**Functions** (all non-Pure):

**`Init`** (inputs `Start` vector, `TargetPos` vector, `FlightSpeed` float, `Slot` integer, `bLog` boolean):
```
StartPos = Start ; Target = TargetPos ; Speed = FlightSpeed ; SlotIndex = Slot ; bLogEvents = bLog
SetActorLocation(Start)
bActive = true
```

**`IsInsideExplosion`** (no inputs; output `bHit` boolean):
```
Loc = GetActorLocation ; Hit = false (local)
For Each E in GM.ActiveExplosions:
    If IsValid(E): r = E.ContainsPoint(Loc) ; If r → Hit = true
return Hit
```
(Do not spawn or destroy anything inside this loop. Store `r` before using it.)

**`Advance`** (input `Dt` float):
```
If NOT bActive → return
Hit = IsInsideExplosion()                      // store in a local first
If Hit → Die() ; return
Loc = GetActorLocation
Remaining = Distance(Loc, Target)
Step = Speed * Dt
If Step >= Remaining → Impact()
Else → Dir = (Target - Loc) / Remaining ; SetActorLocation(Loc + Dir * Step)
```
(The division is safe: the `Else` branch is reached only when `Remaining > Step >= 0`.)

**`Die`** (no inputs):
```
bActive = false
P = GetActorLocation                           // store first
GM.EnemiesKilled = GM.EnemiesKilled + 1
GM.LastKillPos = P
GM.SpawnExplosion(P, bLogEvents)               // the chain-reaction explosion
If bLogEvents:
    DDLog("Killed slot=" + GM.GetSlotName(SlotIndex) + " X=" + Round(P.X) + " Z=" + Round(P.Z), "Enemy", false)
Destroy Actor
```

**`Impact`** (no inputs):
```
bActive = false
SetActorLocation(Target)
GM.EnemyImpacts = GM.EnemyImpacts + 1
GM.LastImpactPos = Target
GM.LastImpactSlot = SlotIndex
If bLogEvents:
    DDLog("Impact slot=" + GM.GetSlotName(SlotIndex) + " X=" + Round(Target.X) + " Z=" + Round(Target.Z), "Enemy", false)
Destroy Actor
```

**Event Tick(DeltaSeconds):** `Advance(DeltaSeconds)`.

**Event EndPlay:** `If IsValid(GM) → GM.ActiveEnemies.RemoveItem(Self)`.

**Check:** compiles clean. Read back both tick flags `true`, `Dot` location Y = 70, scale 0.4, material `MI_DD_Enemy`, collision disabled.

#### 3d. `BP_DDGameMode`: functions and events

**Functions** (non-Pure):

**`GetSlotPos`** (input `Index` integer; output `Pos` vector): the table of section 3.1.
```
X = switch on Index: 0 → Config.BaseLeftX ; 1 → Config.CityX[0] ; 2 → Config.CityX[1] ; 3 → Config.CityX[2] ;
                     4 → Config.BaseCenterX ; 5 → Config.CityX[3] ; 6 → Config.CityX[4] ; 7 → Config.CityX[5] ;
                     8 → Config.BaseRightX ; otherwise 0
return (X, Config.PlayPlaneY, Config.GroundTopZ)
```

**`GetSlotName`** (input `Index` integer; output `Name` string): the names of section 3.1; otherwise `"INVALID"`.

**`SpawnEnemy`** (inputs `StartPos` vector, `TargetPos` vector, `FlightSpeed` float, `Slot` integer, `bLog` boolean; output `Enemy` reference `BP_EnemyMissile`):
```
E = Spawn Actor From Class(BP_EnemyMissile, Location = StartPos, Collision = Always Spawn)
E.Init(StartPos, TargetPos, FlightSpeed, Slot, bLog)
ActiveEnemies.Add(E)
EnemiesSpawned = EnemiesSpawned + 1
If bLog → DDLog("Spawn slot=" + GetSlotName(Slot) + " fromX=" + Round(StartPos.X) + " toX=" + Round(TargetPos.X), "Enemy", false)
return E
```

**`SpawnRandomEnemy`** (input `bLog` boolean; output `Enemy` reference):
```
Slot = Random Integer In Range(0, 8)                                   // store in a variable FIRST
X = Random Float In Range(VisibleMinX, VisibleMaxX)                    // store in a variable FIRST
Start = (X, Config.PlayPlaneY, VisibleMaxZ)
Tg = GetSlotPos(Slot)
return SpawnEnemy(Start, Tg, Config.EnemyBaseSpeed, Slot, bLog)
```

**`ClearEnemies`** (input `bLog` boolean; output `Count` integer):
```
n = Length(ActiveEnemies)
Copy = a copy of ActiveEnemies
For Each E in Copy: If IsValid(E) → Destroy Actor (E)
If bLog → DDLog("Cleared n=" + n, "Enemy", false)
return n
```
(No explosions, no kill or impact counting.)

**`StopSpawner`** (no inputs): `bSpawnerRunning = false`.

**Custom events** (event graph; not functions):

**`StartSpawner`:**
```
If Config.bEnemyAutoSpawn AND NOT bSpawnerRunning:
    bSpawnerRunning = true
    DDLog("Spawner started interval=" + Round(Max(Config.EnemySpawnInterval, 0.2) * 1000) + "ms", "Enemy", false)
    EnemySpawnLoop
```
**`EnemySpawnLoop`:**
```
Delay(Max(Config.EnemySpawnInterval, 0.2))
If bSpawnerRunning:
    SpawnRandomEnemy(true)
    EnemySpawnLoop
```

**Check:** `BP_DDGameMode` and `BP_EnemyMissile` compile clean. `SpawnExplosion` (Phase 3) is unchanged.

---

### Step 4 — `BP_DDPlayerController`

Read the existing graphs first. Keep everything else.

**4a. Debug keys.** With `create_node` + `connect_pins` (Pressed pin only):
- **`E`** → `GM.SpawnRandomEnemy(true)`
- **`K`** → `GM.ClearEnemies(true)`

`H` and `R` stay. No other keys.

**4b. New variables** (not Instance Editable)

| Variable | Type | Default |
|---|---|---|
| `TestEnemyA` | `BP_EnemyMissile` reference | none |
| `TestEnemyB` | `BP_EnemyMissile` reference | none |
| `TestEnemyLaunched` | boolean | false |
| `TestEnemyImpactsBefore` | integer | 0 |
| `TestEnemyKillsBefore` | integer | 0 |
| `TestEnemyExplBefore` | integer | 0 |
| `TestEnemyFinishedBefore` | integer | 0 |
| `TestEnemyActiveBefore` | integer | 0 |
| `TestEnemyActiveExplBefore` | integer | 0 |

**4c. New function `RunPhase4Tests`** (non-Pure, no inputs). Sample values are function locals; store each result before using it. `Y = Cfg.PlayPlaneY`; `V = 130.0` (a literal test speed); `M, G, H` are `Cfg.ExplosionMaxRadius`, `Cfg.ExplosionGrowTime`, `Cfg.ExplosionHoldTime`; `TT = G + H + Cfg.ExplosionShrinkTime`. All position tolerances are 0.5 units.

```
T20 slots
    p0..p8 = GM.GetSlotPos(0..8)                       (nine separate locals)
    expX = [Cfg.BaseLeftX, Cfg.CityX[0], Cfg.CityX[1], Cfg.CityX[2], Cfg.BaseCenterX,
            Cfg.CityX[3], Cfg.CityX[4], Cfg.CityX[5], Cfg.BaseRightX]
    posOK   = for every i in 0..8: |pi.X - expX[i]| <= 0.5 AND |pi.Y - Y| <= 0.5 AND |pi.Z - Cfg.GroundTopZ| <= 0.5
    orderOK = p0.X < p1.X < p2.X < p3.X < p4.X < p5.X < p6.X < p7.X < p8.X
    n0 = GetSlotName(0) ; n1 = GetSlotName(1) ; n3 = GetSlotName(3) ; n4 = GetSlotName(4)
    n5 = GetSlotName(5) ; n7 = GetSlotName(7) ; n8 = GetSlotName(8) ; n9 = GetSlotName(9)
    nameOK  = n0 == "BASE-LEFT" AND n1 == "CITY1" AND n3 == "CITY3" AND n4 == "BASE-CENTER"
              AND n5 == "CITY4" AND n7 == "CITY6" AND n8 == "BASE-RIGHT" AND n9 == "INVALID"
    ok = posOK AND orderOK AND nameOK
    If ok → TestPassCount++
    DDLog("T20 slots: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)

T21 random enemy spawns            (30 spawns; the chance of a false failure is below one in a million)
    nBefore = Length(GM.ActiveEnemies) ; sBefore = GM.EnemiesSpawned
    allOK = true ; minSlot = 99 ; maxSlot = -1 ; minX = 1000000 ; maxX = -1000000
    Repeat 30 times (For Loop 0..29):
        e = GM.SpawnRandomEnemy(false) ; e.SetActorTickEnabled(false)
        slot = e.SlotIndex ; sx = e.StartPos.X
        slotOK   = (slot >= 0 AND slot <= 8)
        startOK  = |e.StartPos.Z - GM.VisibleMaxZ| <= 0.5 AND sx >= GM.VisibleMinX AND sx <= GM.VisibleMaxX
        targetOK = Distance(e.Target, GM.GetSlotPos(slot)) <= 0.5
        speedOK  = |e.Speed - Cfg.EnemyBaseSpeed| <= 0.001
        allOK = allOK AND slotOK AND startOK AND targetOK AND speedOK
        minSlot = Min(minSlot, slot) ; maxSlot = Max(maxSlot, slot) ; minX = Min(minX, sx) ; maxX = Max(maxX, sx)
    cnt = Length(GM.ActiveEnemies) ; spn = GM.EnemiesSpawned
    GM.ClearEnemies(false)
    ok = allOK AND (maxSlot - minSlot >= 3) AND (maxX - minX >= 0.5 * (GM.VisibleMaxX - GM.VisibleMinX))
         AND cnt == nBefore + 30 AND spn == sBefore + 30 AND Length(GM.ActiveEnemies) == 0
    If ok → TestPassCount++
    DDLog("T21 random enemy spawns: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)

T22 enemy flight and impact (stepped)
    iBefore = GM.EnemyImpacts ; kBefore = GM.EnemiesKilled ; xBefore = GM.ExplosionCount ; nBefore = Length(GM.ActiveEnemies)
    St = (-800, Y, 1800) ; Tg = GM.GetSlotPos(8)
    E = GM.SpawnEnemy(St, Tg, V, 8, false) ; E.SetActorTickEnabled(false)
    d0 = Distance(St, Tg) ; dir = (Tg - St) / d0
    E.Advance(2.0) ; p1 = E.GetActorLocation ; exp1 = St + dir * (V * 2.0)
    E.Advance(1.0) ; p2 = E.GetActorLocation ; exp2 = St + dir * (V * 3.0)
    midOK = (Length(GM.ActiveEnemies) == nBefore + 1) AND (GM.EnemyImpacts == iBefore)
    E.Advance(1000.0)                                   // far longer than the remaining flight: impact
    ok = Distance(p1, exp1) <= 0.5 AND Distance(p2, exp2) <= 0.5 AND midOK
         AND GM.EnemyImpacts == iBefore + 1 AND Distance(GM.LastImpactPos, Tg) <= 0.5 AND GM.LastImpactSlot == 8
         AND Length(GM.ActiveEnemies) == nBefore
         AND GM.EnemiesKilled == kBefore AND GM.ExplosionCount == xBefore
    If ok → TestPassCount++
    DDLog("T22 enemy flight and impact: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)

T23 kill, chain reaction and control (stepped)
    a = 0.625 * M ; b = 1.375 * M ; c = 4.375 * M      // 100, 220, 700 for M = 160
    kBefore = GM.EnemiesKilled ; xBefore = GM.ExplosionCount
    nBefore = Length(GM.ActiveEnemies) ; aBefore = Length(GM.ActiveExplosions)
    EX = GM.SpawnExplosion((0, Y, 900), false) ; EX.SetActorTickEnabled(false) ; EX.Advance(G + H * 0.5)    // full radius M at (0,900)
    s4 = GM.GetSlotPos(4)
    E1 = GM.SpawnEnemy((a, Y, 900), s4, V, 4, false) ; E1.SetActorTickEnabled(false)      // inside EX
    E2 = GM.SpawnEnemy((b, Y, 900), s4, V, 4, false) ; E2.SetActorTickEnabled(false)      // outside EX
    E3 = GM.SpawnEnemy((c, Y, 900), s4, V, 4, false) ; E3.SetActorTickEnabled(false)      // far away (control)
    nSpawn = Length(GM.ActiveEnemies)
    E1.Advance(0)                                       // inside EX: dies, spawns EY at E1's position (radius 0)
    k1 = GM.EnemiesKilled ; n1 = Length(GM.ActiveEnemies) ; x1 = GM.ExplosionCount ; pos1 = GM.LastExplosionPos
    EY = GM.LastExplosion ; EY.SetActorTickEnabled(false)
    E2.Advance(0)                                       // EX does not reach it, EY has radius 0: survives
    k2 = GM.EnemiesKilled ; n2 = Length(GM.ActiveEnemies)
    EY.Advance(G + H * 0.5)                             // EY at full radius: reaches E2 (distance b - a = 0.75 M)
    E2.Advance(0)                                       // chain reaction: E2 dies, spawns EZ
    k3 = GM.EnemiesKilled ; n3 = Length(GM.ActiveEnemies) ; x3 = GM.ExplosionCount
    EZ = GM.LastExplosion ; EZ.SetActorTickEnabled(false)
    E3.Advance(0)                                       // far from EX, EY and EZ (radius 0): survives
    k4 = GM.EnemiesKilled ; n4 = Length(GM.ActiveEnemies)
    Destroy Actor (E3) ; Destroy Actor (EX) ; Destroy Actor (EY) ; Destroy Actor (EZ)
    nEnd = Length(GM.ActiveEnemies) ; aEnd = Length(GM.ActiveExplosions)
    ok = nSpawn == nBefore + 3
         AND k1 == kBefore + 1 AND n1 == nBefore + 2 AND x1 == xBefore + 2
         AND Abs(pos1.X - a) <= 0.5 AND Abs(pos1.Z - 900) <= 0.5
         AND k2 == kBefore + 1 AND n2 == nBefore + 2
         AND k3 == kBefore + 2 AND n3 == nBefore + 1 AND x3 == xBefore + 3
         AND k4 == kBefore + 2 AND n4 == nBefore + 1
         AND nEnd == nBefore AND aEnd == aBefore
    If ok → TestPassCount++
    DDLog("T23 kill, chain reaction and control: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)

T24 clear enemies
    nBefore = Length(GM.ActiveEnemies) ; kBefore = GM.EnemiesKilled ; xBefore = GM.ExplosionCount ; iBefore = GM.EnemyImpacts
    Spawn three enemies: GM.SpawnEnemy((0, Y, 1800), GM.GetSlotPos(0 / 1 / 2), V, 0 / 1 / 2, false), each with SetActorTickEnabled(false)
    n = GM.ClearEnemies(false)
    ok = n == nBefore + 3 AND Length(GM.ActiveEnemies) == 0
         AND GM.EnemiesKilled == kBefore AND GM.ExplosionCount == xBefore AND GM.EnemyImpacts == iBefore
    If ok → TestPassCount++
    DDLog("T24 clear enemies: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)
```
(T23's offsets scale with `M`, so it still holds if the explosion radius is tuned. Store every `k`, `n`, `x`, `pos` value in its own local at the point shown, before running the next line.)

**4d. New functions for the live segment** (non-Pure, no inputs)

**`SelfTestEnemyStart`**
```
FireCount = 0 ; LastFirePos = (0,0,0) ; LastFireTime = -1000
TestEnemyImpactsBefore = GM.EnemyImpacts ; TestEnemyKillsBefore = GM.EnemiesKilled
TestEnemyExplBefore = GM.ExplosionCount ; TestEnemyFinishedBefore = GM.ExplosionsFinished
TestEnemyActiveBefore = Length(GM.ActiveEnemies) ; TestEnemyActiveExplBefore = Length(GM.ActiveExplosions)
GM.RefillBases(false)
TestEnemyA = GM.SpawnEnemy((-300, Y, 300), GM.GetSlotPos(3), 130.0, 3, true)    // short flight to CITY3, about 1.6 s, far from every explosion
TestEnemyB = GM.SpawnEnemy((0, Y, 960), GM.GetSlotPos(4), 130.0, 4, true)       // waits in the path of the coming explosion
TestEnemyLaunched = LaunchAt((0, Y, 900))                                        // Center is the nearest base; flight about 0.25 s
```
Both live enemies keep ticking (do **not** disable their ticks).

**`SelfTestEnemyEnd`** (runs 3.5 s after `SelfTestEnemyStart`)
```
aImpact  = (GM.EnemyImpacts == TestEnemyImpactsBefore + 1)
aPos     = Distance(GM.LastImpactPos, GM.GetSlotPos(3)) <= 0.5 AND GM.LastImpactSlot == 3
aGone    = NOT IsValid(TestEnemyA)
ok25 = aImpact AND aPos AND aGone
If ok25 → TestPassCount++
DDLog("T25 enemy live impact: " + (ok25 ? "PASS" : "FAIL"), "Test", NOT ok25)

bKilled  = (GM.EnemiesKilled == TestEnemyKillsBefore + 1)
xOK      = (GM.ExplosionCount == TestEnemyExplBefore + 2) AND (GM.ExplosionsFinished == TestEnemyFinishedBefore + 2)
goneOK   = NOT IsValid(TestEnemyB) AND Length(GM.ActiveEnemies) == TestEnemyActiveBefore
           AND Length(GM.ActiveExplosions) == TestEnemyActiveExplBefore
posOK    = Abs(GM.LastKillPos.X) <= 0.5 AND GM.LastKillPos.Z >= 900 - Cfg.ExplosionMaxRadius AND GM.LastKillPos.Z <= 960.5
abmOK    = TestEnemyLaunched AND NOT IsValid(LastAbm)
ok26 = bKilled AND xOK AND goneOK AND posOK AND abmOK
If ok26 → TestPassCount++
DDLog("T26 enemy killed by explosion: " + (ok26 ? "PASS" : "FAIL") + " killZ=" + Round(GM.LastKillPos.Z), "Test", NOT ok26)

GM.RefillBases(false) ; FireCount = 0 ; LastFirePos = (0,0,0) ; LastFireTime = -1000
DDLog("SelfTest done: " + TestPassCount + "/26 PASS", "Test", TestPassCount != 26)
GM.StartSpawner
```
(Why the numbers work: the Center ABM arrives within its first frame, so its explosion appears around enemy B at once; B is within the explosion's radius within one or two frames and dies, which spawns B's own explosion. Enemy A never gets near any explosion. All explosions have ended well before 3.5 s, even at 3 fps. The explosion count rises by exactly two: the ABM's and B's.)

**4e. Change `SelfTestExplosionEnd`:** keep T19 exactly. **Remove its last two lines** (the refill-and-reset line and the `SelfTest done` log). They moved to `SelfTestEnemyEnd`.

**4f. `Event BeginPlay`: change the self-test chain** to:
`Branch(bSelfTest)`
- **True** → `Delay(0.5)` → `UpdateArea(Get Viewport Size)` → `Delay(0.5)` → `RunSelfTest()` → `RunPhase3Tests()` → `RunPhase4Tests()` → `SelfTestFlightStart()` → `Delay(0.5)` → `SelfTestFlightMid()` → `Delay(2.5)` → `SelfTestFlightEnd()` → `Delay(2.0)` → `SelfTestExplosionEnd()` → `SelfTestEnemyStart()` → `Delay(3.5)` → `SelfTestEnemyEnd()`
- **False** → `GM.StartSpawner`

**Check:** compiles clean; keys `H`, `R`, `E`, `K` exist; `SelfTestExplosionEnd` has no refill and no `SelfTest done` line; the chain matches 4f; the spawner is started in exactly two places (`SelfTestEnemyEnd` and the False branch).

---

### Step 5 — `BP_DDHUD`

Change **line 1 only** to: `"DOME DEFENSE | PHASE 4 | ENEMY MISSILES"`. Keep lines 2 to 4 as they are. Add no enemy text to the HUD.

**Check:** compiles clean.

---

### Step 6 — Save and run

1. **Save All** (save the level before any level load).
2. `StartPIE`. The self-test now runs for about **10 seconds**, then the spawner starts. Wait about **18 seconds**.
3. Take `CaptureEditorImage` about once per second from about 6 s to 9 s (the live enemies and explosions are on screen then), and one at about 14 s and one at about 17 s (an automatic enemy is on screen).
4. `GetLogEntries` (filter `[DD]`), then `StopPIE`.
5. Fill in the report (section 8).

---

## 7. Acceptance tests

### A. Static checks
- [ ] **A1** New assets: `BP_EnemyMissile`, `MI_DD_Enemy`. No others.
- [ ] **A2** Every changed Blueprint compiles with 0 errors and 0 warnings.
- [ ] **A3** `DA_DDConfig`: `bEnemyAutoSpawn = true`, `EnemySpawnInterval = 2.5`; `EnemyBaseSpeed` recorded; no other value changed.
- [ ] **A4** `BP_EnemyMissile` has `Init`, `IsInsideExplosion`, `Advance`, `Die`, `Impact` and the events BeginPlay, Tick, EndPlay. Both tick flags are true. `Dot` relative Y = 70, scale 0.4.
- [ ] **A5** `BP_DDGameMode` has the eight new variables, the functions `GetSlotPos`, `GetSlotName`, `SpawnEnemy`, `SpawnRandomEnemy`, `ClearEnemies`, `StopSpawner`, and the custom events `StartSpawner` and `EnemySpawnLoop`.
- [ ] **A6** `SpawnExplosion` is called only from `BP_ABM.OnArrived`, `BP_EnemyMissile.Die` and the controller's test functions (search the graphs). `SpawnEnemy` and `SpawnRandomEnemy` are called only from the spawner loop, the keys and the test functions.
- [ ] **A7** The controller has key events `H`, `R`, `E`, `K` and no other keys. `SelfTestExplosionEnd` no longer has the refill line or the summary line. `StartSpawner` is called from exactly two places.
- [ ] **A8** No actors were added to the level. No `Draw Debug` calls. `BP_ABM`, `BP_Explosion` and the base logic are unchanged.

### B. Log checks (agent)
Order is not guaranteed. Wait about 18 seconds.

- [ ] **B1** The Phase 0 to 3 startup lines still appear (GameMode, Config OK `Width=2250.0 Height=1800.0 ...`, Camera, PlayerController, View target, Crosshair spawned, HUD, PlayfieldDebug, `[DD][Area] Viewport=...`) and `Bases spawned n=3`.
- [ ] **B2** `T1` to `T6`, `T8` to `T19` are PASS exactly as in Phase 3, including `[DD][Explosion] Start X=1000 Z=1700 max=160` once and `[DD][Explosion] End X=1000 Z=1700 peak=<n> life=<n>ms` once.
- [ ] **B3** `T20 slots: PASS`
- [ ] **B4** `T21 random enemy spawns: PASS`
- [ ] **B5** `T22 enemy flight and impact: PASS`
- [ ] **B6** `T23 kill, chain reaction and control: PASS`
- [ ] **B7** `T24 clear enemies: PASS`
- [ ] **B8** `[DD][Enemy] Spawn slot=CITY3 fromX=-300 toX=-252` and `[DD][Enemy] Spawn slot=BASE-CENTER fromX=0 toX=0`, each once, **before** `SelfTest done`. The synthetic enemies of T21 to T24 log nothing.
- [ ] **B9** `[DD][Input] Fire request base=CENTER X=0 Z=900 n=1 ammo=9` and `[DD][ABM] Arrived base=CENTER X=0 Z=900`.
- [ ] **B10** `[DD][Enemy] Killed slot=BASE-CENTER X=0 Z=<n>` with `n` between 740 and 960, and `[DD][Enemy] Impact slot=CITY3 X=-252 Z=100`, each once.
- [ ] **B11** Before `SelfTest done`, exactly **three** `[DD][Explosion] Start` lines (at X=1000 Z=1700; at X=0 Z=900; and the enemy's own at X=0 Z=`n` from B10) and exactly three `[DD][Explosion] End` lines.
- [ ] **B12** `T25 enemy live impact: PASS` and `T26 enemy killed by explosion: PASS killZ=<n>`
- [ ] **B13** `[DD][Test] SelfTest done: 26/26 PASS`
- [ ] **B14** `[DD][Enemy] Spawner started interval=2500ms` exactly once, **after** `SelfTest done`; then at least one more `[DD][Enemy] Spawn slot=...` line (the first automatic enemy comes one interval later).
- [ ] **B15** No line containing `ERROR`, `Accessed None`, or `Blueprint Runtime Error`.

### C. Visual checks (editor image of the play view; NEEDS-HUMAN if you cannot capture it)
- [ ] **C1** Phase 3 visuals unchanged: black background, no bars, frames, brown ground, green city markers, three gray-blue bases with numbers. After the test the numbers read `10`.
- [ ] **C2** In at least one capture between about 6 s and 9 s, a **red dot** and/or the orange disc near X 0, Z 900 is visible. If no capture caught it, mark **NEEDS-HUMAN**; do **not** claim it.
- [ ] **C3** In a capture at about 14 s or later, at least one **red dot** is on screen between the top edge and the ground. If not caught, **NEEDS-HUMAN**.
- [ ] **C4** HUD line 1 reads `DOME DEFENSE | PHASE 4 | ENEMY MISSILES`.

### D. Interactive checks (human, PIE; click inside the viewport first; `Shift+F1` frees the mouse, `Esc` stops PIE; **wait about 10 seconds after Play so the self-test is finished; do not click or press `E`/`K` before that**)
- [ ] **D1** Red dots start entering from the **top edge** about every 2.5 s, at random places across the whole width, each flying **straight** toward a base or a city on the ground. They are slow (about 13 s for a vertical flight). At the end of its flight a dot simply disappears at its slot; nothing else happens.
- [ ] **D2** Shoot an enemy: when your explosion touches a red dot, the dot **disappears at once** and a new **orange disc** grows where it was.
- [ ] **D3** Chain reaction: when two or more dots are close together, one kill can set off the next.
- [ ] **D4** Layering: red dots are drawn over the ground, bases and cities, and **under** the blue ABM dots and the orange discs.
- [ ] **D5** `E` creates one enemy at once; `K` removes all enemies **without** explosions; after `K` the automatic enemies keep coming.
- [ ] **D6** Resize the window (wide, 5:4, nearly square): enemies start on the top edge across the **whole visible width** in every shape.
- [ ] **D7** Phase 2 and 3 behavior unchanged: base choice, ammo numbers, `R` refill, `H` overlay, crosshair, explosions.
- [ ] **D8** Feel (no pass/fail): note whether the speed and the 2.5 s interval feel right. Tune them in `DA_DDConfig` (`EnemyBaseSpeed`, `EnemySpawnInterval`, `bEnemyAutoSpawn`).

### D-DEVICE. Touch (carried from Phase 1.1; still open)
- [ ] **T-1** A tap launches from the nearest base, and its explosion can kill enemies.
- [ ] **T-2** A second finger is ignored.
- [ ] **T-3** One tap makes exactly one launch.
- [ ] **T-4** No on-screen joysticks. Automatic enemies also appear on the device without keys.

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped.

---

## 8. Report format (the agent's final message)

```
## Phase 4 report
**Engine version:** x.y.z
**MCP tools used:** (list)

### Assets created or changed
(table: path | type | change | compiled clean)

### Config values read
(bEnemyAutoSpawn, EnemySpawnInterval, EnemyBaseSpeed)

### Acceptance results
A1 PASS/FAIL ... through E1 PASS/FAIL/NEEDS-HUMAN/NEEDS-DEVICE

### Log excerpt
(all [DD] lines from PIE, including every [DD][Test], [DD][ABM], [DD][Input], [DD][Explosion] and [DD][Enemy] line)

### Screenshot notes
(what the enemies looked like, when each capture was taken)

### Deviations
### Open questions / risks
```

---

## 9. What this hands to Phase 5

| Provided | How Phase 5 uses it |
|---|---|
| `BP_EnemyMissile.Impact` | The hook for damage: after the counters, call a damage function for `SlotIndex` (destroy a city, zero a base's ammo through `SetAmmo(0)`). |
| `GM.GetSlotPos(Index)`, `GetSlotName(Index)` | The slot numbering: bases 0, 4, 8; cities 1, 2, 3, 5, 6, 7. |
| `GM.SpawnRandomEnemy` | Phase 5 changes the target choice to **surviving slots only**. |
| `GM.StopSpawner` | Game over stops the enemies. |
| `GM.EnemyImpacts`, `LastImpactPos`, `LastImpactSlot` | Test hooks for damage. |
| `BP_EnemyMissile.Die` | Phase 6 adds scoring right after `EnemiesKilled` is incremented. |
| `GM.ActiveEnemies`, `GM.ClearEnemies` | Wave end and restart clear the field. |
| Self-test chain (sync stepped tests, then live segments) | Phase 5 adds damage tests the same way. |

---

## 10. Notes for the human (not agent steps)

1. **Nothing is damaged yet.** Enemies that reach the ground just vanish. The game cannot be lost until Phase 5.
2. **Ammo will run out.** The automatic enemies never stop, and each base holds 10 missiles. Press `R` to refill (debug only) until the wave phase adds a real restock.
3. **The self-test now takes about 10 seconds** at the start of every play session, then enemies begin. Do not click or press `E`/`K` during it. You said it will be dropped later; it must not ship in the Android build.
4. **Tuning** lives in `DA_DDConfig`: `EnemyBaseSpeed`, `EnemySpawnInterval`, `bEnemyAutoSpawn`, and the explosion values from Phase 3. The tests use their own fixed speeds, so tuning does not break them.
5. **Android:** `E` and `K` do not exist on a device; the automatic spawner works without them.
6. **Leftover names:** the old test variables (`C1`–`C6` and friends) still have their short names. The tools cannot rename them; rename by hand in the editor if they bother you, or leave them until the self-test goes.
7. **Touch is still untested.** **Commit** the accepted state when you are happy.

---

## 11. Common pitfalls

1. **Random numbers read twice.** `Random Integer In Range` and `Random Float In Range` roll again on every read. Store the slot and the X in variables first (`SpawnRandomEnemy`).
2. **Spawning inside the loop over `ActiveExplosions`.** `IsInsideExplosion` only collects; `Die` spawns afterwards.
3. **Missing Can Ever Tick.** Without it the enemy never moves in the real game even though the stepped tests pass. The live tests T25 and T26 exist to catch this.
4. **Starting the spawner too early.** An automatic enemy during the self-test breaks the explosion counts. Call `StartSpawner` only from `SelfTestEnemyEnd` and from the `bSelfTest = false` branch.
5. **Test enemies left ticking.** Disable ticks on all synthetic test enemies and explosions right after spawning. Only the two live test enemies tick.
6. **`IsValid` on an actor destroyed in the same frame.** Use `Length(GM.ActiveEnemies)` or `Length(GM.ActiveExplosions)`.
7. **`ClearEnemies` looping over the live array.** Destroying an enemy removes it from `ActiveEnemies`; loop over a copy.
8. **Using `Delay` inside a function.** `StartSpawner` and `EnemySpawnLoop` are custom events in the event graph.
9. **Counting kills for enemies removed by `K` or by the tests.** Only `Die` increments `EnemiesKilled`; only `Impact` increments `EnemyImpacts`.
10. **Moving the live enemy segment before T19.** The extra explosions would break T19's count. Keep the order of 4f.
11. **Hard-coded slot X values.** Always read them from config through `GetSlotPos`.
