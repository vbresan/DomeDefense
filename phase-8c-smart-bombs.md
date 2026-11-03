# Phase 8c — Smart Bombs (Dodging Missiles)

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Depends on** | Phase 8b (accepted: agent run and human play-tests, including pause, `K`, `N` and quit to title with flyers on screen) |
| **Phase goal** | From wave 6, some of a wave's enemy missiles are **smart bombs**: bigger magenta missiles that **sidestep nearby explosions**. A smart bomb flies like any missile toward its target at the wave speed, but whenever an explosion's edge comes within `SmartBombSenseRadius` of it, it is pushed away from that explosion at `SmartBombDodgeSpeed` and then keeps re-aiming at its own target. It can still be killed: by an explosion placed **in front of it on its path** (the push is slower than the explosion grows), by a very close shot, or by two explosions on opposite sides. A killed smart bomb scores **`ScoreSmartBomb` (125) times the wave multiplier** under the usual kill-origin rule and leaves a chain explosion. Smart bombs are part of the wave's normal missile count (each counts as one), are scheduled by a per-wave table (the same "mark an ordinary spawn" idea as the 8a MIRVs), at most 3 on screen. A desktop key `S` spawns one at once. This is the last part of Phase 8. |

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **No bars, no aspect caps.** The whole window is the playfield.
3. **Existing logic must survive** unless a step says to change it. Read each graph (`read_graph_dsl`) before editing it.
4. **Compile after every Blueprint change.** Every Blueprint ends with no errors and no warnings.
5. **Run each step's "Check"** before moving on. **Save All after each step's Check passes.**
6. **You cannot press keys, click, touch, or resize in PIE, and you cannot start a game** (a game starts with a tap on the title). You therefore **cannot exercise** the smart-bomb code. What you can verify: compiles, static inspection of every graph by `read_graph_dsl`, the config values, the assets' properties, and one clean title-screen run. Interactive checks (group D) are **NEEDS-HUMAN**; device checks are **NEEDS-DEVICE**. Never mark a D or T item PASS.
7. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
8. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after the run. **Save before loading a level.** **Do not touch Project Settings, input settings or any `Config/*.ini` file.**
9. **This phase adds exactly five config values** (Step 1), **one asset** (`MI_DD_SmartBomb`) and no other asset or config change.
10. **NEVER REWRITE AN EXISTING FUNCTION OR EVENT GRAPH** with `write_graph_dsl` (it deletes all nodes and rebuilds; on big graphs this ran the editor out of memory in Phase 5). A **brand-new** function may be written once, in **one small write**; aim for **about 60 nodes at most** and split into helpers if it would be bigger. **Existing graphs are changed only by small node edits** (`create_node`, `connect_pins`, break a link, delete one node). The key event `S` is made with `create_node` and `connect_pins`.
11. **READ EVERY NEW FUNCTION BACK right after writing it** (`read_graph_dsl`). In Phase 7 three writes reported success and read back empty. If a body reads back empty or incomplete: write it **once more**; if it is still wrong, **stop and report** under *Tool problems*. Do not continue on top of an empty function.
12. **Check every call you build for a boolean or number pin left at its default** (the cause of the Phase 7 pause defect). After building a call, read the node's input pins back. In this phase pay special attention to: both `Set Visibility` pins (`Dot`, `DotSmart`), `ArmSmart(Sense, Dodge)`, `OnSmartBombKilled(bLog)` in `Die`, `MaybeMakeSmart(e)` in `SpawnDue`, and the `bLog` of the `S` key call.
13. **After any tool error, do not retry the same large write.** Stop that item, try one smaller piece at most, and if it still fails **stop and report**. **If the editor stops responding or restarts, stop and report.** Do not continue with a half-edited graph.
14. **Order is part of the spec:** callees before callers (Steps 1 to 7).
15. **FROZEN: do not edit** `ApplyPause`, `GameOver`, `CheckGameOver`, `FinishGame`, `EndWave`, `EndWaveNoTargets`, `CheckWaveEnd`, `StepTally`, `RebuildCities`, `SpawnRandomEnemy`, `SpawnEnemy`, `DamageSlot`, `OnEnemyKilled`, `AddScore`, `SpawnExplosion`, `ClearEnemies`, `ClearFlyers`, the 8a functions (`ArmMirv`, `Split`, `CheckSplit`, `GetWaveMirvCount`, `MakeMirv`, `SpawnMirvChildren`, `MaybeMakeMirv`, `SpawnMirvDebug`), the 8b functions and `BP_Flyer`, `BP_EnemyMissile.Init`, `IsInsideExplosion`, `Impact`, `BP_Explosion`, `BP_ABM`, the flow functions (`EnterTitle`, `StartNewGame`, `ClearField`, `PauseGame`, `ResumeGame`, `QuitToTitle`), the HUD. **The only edits to existing functions are the four listed in Steps 5 and 6** (`BP_EnemyMissile.Advance`, `BP_EnemyMissile.Die`, `StartWave`, `SpawnDue`).
16. **Do not clean up anything this document does not list** (`StopSpawner`, `bDirectorSilent` and the places that test it, the two disconnected spawn chains in `SpawnRandomEnemy`, HUD lines 2 to 4, the overlay `BP_PlayfieldDebug`).

