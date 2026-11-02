# Phase 8b — Flyers (Bombers and Killer Satellites)

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Depends on** | Phase 8a (accepted: agent run and a human play-test of two games, 16 waves, no errors) |
| **Phase goal** | From wave 3, **bombers** cross the screen horizontally; from wave 5, faster **killer satellites** join them. A flyer enters from a random side at a random height (55% to 75% of the screen height), launches **ordinary enemy missiles** at standing targets while it is on screen, and leaves on the far side. A player-made explosion destroys it: **100 points times the wave multiplier**, and it leaves a chain-reaction explosion like a missile does. Flyers are scheduled by per-wave tables in `DA_DDConfig` (same sequential-sampling idea as the 8a MIRVs), are capped at 2 on screen, and are **not** part of the wave's missile count. **A wave does not end while a flyer is alive.** Debug keys `F` (bomber) and `G` (satellite) spawn one at once. Placeholder shapes only. This is the second of three Phase 8 parts (8c smart bombs). |

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **No bars, no aspect caps.** The whole window is the playfield.
3. **Existing logic must survive** unless a step says to change it. Read each graph (`read_graph_dsl`) before editing it.
4. **Compile after every Blueprint change.** Every Blueprint ends with no errors and no warnings.
5. **Run each step's "Check"** before moving on. **Save All after each step's Check passes.**
6. **You cannot press keys, click, touch, or resize in PIE, and you cannot start a game** (a game starts with a tap on the title). You therefore **cannot exercise** the flyer code. What you can verify: compiles, static inspection of every graph by `read_graph_dsl`, the config values, the assets' properties, and one clean title-screen run. Interactive checks (group D) are **NEEDS-HUMAN**; device checks are **NEEDS-DEVICE**. Never mark a D or T item PASS.
7. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
8. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after the run. **Save before loading a level.** **Do not touch Project Settings, input settings or any `Config/*.ini` file.**
9. **This phase adds exactly 13 config values** (Step 1), **three assets** (`BP_Flyer`, `MI_DD_Bomber`, `MI_DD_Satellite`) and no other asset or config change.
10. **NEVER REWRITE AN EXISTING FUNCTION OR EVENT GRAPH** with `write_graph_dsl` (it deletes all nodes and rebuilds; on big graphs this ran the editor out of memory in Phase 5). A **brand-new** function may be written once, in **one small write**; aim for **about 50 nodes at most** and, if a function would be bigger, split it into helpers. **Existing graphs are changed only by small node edits** (`create_node`, `connect_pins`, break a link, delete one node). The key events `F` and `G` are made with `create_node` and `connect_pins`.
11. **READ EVERY NEW FUNCTION BACK right after writing it** (`read_graph_dsl`). In Phase 7 three writes reported success and read back empty. If a body reads back empty or incomplete: write it **once more**; if it is still wrong, **stop and report** under *Tool problems*. Do not continue on top of an empty function.
12. **Check every call you build for a boolean or number pin left at its default** (the cause of the Phase 7 pause defect). After building a call, read the node's input pins back. In this phase pay special attention to: `Set Visibility` (both pins), `ClearFlyers(bLog)`, `SpawnFlyer(Kind, bLog)` and the integer `Kind` literals of the keys `F` (0) and `G` (1).
13. **After any tool error, do not retry the same large write.** Stop that item, try one smaller piece at most, and if it still fails **stop and report**. **If the editor stops responding or restarts, stop and report.** Do not continue with a half-edited graph.
14. **Order is part of the spec:** callees before callers (Steps 1 to 9).
15. **FROZEN: do not edit** `ApplyPause`, `GameOver`, `CheckGameOver`, `FinishGame`, `EndWave`, `EndWaveNoTargets`, `StepTally`, `RebuildCities`, `SpawnRandomEnemy`, `SpawnEnemy`, `DamageSlot`, `OnEnemyKilled`, `AddScore`, `SpawnExplosion`, the 8a functions (`ArmMirv`, `Split`, `CheckSplit`, `GetWaveMirvCount`, `MakeMirv`, `SpawnMirvChildren`, `MaybeMakeMirv`, `SpawnMirvDebug`), `BP_EnemyMissile` (all of it), `BP_Explosion`, `BP_ABM`, the flow functions (`EnterTitle`, `StartNewGame`, `ClearField`, `PauseGame`, `ResumeGame`, `QuitToTitle`), the HUD. **The only edits to existing functions are the four listed in Step 7** (`StartWave`, `SpawnDue`, `CheckWaveEnd`, `ClearEnemies`).
16. **Do not clean up anything this document does not list** (`StopSpawner`, `bDirectorSilent` and the places that test it, the two disconnected spawn chains in `SpawnRandomEnemy`, HUD lines 2 to 4, the overlay `BP_PlayfieldDebug`).

---

## 2. Scope

### In scope
- `PDA_DDConfig` and the instance `DA_DDConfig`: 13 new values.
- Two material instances: `MI_DD_Bomber`, `MI_DD_Satellite`.
- New Blueprint `BP_Flyer`.
- `BP_DDGameMode`: new variables `ActiveFlyers`, `BomberToSpawn`, `SatelliteToSpawn`; new functions `GetFlyerName`, `GetFlyerSpeed`, `GetFlyerShotsMax`, `GetWaveBomberCount`, `GetWaveSatelliteCount`, `FlyerShoot`, `OnFlyerKilled`, `ClearFlyers`, `SpawnFlyer`, `MaybeSpawnFlyer`; four small edits (Step 7).
- `BP_DDPlayerController`: new key events `F` and `G`.

### Out of scope (do NOT do)
- Real art, trails, sounds, particles, score pop-ups, any HUD change (no flyer counter, no marker).
- Smart bombs and UFOs (Phase 8c). Flyers that split, dodge, drop several missiles at once, or fly at an angle.
- Any change to waves, wave speeds, missile scoring, damage, the tally, bonus cities, targeting in `SpawnRandomEnemy`, the flow screens, or the MIRV logic.
- Any other key. The key set after this phase is exactly: `H`, `R`, `E`, `K`, `X`, `V`, `N`, `P`, `Delete`, `B`, `M`, **`F`**, **`G`** (new) and the Android Back key.

---

## 3. Design (decided; do not change)