---

## 2. Scope

### In scope
- `PDA_DDConfig` and the instance `DA_DDConfig`: five new values.
- Material instance `MI_DD_SmartBomb`.
- `BP_EnemyMissile`: new variables, a second visual component `DotSmart`, new functions `ArmSmart`, `ComputePush`, `ApplyDodge`; two small edits (`Advance`, `Die`).
- `BP_DDGameMode`: new variable `SmartToSpawn`; new functions `GetWaveSmartCount`, `CountSmartBombs`, `OnSmartBombKilled`, `MaybeMakeSmart`, `SpawnSmartDebug`; two small edits (`StartWave`, `SpawnDue`).
- `BP_DDPlayerController`: new key event `S`.

### Out of scope (do NOT do)
- Real art, trails, sounds, particles, score pop-ups, any HUD change.
- Smart bombs that split, that are launched by flyers, or that change speed; any change to wave speeds, the other scoring, damage, the tally, bonus cities, targeting, the flow screens, the MIRV logic or the flyers.
- Any other key. The key set after this phase is exactly: `H`, `R`, `E`, `K`, `X`, `V`, `N`, `P`, `Delete`, `B`, `M`, `F`, `G`, **`S`** (new) and the Android Back key.

---

## 3. Design (decided; do not change)

### 3.1 What a smart bomb is
An ordinary enemy missile (`BP_EnemyMissile`) with `bSmart = true`. It has the same speed (the wave speed), target, impact, damage and kill rules as any missile. It is **never** a MIRV and never splits. Differences: its look (3.6), its score (3.4) and its **dodge** (3.2).

### 3.2 The dodge
Every tick, **after the explosion test** (so a bomb already inside an explosion still dies) and **before the normal flight step**, a smart bomb runs `ApplyDodge(Dt)`:

1. For every explosion `E` in `GM.ActiveExplosions` that is valid and `bActive`: `dx = Loc.X - E.X`, `dz = Loc.Z - E.Z`, `dist = sqrt(dx² + dz²)`. The explosion is **sensed** when `dist < E.Radius + SenseRadius` and `dist > 0.001` (`E.Radius` is the explosion's **current** radius; it grows from 0 to its maximum in 0.4 s). Each sensed explosion adds the unit vector `(dx / dist, dz / dist)` (pointing **away** from it) to the sum `(px, pz)`.
2. `len = sqrt(px² + pz²)`. The push magnitude is `mag = 1` if `len <= 1`, otherwise `1 / len` (so the push is never faster than `DodgeSpeed`, and two opposite explosions cancel each other).
3. If at least one explosion was sensed and `len > 0.001`: the bomb moves by `(px, pz) * mag * DodgeSpeed * Dt`, its new `X` clamped to `GM.VisibleMinX .. GM.VisibleMaxX` and its new `Z` clamped to `Target.Z .. GM.VisibleMaxZ` (a bomb never leaves the screen sideways or sinks below the ground line). `Y` does not change.
4. The normal flight step then runs from the **new** position (the existing `Advance` recomputes the direction to the target every tick, so the bomb re-aims by itself).

**Why it is killable (tuning note, not logic).** The explosion's radius grows at `ExplosionMaxRadius / ExplosionGrowTime = 160 / 0.4 = 400 units/s`; the default `DodgeSpeed` is 300, slower than that. A bomb flying **toward** an explosion approaches it at `Speed * cos(angle) - 300` and is caught; a bomb that is hit from **behind or beside** flees at 300 and usually escapes. Do not change these numbers; they are config values.

`bDodging` is true while the bomb senses at least one explosion; the **rising edge** (false to true) logs once (3.7).

### 3.3 Which missiles are smart bombs
- Per wave, up to `GetWaveSmartCount(Wave)` of the wave's director-spawned missiles; they are **part of** the wave's normal count `WaveTotal` (each counts as one).
- Chosen by sequential sampling, like the MIRVs: after each director spawn (after the MIRV roll, **before** the decrement of `WaveToSpawn`), if the missile is **not a MIRV** and fewer than `SmartBombMaxActive` smart bombs are alive, it becomes a smart bomb with probability `SmartToSpawn / WaveToSpawn`; then `SmartToSpawn` goes down by one. If the roll fails, the cap blocks it, or the missile is a MIRV, nothing happens and `SmartToSpawn` stays (the next spawn tries again). **The per-wave count can therefore be lower than the table** when the last spawns of a wave were MIRVs or the cap blocked; that is accepted.
- `GetWaveSmartCount(w)`: 0 if `w < SmartBombStartWave`; 0 if the table is empty; otherwise `Clamp(WaveSmartBombCounts[(w - 1) mod Length], 0, GetWaveCount(w))` (the table repeats like `WaveCounts`).
- Missiles from the keys `E`, `M` and `F`/`G` (flyer missiles), and warheads, are never smart; only `MaybeMakeSmart` (director spawns) and `SpawnSmartDebug` arm one.

### 3.4 Kill and score
- The kill test, the chain-reaction explosion and the kill-origin gate (`bPlayerKill`, 6.1) are unchanged.
- In `Die`, when `bPlayerKill` is true: a smart bomb calls `GM.OnSmartBombKilled(bLogEvents)`; any other missile still calls `GM.OnEnemyKilled(bLogEvents)`.
- `OnSmartBombKilled(bLog)`: nothing while `Wave < 1`; otherwise `pts = Config.ScoreSmartBomb * Multiplier`, `LastKillPoints = pts`, `AddScore(pts)`, and the log line below. `EnemiesKilled`, the `Killed` log line and the explosion are produced by `Die` for every missile as before.

### 3.5 Config
Instance Editable on `PDA_DDConfig`, category `SmartBombs`.

| Variable | Type | Default |
|---|---|---|
| `SmartBombStartWave` | integer | `6` |
| `WaveSmartBombCounts` | array of integer | `[0, 0, 0, 0, 0, 1, 1, 1, 2, 1, 2, 2]` |
| `SmartBombMaxActive` | integer | `3` |
| `SmartBombSenseRadius` | float | `120.0` |
| `SmartBombDodgeSpeed` | float | `300.0` |

(`ScoreSmartBomb` = 125 already exists from Phase 0; do not change it.)

### 3.6 Look (placeholder)
`BP_EnemyMissile` keeps its red `Dot`. A new component `DotSmart` (engine Sphere, relative location `(0, 70, 0)`, relative scale `(0.7, 0.7, 0.7)`, material `MI_DD_SmartBomb`, NoCollision) is **hidden by default**. `ArmSmart` hides `Dot` and shows `DotSmart`. Layering: the same enemy layer (relative Y = 70).

### 3.7 Debug key and log lines
- Key **`S`** (desktop only) → `GM.SpawnSmartDebug(true)`: spawns one random enemy (like `E`) and makes it a smart bomb. Nothing spawns during game over (the same as `E`). It works on the title and in a game.
- `K`, `N`, game over, quit to title: unchanged (smart bombs are ordinary missiles in `ActiveEnemies`).

| Category | Line |
|---|---|
| `Enemy` | `[DD][Enemy] Smart armed slot=<NAME> speed=<int>` (once per smart bomb, at arming; from `MaybeMakeSmart` guarded by `NOT bDirectorSilent`; from `SpawnSmartDebug` when `bLog`) |
| `Enemy` | `[DD][Enemy] Dodge X=<int> Z=<int>` (from the missile, on the rising edge of `bDodging`, only when its `bLogEvents` is true) |
| `Score` | `[DD][Score] SmartKill +<pts> score=<n>` (from `OnSmartBombKilled`, when its `bLog` is true) |

**Order of a kill:** `[DD][Score] SmartKill`, `[DD][Explosion] Start`, `[DD][Enemy] Killed` (the same pattern as a missile).
**Accounting is unchanged:** smart bombs are ordinary spawns inside the wave count. Per wave, `[DD][Enemy] Spawn` lines = `WaveTotal` + the sum of the `Split n=` values + the `Flyer Shot` lines whose `slot` is not `INVALID` (+ one per press of `E`, `M` or `S`); kills + impacts = `Spawn` lines.

---

## 4. What this phase changes in earlier specs

| Earlier text | Now |
|---|---|
| Phase 4 `Advance` (after 8a) | One call inserted: `ApplyDodge(Dt)` (Step 5a). |
| Phase 6.1 `Die` | The score call is chosen by `bSmart` (Step 5b). |
| Phase 6 `StartWave`, `SpawnDue` | One small addition each (Step 6). |
| Phase 0 `ScoreSmartBomb` | Now used. |

---

## 5. Tooling notes

1. **Do not guess node IDs.** Use `find_node_types`. Key event ID pattern: `Input|KeyboardEvents|S`.
2. **A value read by a later node is evaluated when that node runs.** This includes **random nodes** (each read rolls again) and **loop bounds**. Store every random read in a local before using it, and store a loop bound in a local before the loop.
3. **Non-Pure functions need exec wires.** Call each in sequence and store each result in its own local. A non-Pure node's output may be wired directly into the very next node of the same exec chain.
4. **No nodes after a Branch or a Return** in the same chain. Where steps follow an `If`, connect **both branch ends to the same next exec pin**. A void function cannot create a Return node; a path ending on an unconnected Then pin is fine.
5. **Divisions are guarded by a Branch, never by a Select.** A Select evaluates both inputs, and a float division by zero logs `Blueprint Runtime Error: Divide by zero` (the acceptance check B4 searches for that text). Every `/` in `ComputePush` and `ApplyDodge` sits on a path that is reached only when the divisor is greater than its guard.
6. **Seen in 8a and 8b (saves retries):** `RandomFloat` has no inputs, use `RandomFloatinRange` and `RandomIntegerinRange`; pin names such as `Index` (`GetSlotName`, `GetSlotPos`) and `WaveNumber` (`GetWaveCount`); `Max`, `Min`, `Clamp`, the integer remainder (`Percent`), the array `Get` and `RemoveIndex` have type IDs with parentheses that the graph writer cannot parse: build them with `create_node` and `connect_pins`, or use a Select as before (but not for divisions, note 5); log strings are Append chains (string `+` does not append); `Actor|SetActorLocation` is not a write id, add the node with `create_node`; the break-vector type ID is `Math|Vector|BreakVector`; the game-over getter write id is `GetGameOver`. Every such node is read back after placing.
7. **Function with several outputs:** `ComputePush` has three outputs. Store each output in its own local after the call (as `HandleTitleTapAt` did with the four outputs of `GetPauseButtonRect`).
8. **Components set in the Blueprint defaults:** read back mesh, relative location, relative scale, material, **visibility** and the **collision-enabled flag** (not only the preset). The tick flags are `PrimaryActorTick.bCanEverTick` and `bStartWithTickEnabled`; collision is `bodyInstance.collisionEnabled = NoCollision` (found in 8b).
9. **A new function cannot return an object:** the helpers below return numbers, booleans and integers only.
10. **PIE tools:** `StartPIE`, `StopPIE`, `IsPIERunning`, `GetLogEntries`, `CaptureViewport`, `CaptureEditorImage`.

---

## 6. Implementation steps

### Step 1 — Config

Add the five variables of 3.5 to `PDA_DDConfig` (Instance Editable, category `SmartBombs`). Set the values on `DA_DDConfig` and **read all five back** (the array shows all twelve numbers). Also read back `ScoreSmartBomb` (125), `ExplosionMaxRadius` (160), `ExplosionGrowTime` (0.4) and `BomberStartWave` (3) as unchanged samples. **Do not change any other value.**

**Check:** five values read back; `PDA_DDConfig` compiles clean. **Save All.**

---

### Step 2 — Material

Create Material Instance `MI_DD_SmartBomb` in `/Game/DomeDefense/Materials/`, parent `M_DD_Unlit`, `Color = (1.0, 0.2, 1.0, 1.0)` (magenta).

**Check:** exists; parent is `M_DD_Unlit`; the color reads back. **Save All.**

---

### Step 3 — `BP_EnemyMissile`: variables, visual and new functions

Read the Blueprint first (its components `Root` and `Dot`, its variables, and `Init`, `IsInsideExplosion`, `Advance`, `Die`, `Impact`, `Split`, `CheckSplit`, `ArmMirv`). **Do not change any existing function in this step.**

**3a. Variables** (not Instance Editable):

| Variable | Type | Default |
|---|---|---|
| `bSmart` | boolean | false |
| `SenseRadius` | float | 0 |
| `DodgeSpeed` | float | 0 |
| `bDodging` | boolean | false |

**3b. Component `DotSmart`:** StaticMeshComponent, mesh `/Engine/BasicShapes/Sphere`, relative location `(0, 70, 0)`, relative scale `(0.7, 0.7, 0.7)`, material `MI_DD_SmartBomb`, NoCollision, **Visible = false**. **Fallback:** if the default visibility cannot be set to false, leave it and add `DotSmart.SetVisibility(false, false)` to the end of `Event BeginPlay` (one small node edit; both pins wired); record it.

Compile; read back the variables and `DotSmart` properties.

**3c. New functions** (non-Pure; one small write each, in this order; **each read back**):

**`ArmSmart`** (inputs `SenseIn` float, `DodgeIn` float):
```
bSmart = true
SenseRadius = Select(SenseIn > 0.0, SenseIn, 0.0)
DodgeSpeed  = Select(DodgeIn > 0.0, DodgeIn, 0.0)
bDodging = false
Dot.SetVisibility(false, false)                                   // both pins wired; rule 12
DotSmart.SetVisibility(true, false)
```

**`ComputePush`** (no inputs; outputs `PushX` float, `PushZ` float, `bSensed` boolean) — the loop of 3.2 step 1:
```
Loc = GetActorLocation                                            // store
px = 0.0 ; pz = 0.0 ; sensed = false                              // function locals
For Each E in GM.ActiveExplosions:
    If IsValid(E):
        act = E.bActive ; rad = E.Radius ; EL = E.GetActorLocation          // store each
        dx = Loc.X - EL.X ; dz = Loc.Z - EL.Z                               // store both
        dist = Sqrt(dx * dx + dz * dz)                                      // store
        reach = rad + SenseRadius                                           // store
        If act AND dist < reach AND dist > 0.001:
            px = px + dx / dist ; pz = pz + dz / dist ; sensed = true       // division safe: dist > 0.001 on this path
return PushX = px ; PushZ = pz ; bSensed = sensed
```
(Do not spawn or destroy anything inside the loop. The loop body ends in a `Branch`; both ends lead to the loop's next iteration. Set the three outputs after the loop's **Completed** pin.)

**`ApplyDodge`** (input `Dt` float) — 3.2 steps 2 to 4:
```
If NOT bSmart → return
pr = ComputePush()  →  store the three outputs in locals: pushX, pushZ, sensed
Loc = GetActorLocation                                            // store
len = Sqrt(pushX * pushX + pushZ * pushZ)                         // store
mag = 1.0 (local float)
If len > 1.0 → mag = 1.0 / len                                    // a Branch, not a Select (note 5)
moving = sensed AND (len > 0.001)                                 // store a boolean local
If moving:
    step = DodgeSpeed * Dt * mag                                  // store
    nx = Clamp(Loc.X + pushX * step, GM.VisibleMinX, GM.VisibleMaxX)       // store
    nz = Clamp(Loc.Z + pushZ * step, Target.Z, GM.VisibleMaxZ)             // store
    SetActorLocation((nx, Loc.Y, nz))
(both ends of the If lead on)
If sensed AND NOT bDodging AND bLogEvents → DDLog("Dodge X=" + Round(Loc.X) + " Z=" + Round(Loc.Z), "Enemy", false)
(both ends of that If lead on)
bDodging = sensed
```
(`Loc` is the position **before** the move; the log uses it.)

**Check:** `BP_EnemyMissile` compiles clean; `ArmSmart`, `ComputePush`, `ApplyDodge` read back non-empty; none is called from anywhere yet; no existing function was changed. **Save All.**

---

### Step 4 — `BP_DDGameMode`: new variable and functions (callees first)

Read the existing graphs first. Add the variable **`SmartToSpawn`** (integer, default 0, not Instance Editable); compile; read the default back. Then add these **new** functions (one small write each, in this order, non-Pure), **each read back**:

**`GetWaveSmartCount`** (input `WaveNumber` integer; output `Count` integer):
```
w = Max(WaveNumber, 1)                                            // store
If w < Config.SmartBombStartWave → return 0
n = Length(Config.WaveSmartBombCounts)                            // store
If n <= 0 → return 0
i = (w - 1) Percent n                                             // store
tot = GetWaveCount(w)                                             // store
return Clamp(Config.WaveSmartBombCounts[i], 0, tot)
```

**`CountSmartBombs`** (no inputs; output `Count` integer):
```
n = Length(ActiveEnemies)                                         // store the loop bound first
c = 0 (local integer)
For i = 0 to n - 1:
    e = ActiveEnemies[i]                                          // store
    If IsValid(e) AND e.bSmart → c = c + 1
return c
```

**`OnSmartBombKilled`** (input `bLog` boolean):
```
If Wave < 1 → return
pts = Config.ScoreSmartBomb * Multiplier                          // store
LastKillPoints = pts
AddScore(pts)
If bLog → DDLog("SmartKill +" + pts + " score=" + Score, "Score", false)
```

**`MaybeMakeSmart`** (input `E` `BP_EnemyMissile` reference):
```
If NOT IsValid(E) → return
If E.bMirv → return
st = SmartToSpawn ; wt = WaveToSpawn                              // store both
If st <= 0 OR wt <= 0 → return
act = CountSmartBombs() ; capRaw = Config.SmartBombMaxActive      // store both
cap = Select(capRaw > 1, capRaw, 1)                               // store
If act >= cap → return
pm = float(st) / float(wt)                                        // store (wt > 0 here)
r = Random Float In Range(0.0, 1.0)                               // store FIRST
If r < pm:
    E.ArmSmart(Config.SmartBombSenseRadius, Config.SmartBombDodgeSpeed)
    SmartToSpawn = st - 1
    If NOT bDirectorSilent → DDLog("Smart armed slot=" + GetSlotName(E.SlotIndex) + " speed=" + Round(E.Speed), "Enemy", false)
```
(All the `return`s and the nested `If` end the function; nothing follows them.)

**`SpawnSmartDebug`** (input `bLog` boolean):
```
e = SpawnRandomEnemy(bLog)                                        // store
If NOT IsValid(e) → return
e.ArmSmart(Config.SmartBombSenseRadius, Config.SmartBombDodgeSpeed)
If bLog → DDLog("Smart armed slot=" + GetSlotName(e.SlotIndex) + " speed=" + Round(e.Speed), "Enemy", false)
```

**Check:** all five compile clean and read back non-empty; none is called from anywhere yet; `SmartToSpawn` default 0. **Save All.**

---

### Step 5 — `BP_EnemyMissile`: two small edits in existing functions

Read each function first. **Small node edits only. No `write_graph_dsl` on these two.**

**5a. `Advance`.** After 8a it reads: `If NOT bActive → return ; Hit = IsInsideExplosion() ; If Hit → Die() ; return ; Loc ... Remaining ... Step ... If Step >= Remaining → Impact Else → SetActorLocation(Loc + Dir * Step) → CheckSplit()`. **On the False pin of the `Hit` branch, before the first node of the flight calculation (the node that reads the location or the distance to `Target`), insert one call `ApplyDodge(Dt)`**, its `Dt` pin wired from `Advance`'s own `Dt` input. Nothing else in `Advance` changes: not the `Hit` test, not the `Impact` branch, not `CheckSplit`. (If the flight nodes read `GetActorLocation` through pure links, they now see the dodged position; that is intended. If `Loc` is stored by a Set node, that Set node must come **after** the new call.)

**5b. `Die`.** After 6.1 it reads: `bActive = false ; P = GetActorLocation ; GM.EnemiesKilled + 1 ; GM.LastKillPos = P ; Branch(bPlayerKill): True → GM.OnEnemyKilled(bLogEvents) → [next] ; False → [next]` where `[next]` is the `GM.SpawnExplosion(P, bLogEvents)` call, then the `bPlayerMade` set, the log and `Destroy Actor`. **Replace the True path** by a second Branch on `bSmart`:
- `Branch(bPlayerKill)` True → **new `Branch(bSmart)`**;
- `bSmart` True → new call **`GM.OnSmartBombKilled(bLogEvents)`** → `[next]`;
- `bSmart` False → the existing `GM.OnEnemyKilled(bLogEvents)` → `[next]`;
- `Branch(bPlayerKill)` False → `[next]` (unchanged).

(Several wires may lead into the same `[next]` exec pin. The `bLog` pin of the new call is wired from `bLogEvents`; read it back.)

**Check:** `BP_EnemyMissile` and `BP_DDGameMode` compile clean. `ApplyDodge` is called from `Advance` only; `ComputePush` from `ApplyDodge` only; `ArmSmart` from `MaybeMakeSmart` and `SpawnSmartDebug` only; `OnSmartBombKilled` from `Die` only; `OnEnemyKilled` is still called from `Die` only. **Save All.**

---

### Step 6 — `BP_DDGameMode`: two small edits

Read each function first. **Small node edits only.**

**6a. `StartWave`.** After 8b it reads: `... WaveToSpawn = WaveTotal ; m = GetWaveMirvCount(Wave) ; MirvToSpawn = m ; b = GetWaveBomberCount(Wave) ; BomberToSpawn = b ; s = GetWaveSatelliteCount(Wave) ; SatelliteToSpawn = s ; Multiplier = ... ; RebuildBases() ; ...`. **After `SatelliteToSpawn = s` and before the next node**, insert, in this order:
```
sm = GetWaveSmartCount(Wave)          // non-Pure call; its output goes straight into the next node
SmartToSpawn = sm
```
Nothing else changes.

**6b. `SpawnDue`.** After 8b its loop body is `e = SpawnRandomEnemy(NOT bDirectorSilent) ; If IsValid(e) → MaybeMakeMirv(e) → MaybeSpawnFlyer() → WaveToSpawn = WaveToSpawn - 1`. **Between `MaybeMakeMirv(e)` and `MaybeSpawnFlyer()`**, insert one call **`MaybeMakeSmart(e)`**, its `E` pin wired from the `e` local (read it back). (All three read `WaveToSpawn` before the decrement.) The loop bound, the early `return`s and `NextSpawnTime` stay exactly as they are.

**Check:** `BP_DDGameMode` compiles clean. `GetWaveSmartCount` is called from `StartWave` only; `MaybeMakeSmart` from `SpawnDue` only, after `MaybeMakeMirv(e)` and before `MaybeSpawnFlyer()`; `SpawnSmartDebug` from nowhere yet (the key comes in Step 7). **Save All.**

---

### Step 7 — `BP_DDPlayerController`: key `S`

Read the controller's event graph first. Create the key event **`S`** (`create_node` + `connect_pins`, **Pressed** pin only): `S` → `GM.SpawnSmartDebug(true)` (its `bLog` pin **true**; read it back). No other event changes.

**Check:** `BP_DDPlayerController` compiles clean; the keys are exactly `H`, `R`, `E`, `K`, `X`, `V`, `N`, `P`, `Delete`, `B`, `M`, `F`, `G`, `S` (+ Android Back); mouse and touch events unchanged; the startup chain unchanged. **Save All.**

---

### Step 8 — Run

1. **Save All.**
2. `StartPIE`. The title screen appears at once. **Wait about 20 seconds**; nobody taps, so no game starts.
3. `CaptureEditorImage` once at about 5 s (the title with the exit icon, as in 8b).
4. `GetLogEntries` (filter `[DD]`), then `StopPIE`.
5. Fill in the report (section 8).

---

## 7. Acceptance tests

### A. Static checks
- [ ] **A1** New assets: exactly `MI_DD_SmartBomb`. Config: exactly the five new values, read back as in 3.5; no other value changed. No Project Settings, input settings or `.ini` change.
- [ ] **A2** `PDA_DDConfig`, `BP_EnemyMissile`, `BP_DDGameMode`, `BP_DDPlayerController` compile with 0 errors and 0 warnings. (`BP_Flyer`, `BP_Explosion`, `BP_ABM` are untouched.)
- [ ] **A3** `MI_DD_SmartBomb`: parent `M_DD_Unlit`, color `(1.0, 0.2, 1.0, 1.0)`.
- [ ] **A4** `BP_EnemyMissile`: `bSmart` (false), `SenseRadius` (0), `DodgeSpeed` (0), `bDodging` (false); `DotSmart` (Sphere, location Y = 70, scale 0.7, `MI_DD_SmartBomb`, collision disabled, not visible by default, or the BeginPlay fallback recorded); `ArmSmart`, `ComputePush`, `ApplyDodge` read back non-empty and match Step 3 (the report lists each node count).
- [ ] **A5** `ApplyDodge` and `ComputePush`: the divisions (`dx / dist`, `dz / dist`, `1.0 / len`) each sit on a path guarded by a Branch (`dist > 0.001`, `len > 1.0`), not on a Select; the clamps use `GM.VisibleMinX`, `GM.VisibleMaxX`, `Target.Z`, `GM.VisibleMaxZ`; `Y` is not changed; the log is on the rising edge only; `bDodging = sensed` is set last.
- [ ] **A6** `Advance` calls `ApplyDodge(Dt)` once, on the False pin of the `Hit` branch, before the flight calculation; the `Hit` test, the `Impact` branch and `CheckSplit` are unchanged. `Die`: `Branch(bPlayerKill)` True leads to `Branch(bSmart)`; True calls `OnSmartBombKilled(bLogEvents)`, False calls `OnEnemyKilled(bLogEvents)`, both lead to the `SpawnExplosion` call; the `bPlayerKill` False path is unchanged.
- [ ] **A7** `BP_DDGameMode`: `SmartToSpawn` (0); `GetWaveSmartCount`, `CountSmartBombs`, `OnSmartBombKilled`, `MaybeMakeSmart`, `SpawnSmartDebug` read back non-empty and match Step 4 (node counts in the report). `MaybeMakeSmart` returns early for a MIRV (`E.bMirv`), reads `SmartToSpawn` and `WaveToSpawn` into locals first, checks the cap with `CountSmartBombs`, stores the random read first.
- [ ] **A8** `StartWave` sets `SmartToSpawn = GetWaveSmartCount(Wave)` right after `SatelliteToSpawn = s`. `SpawnDue` calls `MaybeMakeSmart(e)` between `MaybeMakeMirv(e)` and `MaybeSpawnFlyer()`, before the decrement; its loop bound is still a stored local.
- [ ] **A9** Call audit (rule 12): the report lists every call node added by this phase with its input pins and where each is wired from: both `SetVisibility` calls in `ArmSmart`, `ComputePush`'s three outputs in `ApplyDodge`, `ApplyDodge(Dt)` in `Advance`, `OnSmartBombKilled(bLog)` in `Die`, `ArmSmart` in `MaybeMakeSmart` and `SpawnSmartDebug`, `MaybeMakeSmart(e)` in `SpawnDue`, key `S` (`bLog` true).
- [ ] **A10** Keys: exactly `H`, `R`, `E`, `K`, `X`, `V`, `N`, `P`, `Delete`, `B`, `M`, `F`, `G`, `S` (+ Android Back).
- [ ] **A11** Unchanged: every function in rule 15; `BP_Flyer`; the HUD; the startup chain; `StopSpawner`, `bDirectorSilent`.

### B. Log checks (agent; the one 20 s run, nobody taps)
- [ ] **B1** The startup lines as in the 8b run: camera, GameMode, config (`Width=2250.0 Height=1800.0`), bases 3, cities 6, PlayerController, view target, crosshair, HUD, `[DD][Area] Viewport=...`.
- [ ] **B2** Exactly one of `[DD][Save] Loaded high=<n>` or `[DD][Save] No file high=<n>`, followed by exactly one `[DD][Flow] Title high=<n>`.
- [ ] **B3** During the 20 s wait: no `Spawner started`, no `[DD][Wave]`, no `[DD][Enemy]` (so no `Smart armed`, no `Dodge`), no `[DD][Flyer]`, no `[DD][Score]`, no `[DD][Damage]`, no `[DD][Input] Fire request`, no `[DD][Game]`, and no `[DD][Flow]` line except the one `Title` line.
- [ ] **B4** No line containing `ERROR`, `Accessed None`, `Divide by zero` or `Blueprint Runtime Error`.

### C. Visual checks (editor image of the play view; NEEDS-HUMAN if you cannot capture it)
- [ ] **C1** The title screen is as in 8b (yellow `DOME DEFENSE`, `HIGH SCORE <n>`, exit icon top right, six cities, three bases with `10`). Nothing new on screen.

### D. Interactive checks (human, PIE) — all NEEDS-HUMAN
List every item as NEEDS-HUMAN in the report; do not run or claim them.
- [ ] **D1** `S` (during a game): a **magenta, bigger** missile appears at the top and flies toward a standing target at the wave speed; log `Enemy Spawn ...`, then `Enemy Smart armed slot=... speed=...`.
- [ ] **D2** Tapping so that an explosion appears **beside or behind** the bomb makes it swerve away; the log shows `Enemy Dodge X=... Z=...` once per dodge. It keeps heading to its target afterwards and usually lands (`Impact`, damage as usual).
- [ ] **D3** Placing an explosion **ahead of the bomb on its path** (roughly 150 to 250 units in front) kills it: log order `Score SmartKill +<125 * multiplier>`, `Explosion Start`, `Enemy Killed`. Wave 1: +125; wave 6: +375; wave 7: +500. A missile close to the bomb's death point is taken by the chain explosion and scores as usual.
- [ ] **D4** Natural play: waves 1 to 5 have no `Smart armed` line; wave 6 has one, waves 7 and 8 one each, wave 9 up to two (table: waves 6 to 12 give 1, 1, 1, 2, 1, 2, 2). A wave may have one fewer than the table (3.3). Never more than 3 smart bombs alive from the director.
- [ ] **D5** Accounting unchanged: per wave, `Enemy Spawn` lines = wave count + sum of `Split n=` + `Flyer Shot` lines with `slot` other than `INVALID` (+ presses of `E`, `M`, `S`); kills + impacts = `Spawn` lines.
- [ ] **D6** Pause freezes smart bombs (and any dodge in progress); `K`, `N`, game over, quit to title clear them like any missile; nothing else changed (`E`, `M`, `F`, `G`, `X`, `B`, `V`, the title and exit icon, bonus cities).
- [ ] **D7** Feel (no pass/fail): is the dodge too easy or too hard to beat? A lead shot should usually kill; a shot from behind should usually miss. Knobs in `DA_DDConfig`: `SmartBombDodgeSpeed` (300; at 400 or more a single explosion can no longer catch a bomb that flees from it), `SmartBombSenseRadius` (120), the count table, `SmartBombMaxActive`, `SmartBombStartWave`.

### D-DEVICE. Android (NEEDS-DEVICE)
- [ ] **T-1** On the device, smart bombs are visible and hittable by tapping, and the game runs smoothly with them; nothing else changed.

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped.

---

## 8. Report format (the agent's final message)

```
## Phase 8c report
**Engine version:** x.y.z
**MCP tools used:** (list)

### Assets created or changed
(table: path | type | change | compiled clean)

### Config values read back
(the five values, as read from DA_DDConfig, plus the four unchanged samples)

### Component and variable read-back (BP_EnemyMissile)
(DotSmart: mesh, location, scale, material, visibility, collision flag; the four new variables)

### Function inventory
(table: Blueprint | function | node count | read back non-empty)

### Call audit (rule 12)
(table: function | call | pin | wired from / value)

### Edits to existing functions (Steps 5 and 6)
(for each of Advance, Die, StartWave, SpawnDue: what the graph looked like before, which nodes were added and where)

### Acceptance results
A1 PASS/FAIL ... through E1 PASS/FAIL/NEEDS-HUMAN/NEEDS-DEVICE

### Log excerpt
(all [DD] lines of the run, with timestamps)

### Screenshot notes
### Tool problems (errors, retries, editor restarts, writes that read back empty)
(every tool error or timeout, what was tried, whether the editor stopped responding or restarted; "none" if none)

### Deviations
(the DotSmart visibility fallback, Select nodes used for Min/Max/Clamp, anything else)

### Open questions / risks
```

---

## 9. What this hands to later phases

| Provided | How a later phase uses it |
|---|---|
| `bSmart`, `ArmSmart`, the `bSmart` branch in `Die` | A different kind of missile can be added the same way: a flag, an arming call, a score function. |
| `ApplyDodge` / `ComputePush` | Any later behavior that reacts to explosions (a flyer that evades, a missile that curves) can reuse the sensing loop. |
| `MaybeMakeSmart`, `SmartToSpawn`, `GetWaveSmartCount` | The pattern for per-wave quotas of marked missiles (the same as MIRVs). |
| All of Phase 8 | Phase 9: art, realistic explosions, sound, and an optional clean-up of the known leftovers below. |

**Known leftovers that stay:** `StopSpawner` (stops nothing), `bDirectorSilent` and every `If NOT bDirectorSilent` (always false now), the two disconnected spawn chains in `SpawnRandomEnemy` that read `EnemyBaseSpeed`, the HUD debug lines 2 to 4, the overlay `BP_PlayfieldDebug`, the keys `M`, `F`, `G`, `S`, `B`, `V`, `X`, `N`, `K`, `E` (debug keys).

---

## 10. Common pitfalls

1. **Rewriting an existing function or event graph with `write_graph_dsl`.** New functions only; the four existing functions (Steps 5 and 6) and the controller graph by small node edits.
2. **A write that reports success and reads back empty.** Read every new function back (rule 11).
3. **A pin left at its default** in a call you built (rule 12): `SetVisibility` on `Dot` and `DotSmart` (a bomb would show both dots or none), `bLog` on `OnSmartBombKilled`, the two floats of `ArmSmart` (zero sense and zero dodge would make the bomb ordinary), the `E` pin of `MaybeMakeSmart`.
4. **A division guarded by a Select instead of a Branch.** It logs `Divide by zero` at run time. Guard `dx / dist`, `dz / dist` and `1.0 / len` with a Branch.
5. **Running `ApplyDodge` before the explosion test.** It goes **after** the `Hit` branch, so a bomb already inside an explosion still dies. It also goes **before** the flight step, so the bomb flies from its new position.
6. **Changing `Y` or leaving the visible area.** The dodge moves only `X` and `Z`; `X` is clamped to the visible width, `Z` to `Target.Z .. GM.VisibleMaxZ`.
7. **Normalising the push always.** The push magnitude is `1` when `len <= 1` and `1 / len` otherwise, so opposite explosions cancel; normalising every push would make a sandwich useless.
8. **Spamming the log.** `Dodge` logs only on the rising edge of `bDodging`.
9. **Calling `MaybeMakeSmart` after the decrement or before `MaybeMakeMirv`.** It sits between `MaybeMakeMirv(e)` and `MaybeSpawnFlyer()`, before `WaveToSpawn - 1`, and it skips MIRVs (`E.bMirv`).
10. **Letting a smart bomb score as a plain missile.** The `Die` branch on `bSmart` is what gives 125 times the multiplier; `OnEnemyKilled` stays untouched.
11. **Making flyer missiles, warheads or `E`-key missiles smart.** Only `MaybeMakeSmart` and `SpawnSmartDebug` arm one.
12. **Editing a frozen function** (rule 15); the only edits to existing functions are the four in Steps 5 and 6.
13. **Marking D or T items PASS.** You cannot start a game, press keys or tap. They are NEEDS-HUMAN and NEEDS-DEVICE.
14. **Cleaning up what this document does not list.**