### 3.1 What a flyer is
An actor `BP_Flyer` with a **type**: `0` = bomber, `1` = killer satellite. It flies in a straight horizontal line at a constant height and constant speed, **time-based** (`Speed * Dt`), so it freezes with the pause. It is not an enemy missile and is **not** in `ActiveEnemies`; the game mode keeps it in `ActiveFlyers`.

| | Bomber | Satellite |
|---|---|---|
| Speed (config) | `BomberSpeed` (280 units/s) | `SatelliteSpeed` (380 units/s) |
| Shots per crossing | random 1 to `BomberShots` (2) | random 1 to `SatelliteShots` (3) |
| Placeholder look | orange flat rectangle, 100 × 30 units | cyan diamond, about 70 × 70 units |
| Score (config) | `ScoreBomber` (100) | `ScoreSatellite` (100) |

Speed is floored at 50 so that a bad config value can never leave a flyer on screen forever.

### 3.2 Flight path
- **Side:** random, left or right. Left start: `X = VisibleMinX - 120`, direction `+1`. Right start: `X = VisibleMaxX + 120`, direction `-1`. (`120` is the **edge margin**.)
- **Height:** `Z = GroundTopZ + h * (VisibleMaxZ - GroundTopZ)`, with `h` a random float between `Min` and `Max` of `FlyerHeightMin` and `FlyerHeightMax` (defaults 0.55 and 0.75; the order does not matter). `Y = PlayPlaneY`.
- **Span:** `VisibleSpan = VisibleMaxX - VisibleMinX` at spawn time. The flyer travels `PathLength = VisibleSpan + 2 * 120` and leaves: the moment its travelled distance `d = (X - StartX) * Dir` reaches `PathLength` it is removed (`Exit`, no score, no penalty).
- A flyer is on screen while `120 <= d <= 120 + VisibleSpan`.

### 3.3 Shots
- At spawn the flyer gets `ShotCount` = random integer from 1 to `Max(shots config, 1)`.
- **Stratified timing:** the on-screen part is split into `ShotCount` equal segments. Shot `i` (0-based) fires when `d` reaches `120 + VisibleSpan * (i + r_i) / ShotCount`, with `r_i` a random float in `[0.15, 0.85]` drawn when that shot is scheduled. So shots are in order, spread out, and always on screen.
- **A shot** launches **one ordinary enemy missile** from the flyer's position at a **random standing slot** (any of the nine slots that `IsSlotAlive`), through the existing `SpawnEnemy`. Speed = `GetEnemySpeed(Wave) * Max(FlyerMissileSpeedFactor, 0.1)`. If no slot stands, nothing is launched (the log shows `slot=INVALID`).
- Flyer missiles are ordinary: they bypass `WaveMaxActive` and do **not** use `WaveToSpawn`; they are never MIRVs; they score `ScoreEnemyMissile * Multiplier` under the kill-origin rule; each logs its own `[DD][Enemy] Spawn`, `Killed` or `Impact`.

### 3.4 Kill, score and chain reaction
- Every tick, **before moving**, the flyer asks every explosion in `GM.ActiveExplosions` `ContainsPoint(its own location)` (the centre point, the same test as a missile). If any says yes, it **dies**. `bPlayerKill` is true if at least one containing explosion has `bPlayerMade` (the same rule as 6.1).
- A dying flyer: **scores** only if `bPlayerKill` (`OnFlyerKilled`: `base * Multiplier`, `base` = `ScoreBomber` or `ScoreSatellite`; nothing while `Wave < 1`); **spawns an explosion at its position** whose `bPlayerMade` = `bPlayerKill` (so a chain started by your ABM keeps scoring); is removed. Spawn the explosion only after the loop over `ActiveExplosions` has finished (the same as `Die` in `BP_EnemyMissile`).
- Flyer kills do **not** change `EnemiesKilled` or any missile counter.

### 3.5 Scheduling (called from `SpawnDue`)
- At `StartWave`: `BomberToSpawn = GetWaveBomberCount(Wave)`, `SatelliteToSpawn = GetWaveSatelliteCount(Wave)`.
- `GetWaveBomberCount(w)`: 0 if `w < BomberStartWave`; 0 if the table is empty; otherwise `Clamp(WaveBomberCounts[(w - 1) mod Length], 0, GetWaveCount(w))` (the table repeats like `WaveCounts`; so does `GetWaveSatelliteCount` with `SatelliteStartWave` and `WaveSatelliteCounts`).
- **Sequential sampling, at most one flyer per spawned missile.** Each time the director spawns a missile, after the 8a MIRV roll and **before** the decrement of `WaveToSpawn`: `rem = BomberToSpawn + SatelliteToSpawn`; with probability `rem / WaveToSpawn` a flyer is due; its type is a bomber with probability `BomberToSpawn / rem`. This places the table's flyers at random moments of the wave, exactly, and always before the last missile of the wave.
- **Cap:** if `Length(ActiveFlyers) >= Max(FlyerMaxActive, 1)` nothing happens and `ToSpawn` stays, so the next missile spawn tries again. If the wave runs out of missile spawns first, that flyer is simply skipped.
- Missiles from the keys `E` and `M` and the flyers' own missiles never trigger flyer scheduling.

### 3.6 Wave end
A wave ends only when `WaveToSpawn == 0`, **no enemy missile** and **no flyer** is alive. (`CheckWaveEnd` gets one more test, Step 7.) A flyer that has already entered can delay the end by up to its crossing time (about 10 to 12 seconds on a wide window).

### 3.7 Clean-up
`ClearEnemies(bLog)` (used by game over, the no-targets valve, `N`, `K`, `ClearField` and therefore title and new game) now **also removes every flyer**, silently, with no explosion and no score. Its returned count stays the number of **missiles** removed.

### 3.8 Debug keys (desktop only)
`F` → `GM.SpawnFlyer(0, true)` (a bomber); `G` → `GM.SpawnFlyer(1, true)` (a satellite). They ignore the cap and the tables. `SpawnFlyer` spawns nothing while `bGameOver`. They work on the title screen and in a game, like `E`. `K` removes flyers too (3.7).

### 3.9 Config
Instance Editable on `PDA_DDConfig`.

| Category | Variable | Type | Default |
|---|---|---|---|
| `Flyers` | `BomberStartWave` | integer | `3` |
| `Flyers` | `SatelliteStartWave` | integer | `5` |
| `Flyers` | `WaveBomberCounts` | array of integer | `[0, 0, 1, 1, 1, 1, 2, 1, 2, 2, 2, 2]` |
| `Flyers` | `WaveSatelliteCounts` | array of integer | `[0, 0, 0, 0, 1, 1, 1, 1, 1, 2, 1, 2]` |
| `Flyers` | `FlyerMaxActive` | integer | `2` |
| `Flyers` | `BomberSpeed` | float | `280.0` |
| `Flyers` | `SatelliteSpeed` | float | `380.0` |
| `Flyers` | `FlyerHeightMin` | float | `0.55` |
| `Flyers` | `FlyerHeightMax` | float | `0.75` |
| `Flyers` | `BomberShots` | integer | `2` |
| `Flyers` | `SatelliteShots` | integer | `3` |
| `Flyers` | `FlyerMissileSpeedFactor` | float | `1.0` |
| `Scoring` | `ScoreSatellite` | integer | `100` |

(`ScoreBomber` = 100 and `ScoreSmartBomb` = 125 already exist from Phase 0 in category `Scoring`; do not change them.)

### 3.10 Layering
Both flyer bodies sit at relative `Y = 70` (the enemy layer). Nothing else changes in the layer table.

### 3.11 Log lines
| Category | Line |
|---|---|
| `Flyer` | `[DD][Flyer] Spawn type=<Bomber\|Satellite> fromX=<int> Z=<int> dir=<1\|-1> shots=<k> speed=<int>` (from `SpawnFlyer`, only when its `bLog` is true) |
| `Flyer` | `[DD][Flyer] Shot type=<..> slot=<NAME\|INVALID> X=<int> Z=<int>` (after the missile's own `[DD][Enemy] Spawn` line) |
| `Flyer` | `[DD][Flyer] Killed type=<..> X=<int> Z=<int>` |
| `Flyer` | `[DD][Flyer] Exit type=<..> X=<int>` |
| `Flyer` | `[DD][Flyer] Cleared n=<k>` (from `ClearFlyers`, only when its `bLog` is true and `k > 0`) |
| `Score` | `[DD][Score] FlyerKill type=<..> +<pts> score=<n>` (from `OnFlyerKilled`, when its `bLog` is true) |

Natural spawns pass `NOT bDirectorSilent` as `bLog`; the keys pass `true`.
**Order of a kill:** `[DD][Score] FlyerKill`, `[DD][Explosion] Start`, `[DD][Flyer] Killed` (the same pattern as a missile).
**Accounting from now on**, per wave: the number of `[DD][Enemy] Spawn` lines = `WaveTotal` + the sum of the `Split n=` values + the number of `[DD][Flyer] Shot` lines whose `slot` is not `INVALID` (+ one per press of `E` or `M`); and kills + impacts = the number of `Spawn` lines.

---

## 4. What this phase changes in earlier specs

| Earlier text | Now |
|---|---|
| Phase 6 `CheckWaveEnd`: wave ends when `WaveToSpawn == 0` and no enemy | And no flyer (Step 7). |
| 8a accounting: spawns = count + warheads | Also + flyer shots (3.11). |
| Phase 4 `ClearEnemies` (and everything that calls it) | Also removes flyers (3.7). |
| Phase 6 `StartWave`, `SpawnDue` | One small addition each (Step 7). |

---

## 5. Tooling notes

1. **Do not guess node IDs.** Use `find_node_types`. Key event ID pattern: `Input|KeyboardEvents|F`.
2. **A value read by a later node is evaluated when that node runs.** This includes **random nodes** (each read rolls again) and **loop bounds**. Store every random read in a local or variable **before** using it, and store a loop bound in a local before the loop.
3. **Non-Pure functions need exec wires.** Call each in sequence and store each result in its own local. A non-Pure node's output may be wired directly into the very next node of the same exec chain.
4. **No nodes after a Branch or a Return** in the same chain. Where steps follow an `If`, connect **both branch ends to the same next exec pin**. A void function cannot create a Return node; a path ending on an unconnected Then pin is fine.
5. **Seen in 8a (saves retries):** the node `RandomFloat` has no inputs, use `RandomFloatinRange` and `RandomIntegerinRange`; `GetSlotPos`'s input pin is named `Index`; `Max`, `Min`, `Clamp`, the integer remainder (`Percent`), the array `Get` and `RemoveIndex` have type IDs with parentheses that the graph writer cannot parse: build them with `create_node` and `connect_pins`, or use a Select as 8a did. The break-vector type ID is `Math|Vector|BreakVector`. Every such node is read back after placing.
6. **Local arrays:** a function-local integer array is fine (`Alive` in `SpawnMirvChildren` worked in 8a). **Fallback:** a game-mode variable `AliveScratch` (array of integer, not Instance Editable), cleared at the start of the function. Record it.
7. **Float and integer:** convert integers to float before dividing. `Percent` is the integer remainder node.
8. **A new function cannot return an object** (tool limit): the helpers below return integers, strings and floats only; objects are passed in or kept in variables.
9. **Spawn Actor From Class:** Collision Handling Override = **Always Spawn**; call the actor's `Init` in the same exec chain right after spawning. `BeginPlay` has run by then, so the cached `GM` is set.
10. **`Destroy Actor` runs `EndPlay` immediately.** Loops over an array that can change (`ClearFlyers`) loop over a **copy**.
11. **Components set in the Blueprint defaults:** read back mesh, relative location, relative scale, relative rotation, material and the **collision-enabled flag** (not only the preset). Properties not settable through ObjectTools: `bIsPureFunc`, `BlueprintType`, `ActorLabel`; read every property you set back.
12. **Circular reference:** `BP_Flyer` and `BP_DDGameMode` refer to each other. Create the empty `BP_Flyer` shell first (Step 3), exactly as Phase 4 did for `BP_EnemyMissile`.
13. **PIE tools:** `StartPIE`, `StopPIE`, `IsPIERunning`, `GetLogEntries`, `CaptureViewport`, `CaptureEditorImage`.

---

## 6. Implementation steps

### Step 1 — Config

Add the 13 variables of 3.9 to `PDA_DDConfig` (Instance Editable, with the categories shown). Set the values on `DA_DDConfig` and **read all 13 back** (both arrays show all twelve numbers). Also read back `ScoreBomber` (100), `MirvStartWave` (2) and `EnemyBaseSpeed` (130) as unchanged samples. **Do not change any other value.**

**Check:** 13 values read back; `PDA_DDConfig` compiles clean. **Save All.**

---

### Step 2 — Materials

Create two Material Instances in `/Game/DomeDefense/Materials/`, parent `M_DD_Unlit` (as `MI_DD_Enemy` was made in Phase 4):

| Asset | `Color` |
|---|---|
| `MI_DD_Bomber` | `(1.0, 0.65, 0.1, 1.0)` |
| `MI_DD_Satellite` | `(0.2, 0.9, 1.0, 1.0)` |

**Check:** both exist, parent is `M_DD_Unlit`, the colors read back. **Save All.**

---

### Step 3 — Empty shell and game-mode variables

**3a.** Create Blueprint class `BP_Flyer` (parent `Actor`) in `/Game/DomeDefense/Actors/`. Compile. Nothing else yet.

**3b.** `BP_DDGameMode`: add variables (not Instance Editable):

| Variable | Type | Default |
|---|---|---|
| `ActiveFlyers` | array of `BP_Flyer` reference | empty |
| `BomberToSpawn` | integer | 0 |
| `SatelliteToSpawn` | integer | 0 |

Compile; read the defaults back.

**Check:** `BP_Flyer` and `BP_DDGameMode` compile clean; defaults read back. **Save All.**

---

### Step 4 — `BP_DDGameMode`: helper functions (callees first)

Read the existing graphs first. Add these **new** functions, one small write each, in this order, non-Pure, **each read back**:

**`GetFlyerName`** (input `Kind` integer; output `Name` string):
```
If Kind == 0 → return "Bomber"   Else → return "Satellite"
```

**`GetFlyerSpeed`** (input `Kind` integer; output `Speed` float):
```
v = If Kind == 0 → Config.BomberSpeed Else Config.SatelliteSpeed       // a Select; store
return Select(v > 50.0, v, 50.0)                                        // floor at 50
```

**`GetFlyerShotsMax`** (input `Kind` integer; output `Count` integer):
```
v = If Kind == 0 → Config.BomberShots Else Config.SatelliteShots       // a Select; store
return Select(v > 1, v, 1)                                              // floor at 1
```

**`GetWaveBomberCount`** (input `WaveNumber` integer; output `Count` integer):
```
w = Max(WaveNumber, 1)                                                  // store
If w < Config.BomberStartWave → return 0
n = Length(Config.WaveBomberCounts)                                     // store
If n <= 0 → return 0
i = (w - 1) Percent n                                                   // store
tot = GetWaveCount(w)                                                   // store
return Clamp(Config.WaveBomberCounts[i], 0, tot)
```

**`GetWaveSatelliteCount`**: the same with `Config.SatelliteStartWave` and `Config.WaveSatelliteCounts`.

**`FlyerShoot`** (inputs `Pos` vector, `bLog` boolean; output `Slot` integer) — launches one missile from `Pos`:
```
Alive = local array of integers (empty)
For s = 0 to 8:
    a = IsSlotAlive(s)                                                  // store
    If a → Alive.Add(s)
n = Length(Alive)                                                       // store
If n <= 0 → return -1
k = Random Integer In Range(0, n - 1)                                   // store FIRST
s2 = Alive[k]                                                           // store
tg = GetSlotPos(s2)                                                     // store
base = GetEnemySpeed(Wave)                                              // store
sp = base * Max(Config.FlyerMissileSpeedFactor, 0.1)                    // store
e = SpawnEnemy(Pos, tg, sp, s2, bLog)                                   // store (ignored)
return s2
```
(The loop body ends in a `Branch`; both ends lead to the loop's next iteration. The `n <= 0` branch returns -1; the rest runs on the False path.)

**`OnFlyerKilled`** (inputs `Kind` integer, `bLog` boolean):
```
If Wave < 1 → return
base = If Kind == 0 → Config.ScoreBomber Else Config.ScoreSatellite    // a Select; store
pts = base * Multiplier                                                 // store
LastKillPoints = pts
AddScore(pts)
If bLog → DDLog("FlyerKill type=" + GetFlyerName(Kind) + " +" + pts + " score=" + Score, "Score", false)
```

**`ClearFlyers`** (input `bLog` boolean; output `Count` integer):
```
n = Length(ActiveFlyers)                                                // store
Copy = a copy of ActiveFlyers
For Each F in Copy: If IsValid(F) → Destroy Actor (F)
If bLog AND n > 0 → DDLog("Cleared n=" + n, "Flyer", false)
return n
```

**Check:** all eight compile clean and read back non-empty; none is called from anywhere yet. **Save All.**

---

### Step 5 — `BP_Flyer`: full content

Read the empty shell first.

**Components:** `Root` (Scene);
- `BodyBomber` StaticMeshComponent: mesh `/Engine/BasicShapes/Cube`, relative location `(0, 70, 0)`, relative scale `(1.0, 0.1, 0.3)`, material `MI_DD_Bomber`, NoCollision.
- `BodySatellite` StaticMeshComponent: mesh `/Engine/BasicShapes/Cube`, relative location `(0, 70, 0)`, relative rotation **Pitch = 45** (a rotation about Y, so the square becomes a diamond in the XZ plane; read back Roll = 0, Pitch = 45, Yaw = 0), relative scale `(0.5, 0.1, 0.5)`, material `MI_DD_Satellite`, NoCollision.

**Class defaults:** **Can Ever Tick = true** and **Start With Tick Enabled = true**.

**Variables** (not Instance Editable):

| Variable | Type | Default |
|---|---|---|
| `FlyerType` | integer | 0 |
| `Dir` | float | 1.0 |
| `Speed` | float | 0 |
| `StartX` | float | 0 |
| `VisibleSpan` | float | 0 |
| `EdgeMargin` | float | 0 |
| `PathLength` | float | 0 |
| `ShotCount` | integer | 0 |
| `ShotIndex` | integer | 0 |
| `NextShotDist` | float | 0 |
| `bActive` | boolean | false |
| `bLogEvents` | boolean | false |
| `bPlayerKill` | boolean | false |
| `GM` | `BP_DDGameMode` reference | none |

**Event BeginPlay:** `GM = Get Game Mode → Cast To BP_DDGameMode`.

**Functions** (all non-Pure; write in this order; read each back):

**`GetTypeName`** (output `Name` string): `return GM.GetFlyerName(FlyerType)` (guard `IsValid(GM)`; otherwise return `"?"`).

**`PickShotDist`** (no inputs) — schedules the next shot (3.3):
```
i = ShotIndex ; k = ShotCount                                           // store both
r = Random Float In Range(0.15, 0.85)                                   // store FIRST
kk = Select(k > 1, k, 1)                                                // store
NextShotDist = EdgeMargin + VisibleSpan * (float(i) + r) / float(kk)
```

**`Init`** (inputs `TypeIn` integer, `StartLoc` vector, `DirIn` float, `SpeedIn` float, `ShotsIn` integer, `SpanIn` float, `MarginIn` float, `bLog` boolean):
```
FlyerType = TypeIn ; Dir = DirIn ; StartX = StartLoc.X
Speed = Select(SpeedIn > 50.0, SpeedIn, 50.0)
VisibleSpan = SpanIn ; EdgeMargin = MarginIn ; PathLength = SpanIn + 2.0 * MarginIn
ShotCount = Select(ShotsIn > 0, ShotsIn, 0) ; ShotIndex = 0 ; bLogEvents = bLog
isB = (TypeIn == 0)                                                     // store a boolean local
SetActorLocation(StartLoc)
BodyBomber.SetVisibility(isB, false)                                    // both pins wired; rule 12
BodySatellite.SetVisibility(NOT isB, false)
If ShotCount > 0 → PickShotDist()
bActive = true
```

**`IsInsideExplosion`** (no inputs; output `bHit` boolean) — the same as the missile's:
```
bPlayerKill = false
Loc = GetActorLocation ; Hit = false (local)
For Each E in GM.ActiveExplosions:
    If IsValid(E):
        r = E.ContainsPoint(Loc)                                        // store
        If r:
            Hit = true
            If E.bPlayerMade → bPlayerKill = true
return Hit
```
(Do not spawn or destroy anything inside this loop.)

**`Shoot`** (no inputs):
```
ShotIndex = ShotIndex + 1
P = GetActorLocation                                                    // store
slot = GM.FlyerShoot(P, bLogEvents)                                     // store
If bLogEvents → DDLog("Shot type=" + GetTypeName() + " slot=" + GM.GetSlotName(slot) + " X=" + Round(P.X) + " Z=" + Round(P.Z), "Flyer", false)
If ShotIndex < ShotCount → PickShotDist()
```
(`GetSlotName(-1)` returns `INVALID`, which is the "nothing launched" case.)

**`Die`** (no inputs):
```
bActive = false
P = GetActorLocation                                                    // store first
If bPlayerKill → GM.OnFlyerKilled(FlyerType, bLogEvents)                // both branch ends go to the next node
ex = GM.SpawnExplosion(P, bLogEvents)  → wire the returned explosion into a Set bPlayerMade node, value = bPlayerKill
If bLogEvents → DDLog("Killed type=" + GetTypeName() + " X=" + Round(P.X) + " Z=" + Round(P.Z), "Flyer", false)
Destroy Actor
```
**Fallback:** if the returned object cannot be wired to the next node (as in 6.1), use a variable `NewExplosionRef` (type `BP_Explosion` reference) on `BP_Flyer`, set right after the call. Record it.

**`Leave`** (no inputs):
```
bActive = false
If bLogEvents → DDLog("Exit type=" + GetTypeName() + " X=" + Round(GetActorLocation.X), "Flyer", false)
Destroy Actor
```

**`Advance`** (input `Dt` float):
```
If NOT bActive → return
Hit = IsInsideExplosion()                                               // store in a local first
If Hit → Die() ; return
Loc = GetActorLocation                                                  // store
nx = Loc.X + Dir * Speed * Dt                                           // store
SetActorLocation((nx, Loc.Y, Loc.Z))
d = (nx - StartX) * Dir                                                 // store
If ShotIndex < ShotCount AND d >= NextShotDist → Shoot()
If d >= PathLength → Leave()                                            // both ends of the previous If lead here
```

**Event Tick(DeltaSeconds):** `Advance(DeltaSeconds)`.

**Event EndPlay:** `If IsValid(GM) → GM.ActiveFlyers.RemoveItem(Self)`.

**Check:** `BP_Flyer` and `BP_DDGameMode` compile clean. Read back both tick flags `true`; for both bodies mesh, location Y = 70, scale, material, and **collision disabled**; the satellite's rotation; every function read back non-empty. **Save All.**

---

### Step 6 — `BP_DDGameMode`: `SpawnFlyer` and `MaybeSpawnFlyer`

New functions, one small write each, in this order, non-Pure, **each read back**:

**`SpawnFlyer`** (inputs `Kind` integer, `bLog` boolean; output `Spawned` integer):
```
If bGameOver → return 0
side = Random Integer In Range(0, 1)                                    // store FIRST
h0 = Config.FlyerHeightMin ; h1 = Config.FlyerHeightMax                 // store both
lo = Select(h0 < h1, h0, h1) ; hi = Select(h0 < h1, h1, h0)             // store both
h = Random Float In Range(lo, hi)                                       // store FIRST
smax = GetFlyerShotsMax(Kind)                                           // store
k = Random Integer In Range(1, smax)                                    // store FIRST
spd = GetFlyerSpeed(Kind)                                               // store
minX = VisibleMinX ; maxX = VisibleMaxX ; maxZ = VisibleMaxZ            // store all three
span = maxX - minX                                                      // store
z = Config.GroundTopZ + h * (maxZ - Config.GroundTopZ)                  // store
sx = Select(side == 0, minX - 120.0, maxX + 120.0)                      // store
dr = Select(side == 0, 1.0, -1.0)                                       // store
start = (sx, Config.PlayPlaneY, z)                                      // store
F = Spawn Actor From Class(BP_Flyer, Location = start, Collision = Always Spawn)   // local BP_Flyer reference
F.Init(Kind, start, dr, spd, k, span, 120.0, bLog)
ActiveFlyers.Add(F)
If bLog → DDLog("Spawn type=" + GetFlyerName(Kind) + " fromX=" + Round(sx) + " Z=" + Round(z) + " dir=" + Round(dr) + " shots=" + k + " speed=" + Round(spd), "Flyer", false)
return 1
```
(If this is more than about 50 nodes, split the position part into a helper that returns floats one at a time; do not write a bigger function in one go.)

**`MaybeSpawnFlyer`** (no inputs):
```
bt = BomberToSpawn ; st = SatelliteToSpawn ; wt = WaveToSpawn          // store all three
rem = bt + st                                                           // store
If rem <= 0 OR wt <= 0 → return
p = float(rem) / float(wt)                                              // store
r = Random Float In Range(0.0, 1.0)                                     // store FIRST
If r >= p → return
r2 = Random Float In Range(0.0, 1.0)                                    // store FIRST
isB = (bt > 0) AND (r2 * float(rem) < float(bt))                        // store a boolean local
kind = Select(isB, 0, 1)                                                // store
act = Length(ActiveFlyers) ; capRaw = Config.FlyerMaxActive             // store both
cap = Select(capRaw > 1, capRaw, 1)                                     // store
If act >= cap → return
sp = SpawnFlyer(kind, NOT bDirectorSilent)                              // store
If sp > 0:
    If kind == 0 → BomberToSpawn = bt - 1
    Else         → SatelliteToSpawn = st - 1
```
(The two `return`s and the nested `If` end the function; nothing follows them.)

**Check:** both compile clean and read back non-empty; neither is called from anywhere yet; `SpawnFlyer` calls `BP_Flyer.Init` with all eight pins wired. **Save All.**

---

### Step 7 — Four small edits in existing `BP_DDGameMode` functions

Read each function first. **Small node edits only. No `write_graph_dsl` on these four.**

**7a. `StartWave`.** After 8a it reads: `Wave = Max(WaveNumber, 1) ; WaveTotal = GetWaveCount(Wave) ; WaveToSpawn = WaveTotal ; m = GetWaveMirvCount(Wave) ; MirvToSpawn = m ; Multiplier = ... ; RebuildBases() ; ...`. **After `MirvToSpawn = m` and before the next node**, insert, in this order:
```
b = GetWaveBomberCount(Wave)          // non-Pure call; its output goes straight into the next node
BomberToSpawn = b
s = GetWaveSatelliteCount(Wave)       // same
SatelliteToSpawn = s
```
Nothing else changes.

**7b. `SpawnDue`.** After 8a its loop body is `e = SpawnRandomEnemy(NOT bDirectorSilent) ; If IsValid(e) → [MaybeMakeMirv(e)] → WaveToSpawn = WaveToSpawn - 1`. **Between `MaybeMakeMirv(e)` and the decrement**, insert one call **`MaybeSpawnFlyer()`**. (Both 8a and flyers read `WaveToSpawn` before the decrement.) The loop bound, the early `return`s and `NextSpawnTime` stay exactly as they are.

**7c. `CheckWaveEnd`.** After 7.1 it reads: `NothingStands → Branch(bNone): True → EndWaveNoTargets(Now) [path ends]; False → If WaveToSpawn > 0 → return ; If Length(ActiveEnemies) > 0 → return ; EndWave(Now)`. **Between the `Length(ActiveEnemies) > 0` branch's False pin and the `EndWave(Now)` call**, insert a new Branch whose condition is `Length(ActiveFlyers) > 0` (a `Length` node and an integer `>` node, second input `0`; read both back): **True → nothing (the path ends)**; **False → `EndWave(Now)`**. Nothing else changes; `EndWave` is still called from `CheckWaveEnd` and `EndWaveNoTargets` only.

**7d. `ClearEnemies`.** It reads: `n = Length(ActiveEnemies) ; Copy ... ; For Each E in Copy → Destroy ; If bLog → DDLog("Cleared n=" ...) ; return n`. **At the very start**, before the first node, insert one call **`ClearFlyers(bLog)`** with its `bLog` pin wired from `ClearEnemies`' own `bLog` input (not left at default). Its returned count is ignored. The rest, including the returned `n`, stays exactly as it is.

**Check:** `BP_DDGameMode` compiles clean. `GetWaveBomberCount` and `GetWaveSatelliteCount` are called from `StartWave` only; `MaybeSpawnFlyer` from `SpawnDue` only, between `MaybeMakeMirv(e)` and the decrement; `ClearFlyers` from `ClearEnemies` only; `SpawnFlyer` from `MaybeSpawnFlyer` only (the keys come in Step 8). **Save All.**

---

### Step 8 — `BP_DDPlayerController`: keys `F` and `G`

Read the controller's event graph first. Create the key events **`F`** and **`G`** (`create_node` + `connect_pins`, **Pressed** pin only):
- `F` → `GM.SpawnFlyer(0, true)` (the `Kind` pin is `0`, `bLog` is `true`)
- `G` → `GM.SpawnFlyer(1, true)` (the `Kind` pin is **`1`**, `bLog` is `true`)

Read the input pins of both call nodes back. No other event changes.

**Check:** `BP_DDPlayerController` compiles clean; the keys are exactly `H`, `R`, `E`, `K`, `X`, `V`, `N`, `P`, `Delete`, `B`, `M`, `F`, `G` (+ Android Back); mouse and touch events unchanged; the startup chain unchanged. **Save All.**

---

### Step 9 — Run

1. **Save All.**
2. `StartPIE`. The title screen appears at once. **Wait about 20 seconds**; nobody taps, so no game starts.
3. `CaptureEditorImage` once at about 5 s (the title with the exit icon, as in 8a).
4. `GetLogEntries` (filter `[DD]`), then `StopPIE`.
5. Fill in the report (section 8).

---

## 7. Acceptance tests

### A. Static checks
- [ ] **A1** New assets: exactly `BP_Flyer`, `MI_DD_Bomber`, `MI_DD_Satellite`. Config: exactly the 13 new values, read back as in 3.9; no other value changed. No Project Settings, input settings or `.ini` change.
- [ ] **A2** `PDA_DDConfig`, `BP_Flyer`, `BP_DDGameMode`, `BP_DDPlayerController` compile with 0 errors and 0 warnings. (`BP_EnemyMissile`, `BP_Explosion`, `BP_ABM` are untouched.)
- [ ] **A3** Materials: parent `M_DD_Unlit`; colors as in Step 2.
- [ ] **A4** `BP_Flyer`: both tick flags true; components and properties as in Step 5 (location Y = 70 for both bodies, scales, the satellite's Pitch 45, materials, collision disabled); the 14 variables with their defaults; `BeginPlay`, `Tick` and `EndPlay` as specified; `GetTypeName`, `PickShotDist`, `Init`, `IsInsideExplosion`, `Shoot`, `Die`, `Leave`, `Advance` read back non-empty and match Step 5 (the report lists each node count).
- [ ] **A5** `BP_DDGameMode`: `ActiveFlyers`, `BomberToSpawn`, `SatelliteToSpawn` exist with their defaults; the ten new functions (`GetFlyerName`, `GetFlyerSpeed`, `GetFlyerShotsMax`, `GetWaveBomberCount`, `GetWaveSatelliteCount`, `FlyerShoot`, `OnFlyerKilled`, `ClearFlyers`, `SpawnFlyer`, `MaybeSpawnFlyer`) read back non-empty and match Steps 4 and 6 (node counts in the report).
- [ ] **A6** `Advance` (flyer): the explosion test comes first; `Die` then `return` on a hit; the move keeps `Y` and `Z`; the shot test uses `ShotIndex < ShotCount AND d >= NextShotDist`; the exit test uses `d >= PathLength`. `Shoot` increments `ShotIndex` **before** `PickShotDist`. `Die` scores only when `bPlayerKill` and sets the new explosion's `bPlayerMade` from `bPlayerKill`.
- [ ] **A7** `SpawnFlyer`: returns 0 while `bGameOver`; every random read is a stored local; both `Init` and the log use the same stored values. `MaybeSpawnFlyer`: the stored `bt`, `st`, `wt`, both random reads stored before use; the cap test; the decrement only when `SpawnFlyer` returned more than 0.
- [ ] **A8** The four edits (Step 7) are exactly as described: `StartWave` (four nodes after `MirvToSpawn = m`), `SpawnDue` (one call between `MaybeMakeMirv(e)` and the decrement; loop bound still a stored local), `CheckWaveEnd` (the new Branch on `Length(ActiveFlyers) > 0` before `EndWave`; the 7.1 first test unchanged), `ClearEnemies` (`ClearFlyers(bLog)` first, `bLog` wired from its input).
- [ ] **A9** Call audit (rule 12): the report lists, for `BP_Flyer.Init`'s two `SetVisibility` calls, `ClearFlyers` in `ClearEnemies`, `SpawnFlyer` in `MaybeSpawnFlyer` and in the keys, `FlyerShoot`, `OnFlyerKilled`, `ClearFlyers`'s caller pins and `BP_Flyer.Init` in `SpawnFlyer`, every input pin with where it is wired from (no pin left at a default by mistake; key `F` Kind = 0, key `G` Kind = 1, both `bLog` true).
- [ ] **A10** Keys: exactly `H`, `R`, `E`, `K`, `X`, `V`, `N`, `P`, `Delete`, `B`, `M`, `F`, `G` (+ Android Back).
- [ ] **A11** Unchanged: every function in rule 15; `BP_EnemyMissile`; the HUD; the startup chain; `StopSpawner`, `bDirectorSilent`.

### B. Log checks (agent; the one 20 s run, nobody taps)
- [ ] **B1** The startup lines as in the 8a run: camera, GameMode, config (`Width=2250.0 Height=1800.0`), bases 3, cities 6, PlayerController, view target, crosshair, HUD, `[DD][Area] Viewport=...`.
- [ ] **B2** Exactly one of `[DD][Save] Loaded high=<n>` or `[DD][Save] No file high=<n>`, followed by exactly one `[DD][Flow] Title high=<n>`.
- [ ] **B3** During the 20 s wait: no `Spawner started`, no `[DD][Wave]`, no `[DD][Enemy]`, **no `[DD][Flyer]`**, no `[DD][Score]`, no `[DD][Damage]`, no `[DD][Input] Fire request`, no `[DD][Game]`, and no `[DD][Flow]` line except the one `Title` line.
- [ ] **B4** No line containing `ERROR`, `Accessed None`, or `Blueprint Runtime Error`.

### C. Visual checks (editor image of the play view; NEEDS-HUMAN if you cannot capture it)
- [ ] **C1** The title screen is as in 8a (yellow `DOME DEFENSE`, `HIGH SCORE <n>`, exit icon top right, six cities, three bases with `10`). **No flyer is visible** (none was spawned).

### D. Interactive checks (human, PIE) — all NEEDS-HUMAN
List every item as NEEDS-HUMAN in the report; do not run or claim them.
- [ ] **D1** `F`: an orange rectangle enters from a random side at a height between 55% and 75% of the screen height, crosses at about 280 units/s, launches 1 or 2 missiles at standing targets, and leaves on the far side. Log: `Flyer Spawn type=Bomber ...`; for each shot an `Enemy Spawn ...` line then `Flyer Shot type=Bomber slot=...`; finally `Flyer Exit type=Bomber ...`.
- [ ] **D2** `G`: a cyan diamond, faster (about 380 units/s), 1 to 3 shots, the same log pattern with `type=Satellite`.
- [ ] **D3** Shooting a flyer (during a game): an explosion at its position; log order `Score FlyerKill type=... +<100 * multiplier>`, `Explosion Start`, `Flyer Killed ...`. Wave 1: +100; wave 3: +200; wave 7: +400. On the title screen no points.
- [ ] **D4** A flyer destroyed next to missiles takes them with it (chain reaction); the missile kills score `25 * multiplier` each.
- [ ] **D5** Natural play: waves 1 and 2 have no `Flyer Spawn`; bombers from wave 3 (table: waves 3 to 12 give 1, 1, 1, 1, 2, 1, 2, 2, 2, 2); satellites from wave 5 (waves 5 to 12 give 1, 1, 1, 1, 1, 2, 1, 2). A flyer may be missing when the cap of 2 blocked it. Never more than 2 on screen from the director.
- [ ] **D6** `[DD][Wave] Complete` comes after the last flyer's `Killed` or `Exit` and after the last missile's `Killed` or `Impact`.
- [ ] **D7** Accounting from the log, per wave: `Enemy Spawn` lines = wave count + sum of `Split n=` + `Flyer Shot` lines with `slot` other than `INVALID` (+ presses of `E` and `M`); kills + impacts = `Enemy Spawn` lines.
- [ ] **D8** `K`, `N`, game over, quit to title and starting a new game all remove flyers: none is left on screen. `K` logs `Flyer Cleared n=<k>` when flyers existed.
- [ ] **D9** Pause freezes flyers (and resume continues from the same place); the rest unchanged: `E`, `M`, `X`, `B`, `V`, the title and exit icon, bonus cities.
- [ ] **D10** Feel (no pass/fail): are the flyers too easy or too hard to hit, too many or too few, do their missiles come too fast? Knobs in `DA_DDConfig`: the two count tables, `BomberSpeed`, `SatelliteSpeed`, `FlyerMissileSpeedFactor`, `FlyerMaxActive`, the shot counts, the heights.

### D-DEVICE. Android (NEEDS-DEVICE)
- [ ] **T-1** On the device, flyers are visible and hittable by tapping, and the game runs smoothly with them; nothing else changed.

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped.

---

## 8. Report format (the agent's final message)

```
## Phase 8b report
**Engine version:** x.y.z
**MCP tools used:** (list)

### Assets created or changed
(table: path | type | change | compiled clean)

### Config values read back
(the 13 values, as read from DA_DDConfig, plus the three unchanged samples)

### Component and variable read-back (BP_Flyer)
(both bodies: mesh, location, rotation, scale, material, collision flag; tick flags; the 14 variables)

### Function inventory
(table: Blueprint | function | node count | read back non-empty)

### Call audit (rule 12)
(table: function | call | pin | wired from / value)

### Edits to existing functions (Step 7)
(for each of StartWave, SpawnDue, CheckWaveEnd, ClearEnemies: what the graph looked like before, which nodes were added and where)

### Acceptance results
A1 PASS/FAIL ... through E1 PASS/FAIL/NEEDS-HUMAN/NEEDS-DEVICE

### Log excerpt
(all [DD] lines of the run, with timestamps)

### Screenshot notes
### Tool problems (errors, retries, editor restarts, writes that read back empty)
(every tool error or timeout, what was tried, whether the editor stopped responding or restarted; "none" if none)

### Deviations
(AliveScratch or NewExplosionRef fallbacks, Select nodes used for Min/Max/Clamp, anything else)

### Open questions / risks
```

---

## 9. What this hands to later phases

| Provided | How a later phase uses it |
|---|---|
| `ActiveFlyers`, `ClearFlyers`, the flyer clause in `CheckWaveEnd` | 8c and later: any new kind of enemy that is not a missile gets its own list, a clear function called from `ClearEnemies`, and a clause in `CheckWaveEnd`. |
| `MaybeSpawnFlyer` in `SpawnDue`, `BomberToSpawn`, `SatelliteToSpawn` | 8c: smart bombs get the same kind of per-wave quota and hook (they are missiles, so they will probably use the `MaybeMakeMirv` pattern instead: mark an ordinary spawn). |
| `OnFlyerKilled(Kind, bLog)`, `ScoreBomber`, `ScoreSatellite` | 8c: `ScoreSmartBomb` (125) is scored the same way. |
| `FlyerShoot` | Any later actor that launches missiles from its own position. |
| Log accounting (3.11) | Later parts extend it. |

**Known leftovers that stay:** `StopSpawner` (stops nothing), `bDirectorSilent` and every `If NOT bDirectorSilent` (always false now), the two disconnected spawn chains in `SpawnRandomEnemy` that read `EnemyBaseSpeed`, the HUD debug lines 2 to 4, the overlay `BP_PlayfieldDebug`.

---

## 10. Common pitfalls

1. **Rewriting an existing function or event graph with `write_graph_dsl`.** New functions only; the four existing functions (Step 7) and the controller graph by small node edits.
2. **A write that reports success and reads back empty.** Read every new function back (rule 11).
3. **A pin left at its default** in a call you built (rule 12): `SetVisibility` (both pins; a flyer with the wrong visibility shows both bodies or none), `ClearFlyers(bLog)` in `ClearEnemies`, `Kind` on the `G` key (default 0 would spawn a bomber), `bLog` everywhere.
4. **Reading a random number twice.** Store `side`, `h`, `k` (`SpawnFlyer`), `r`, `r2` (`MaybeSpawnFlyer`), `r` (`PickShotDist`) and the slot index (`FlyerShoot`) first.
5. **Calling `MaybeSpawnFlyer` after the decrement.** It must read `WaveToSpawn` before `WaveToSpawn - 1`, or the last missile of the wave could never carry a flyer and the probability is wrong. It goes **after** `MaybeMakeMirv(e)`, **before** the decrement.
6. **Putting a flyer into `ActiveEnemies`** or counting it in a missile counter. Flyers live in `ActiveFlyers` only; `EnemiesKilled` is for missiles.
7. **Letting flyers use up `WaveToSpawn`.** They are extras; `SpawnFlyer` never touches the wave counters.
8. **Forgetting the wave-end clause.** Without the `Length(ActiveFlyers) > 0` branch a wave can end, rebuild cities and start the next wave while a flyer is still crossing. The branch goes after the `ActiveEnemies` test and before `EndWave`, on the False path of the 7.1 first test.
9. **Incrementing `ShotIndex` after `PickShotDist`.** `PickShotDist` uses `ShotIndex` as the index of the **next** shot; `Shoot` increments first.
10. **Moving the flyer before the explosion test**, or spawning the death explosion inside the loop over `ActiveExplosions`. The test is first; the explosion is spawned after `IsInsideExplosion` has finished.
11. **Changing Y or Z in `Advance`.** Only X changes.
12. **Scoring a flyer on title** or before the first wave: `OnFlyerKilled` returns at once while `Wave < 1`.
13. **Giving flyer missiles MIRV chances.** `FlyerShoot` calls `SpawnEnemy` directly and never `MaybeMakeMirv`.
14. **Scheduling more than one flyer per missile spawn.** `MaybeSpawnFlyer` makes at most one; that is what keeps the table counts exact.
15. **A flyer that can never leave.** `Speed` is floored at 50 in both `GetFlyerSpeed` and `Init`.
16. **Editing a frozen function** (rule 15); the only edits to existing functions are the four in Step 7.
17. **Marking D or T items PASS.** You cannot start a game, press keys or tap. They are NEEDS-HUMAN and NEEDS-DEVICE.
18. **Cleaning up what this document does not list.**
