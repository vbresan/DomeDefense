# Phase 8a — MIRVs (Splitting Missiles)

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Depends on** | Phase 7.2 (accepted: agent run and human play-test; the game was also tested on a handheld device, touch works, joysticks removed by the human in Project Settings) |
| **Phase goal** | Some of a wave's enemy missiles become **MIRVs**: on the way down, at a random height between 45% and 60% of the screen height, a MIRV **splits** into 1 to 3 **extra warheads**. Each warhead flies from the split point to a **different standing target** (different from the parent's target and from each other). The parent keeps flying to its own target. Warheads are ordinary missiles: 25 points times the wave multiplier, killed by explosions, chain reactions and the kill-origin rule as before. A MIRV killed before the split does not split. MIRVs look like ordinary missiles. Per-wave counts come from a new table in `DA_DDConfig`; wave 1 has none. A desktop debug key `M` spawns a MIRV that splits at once-ish, for testing. This is the first of three Phase 8 parts (8b flyers, 8c smart bombs). |

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **No bars, no aspect caps.** The whole window is the playfield.
3. **Existing logic must survive** unless a step says to change it. Read each graph (`read_graph_dsl`) before editing it.
4. **Compile after every Blueprint change.** Every Blueprint ends with no errors and no warnings.
5. **Run each step's "Check"** before moving on. **Save All after each step's Check passes.**
6. **You cannot press keys, click, touch, or resize in PIE, and you cannot start a game** (a game starts with a tap on the title). You therefore **cannot exercise** the MIRV code. What you can verify: compiles, static inspection of every graph by `read_graph_dsl`, the config values, and one clean title-screen run. Interactive checks (group D) are **NEEDS-HUMAN**; device checks are **NEEDS-DEVICE**. Never mark a D or T item PASS.
7. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
8. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after the run. **Save before loading a level.** **Do not touch Project Settings, input settings or any `Config/*.ini` file.**
9. **This phase adds no asset.** It adds exactly **five config values** (Step 1) and no other config change.
10. **NEVER REWRITE AN EXISTING FUNCTION OR EVENT GRAPH** with `write_graph_dsl` (it deletes all nodes and rebuilds; on big graphs this ran the editor out of memory in Phase 5). A **brand-new** function may be written once, in **one small write** (every new function below is under 40 nodes). **Existing graphs are changed only by small node edits** (`create_node`, `connect_pins`, break a link, delete one node). The key event `M` is made with `create_node` and `connect_pins`.
11. **READ EVERY NEW FUNCTION BACK right after writing it** (`read_graph_dsl`). In Phase 7 three writes reported success and read back empty. If a body reads back empty or incomplete: write it **once more**; if it is still wrong, **stop and report** under *Tool problems*. Do not continue on top of an empty function.
12. **Check every call you build for a boolean or number pin left at its default** (the cause of the Phase 7 pause defect). After building a call, read the node's input pins back.
13. **After any tool error, do not retry the same large write.** Stop that item, try one smaller piece at most, and if it still fails **stop and report**. **If the editor stops responding or restarts, stop and report.** Do not continue with a half-edited graph.
14. **Order is part of the spec:** callees before callers (Steps 1 to 6).
15. **FROZEN: do not edit** `ApplyPause`, `GameOver`, `CheckGameOver`, `FinishGame`, `EndWave`, `EndWaveNoTargets`, `CheckWaveEnd`, `StepTally`, `RebuildCities`, `SpawnRandomEnemy`, `SpawnEnemy`, `DamageSlot`, `BP_EnemyMissile.Init`, `IsInsideExplosion`, `Die`, `Impact`, the flow functions (`EnterTitle`, `StartNewGame`, `PauseGame`, `ResumeGame`, `QuitToTitle`), the HUD, the explosion and ABM Blueprints.
16. **Do not clean up anything this document does not list** (`StopSpawner`, `bDirectorSilent` and the places that test it, the two disconnected spawn chains in `SpawnRandomEnemy`, HUD lines 2 to 4, the overlay `BP_PlayfieldDebug`).

---

## 2. Scope

### In scope
- `PDA_DDConfig` and the instance `DA_DDConfig`: five new values.
- `BP_EnemyMissile`: new variables `bMirv`, `SplitHeight`, `SplitCount`; new functions `ArmMirv`, `Split`, `CheckSplit`; one call inserted in `Advance`.
- `BP_DDGameMode`: new variable `MirvToSpawn`; new functions `GetWaveMirvCount`, `MakeMirv`, `SpawnMirvChildren`, `MaybeMakeMirv`, `SpawnMirvDebug`; small edits in `StartWave` and `SpawnDue`.
- `BP_DDPlayerController`: new key event `M`.

### Out of scope (do NOT do)
- A different look, colour, sound or HUD marker for MIRVs or their warheads. No art.
- Bombers, satellites, smart bombs, UFOs (later parts). The arcade's cap of about three city losses per wave. Any change to waves, speeds, scoring, damage, the tally, bonus cities, targeting (`SpawnRandomEnemy`), the flow screens, or the HUD.
- Any other key. The key set after this phase is exactly: `H`, `R`, `E`, `K`, `X`, `V`, `N`, `P`, `Delete`, `B`, **`M`** (new) and the Android Back key.

---

## 3. Design (decided; do not change)

### 3.1 What a MIRV is
An enemy missile with `bMirv = true`. It flies exactly like any missile (straight line to its target at the wave speed). When, after a move, its height `Z <= SplitHeight`, it **splits once**: `bMirv` becomes false, and `SplitCount` warheads are spawned at its position. If it is killed first (`Die` runs at the top of `Advance`), it never splits.
Arcade basis: a missile splits at about mid-screen, into up to three extra missiles, each with a different target.

### 3.2 Which missiles are MIRVs
- Per wave, exactly `GetWaveMirvCount(Wave)` of the wave's missiles (the ones the director spawns) are MIRVs; they are **part of** the wave's normal count `WaveTotal`.
- Chosen by sequential sampling: each time the director spawns a missile, it is a MIRV with probability `MirvToSpawn / WaveToSpawn` (both read **before** the decrement of `WaveToSpawn`); then `MirvToSpawn` goes down by one. This gives exactly the table's count per wave and spreads them randomly.
- `GetWaveMirvCount(w)`: 0 if `w < MirvStartWave`; 0 if the table is empty; otherwise `Clamp(WaveMirvCounts[(w - 1) mod Length], 0, GetWaveCount(w))` (the table repeats like `WaveCounts`).
- Missiles from the debug key `E`, from `ClearEnemies`-related code, and the warheads themselves are never MIRVs (except the `M` key).

### 3.3 The split
- **Height:** when a MIRV is armed, `f` is a random float between `MirvSplitHeightMin` and `MirvSplitHeightMax` (config; `Min` and `Max` of the two are used, so the order does not matter); `SplitHeight = GroundTopZ + f * (VisibleMaxZ - GroundTopZ)`.
- **Warheads:** `SplitCount` = a random integer from 1 to `Max(MirvMaxChildren, 1)`, chosen at arming. At the split, the **candidate targets** are all standing slots (cities and bases) **except the parent's slot**. For each warhead, one candidate is picked at random and **removed** from the list, so no two warheads share a target. If the list runs out, fewer warheads are spawned (possibly none; then only the log line shows `n=0`).
- Each warhead is created with the existing `SpawnEnemy(Pos, TargetPos, FlightSpeed, Slot, bLog)` at the parent's position, with the **parent's speed** and the parent's `bLogEvents`. So each warhead logs its own `Spawn`, and later its own `Killed` or `Impact`, and its impact damages its slot through the normal path.
- Warheads **bypass** `WaveMaxActive` and do **not** use up `WaveToSpawn`. The wave still ends only when `WaveToSpawn == 0` and no enemy is left, so warheads keep the wave open (no change to `CheckWaveEnd`). While warheads exist they occupy active slots, so the director's `room` is smaller.

### 3.4 Everything else is unchanged
Scoring (a warhead kill is `ScoreEnemyMissile * Multiplier`, gated by the kill-origin rule), explosions, chain reactions, damage, the tally, bonus cities, game over, pause (warheads move on game time), `K`, `X`, `B`, `N`, `V`.

### 3.5 Debug key `M` (desktop only)
`M` → `GM.SpawnMirvDebug(true)`: spawns one random enemy (like `E`) and arms it with `SplitHeight = GroundTopZ + 0.80 * (VisibleMaxZ - GroundTopZ)` (so it splits soon after spawning) and `SplitCount = Max(MirvMaxChildren, 1)`. Works during a game or on the title; nothing spawns during game over (the same as `E`).

### 3.6 Config (Instance Editable, category `Mirv`)
| Variable | Type | Default |
|---|---|---|
| `MirvStartWave` | integer | `2` |
| `WaveMirvCounts` | array of integer | `[0, 1, 1, 2, 2, 2, 3, 2, 3, 3, 4, 3]` |
| `MirvMaxChildren` | integer | `3` |
| `MirvSplitHeightMin` | float | `0.45` |
| `MirvSplitHeightMax` | float | `0.60` |

### 3.7 Log lines
| Category | Line |
|---|---|
| `Enemy` | `[DD][Enemy] Mirv armed splitZ=<int> children=<k>` (once per MIRV, at arming; from `MakeMirv`, guarded by `NOT bDirectorSilent`; from `SpawnMirvDebug` when `bLog`) |
| `Enemy` | `[DD][Enemy] Split n=<made> X=<int> Z=<int>` (once per split, after the warheads' own `Spawn` lines; only when the parent's `bLogEvents` is true) |

Warheads produce ordinary `[DD][Enemy] Spawn slot=...`, `Killed`, `Impact` and `[DD][Damage]`, `[DD][Score]` lines. **Accounting from now on:** per wave, the number of `Spawn` lines = `WaveTotal` + the sum of the `Split n=` values; and kills + impacts = the number of `Spawn` lines.

---

## 4. What this phase changes in earlier specs

| Earlier text | Now |
|---|---|
| Phase 4 and 6: every enemy is a plain missile; per wave, spawns equal the count | Some missiles split; spawns per wave = count + warheads (3.7). |
| Phase 6: `StartWave` and `SpawnDue` | One small addition each (Step 5). |
| Phase 4: `BP_EnemyMissile.Advance` | One call inserted (Step 4). |

---

## 5. Tooling notes

1. **Do not guess node IDs.** Use `find_node_types`. Key event ID pattern: `Input|KeyboardEvents|M`.
2. **A value read by a later node is evaluated when that node runs.** This includes **random nodes** (each read rolls again) and **loop bounds**. Store every random read in a local or variable **before** using it, and store a loop bound in a local before the loop.
3. **Non-Pure functions need exec wires.** Call each in sequence and store each result in its own local. A non-Pure node's output may be wired directly into the very next node of the same exec chain.
4. **No nodes after a Branch or a Return** in the same chain. Where steps follow an `If`, connect **both branch ends to the same next exec pin**. A void function cannot create a Return node; a path ending on an unconnected Then pin is fine.
5. **Local arrays:** a function-local integer array is fine (`DestroyRandomBase` used one in Phase 7.1). **Fallback:** if a local array cannot be created, use a game-mode variable `AliveScratch` (array of integer, not Instance Editable; reuse the one from Phase 5 if it exists), cleared at the start of the function. Record it.
6. **Float and integer:** convert integers to float before dividing (`MaybeMakeMirv`). `Percent` is the integer remainder node.
7. **A new function cannot return an object:** the helpers below return integers only; objects are passed in.
8. **Properties on the config asset:** add the variables to `PDA_DDConfig`, then set the values on the **instance** `DA_DDConfig` and read them back (the array must show all twelve numbers).
9. **PIE tools:** `StartPIE`, `StopPIE`, `IsPIERunning`, `GetLogEntries`, `CaptureViewport`, `CaptureEditorImage`.

---

## 6. Implementation steps

### Step 1 — Config

Add the five variables of 3.6 to `PDA_DDConfig` (Instance Editable, category `Mirv`). Set them on `DA_DDConfig` and **read all five back** (the array shows `0, 1, 1, 2, 2, 2, 3, 2, 3, 3, 4, 3`). **Do not change any other value.**

**Check:** five values read back; `PDA_DDConfig` compiles clean. **Save All.**

---

### Step 2 — `BP_EnemyMissile`: variables and `ArmMirv`

Read the Blueprint first. Add variables (not Instance Editable):

| Variable | Type | Default |
|---|---|---|
| `bMirv` | boolean | false |
| `SplitHeight` | float | 0 |
| `SplitCount` | integer | 0 |

New function **`ArmMirv`** (inputs `HeightZ` float, `Children` integer; non-Pure), read back:
```
bMirv = true ; SplitHeight = HeightZ ; SplitCount = Children
```

**Check:** compiles clean; the three defaults read back; `ArmMirv` reads back non-empty. **Save All.**

---

### Step 3 — `BP_DDGameMode`: the MIRV functions

Read the existing graphs first. Add the variable **`MirvToSpawn`** (integer, default 0, not Instance Editable); compile; read the default back. Then add these **new** functions (one small write each, in this order, non-Pure), **each read back**:

**`GetWaveMirvCount`** (input `WaveNumber` integer; output `Count` integer):
```
w = Max(WaveNumber, 1)                                             // store
If w < Config.MirvStartWave → return 0
n = Length(Config.WaveMirvCounts)                                  // store
If n <= 0 → return 0
i = (w - 1) Percent n                                              // store
tot = GetWaveCount(w)                                              // store
return Clamp(Config.WaveMirvCounts[i], 0, tot)
```

**`MakeMirv`** (input `E` `BP_EnemyMissile` reference):
```
lo = Min(Config.MirvSplitHeightMin, Config.MirvSplitHeightMax)     // store both
hi = Max(Config.MirvSplitHeightMin, Config.MirvSplitHeightMax)
f = Random Float In Range(lo, hi)                                  // store FIRST
k = Random Integer In Range(1, Max(Config.MirvMaxChildren, 1))     // store FIRST
h = Config.GroundTopZ + f * (VisibleMaxZ - Config.GroundTopZ)      // store
E.ArmMirv(h, k)
If NOT bDirectorSilent → DDLog("Mirv armed splitZ=" + Round(h) + " children=" + k, "Enemy", false)
```

**`SpawnMirvChildren`** (inputs `Pos` vector, `ParentSlot` integer, `Count` integer, `FlightSpeed` float, `bLog` boolean; output `Spawned` integer):
```
Alive = local array of integers (empty)
For Slot = 0 to 8:
    a = IsSlotAlive(Slot)                                          // store
    If a AND Slot != ParentSlot → Alive.Add(Slot)
made = 0 (local integer) ; cnt = Count                             // store the loop bound first
For i = 0 to cnt - 1:
    n = Length(Alive)                                              // store
    If n > 0:
        k = Random Integer In Range(0, n - 1)                      // store FIRST
        s = Alive[k]                                               // store
        Alive.RemoveAt(k)                                          // each target is used once
        tg = GetSlotPos(s)                                         // store
        e = SpawnEnemy(Pos, tg, FlightSpeed, s, bLog)              // store (ignored)
        made = made + 1
If bLog → DDLog("Split n=" + made + " X=" + Round(Pos.X) + " Z=" + Round(Pos.Z), "Enemy", false)
return made
```
(The loop body ends in a `Branch`; both its ends lead to the loop's next iteration.)

**`MaybeMakeMirv`** (input `E` `BP_EnemyMissile` reference):
```
If NOT IsValid(E) → return
mt = MirvToSpawn ; wt = WaveToSpawn                                // store both
If mt <= 0 OR wt <= 0 → return
pm = float(mt) / float(wt)                                         // store
r = Random Float In Range(0.0, 1.0)                                // store FIRST
If r < pm:
    MakeMirv(E)
    MirvToSpawn = mt - 1
```

**`SpawnMirvDebug`** (input `bLog` boolean):
```
e = SpawnRandomEnemy(bLog)                                         // store
If NOT IsValid(e) → return
h = Config.GroundTopZ + 0.80 * (VisibleMaxZ - Config.GroundTopZ)   // store
k = Max(Config.MirvMaxChildren, 1)                                 // store
e.ArmMirv(h, k)
If bLog → DDLog("Mirv armed splitZ=" + Round(h) + " children=" + k, "Enemy", false)
```

**Check:** all five compile clean and read back non-empty; none is called from anywhere yet; `MirvToSpawn` default 0. **Save All.**

---

### Step 4 — `BP_EnemyMissile`: the split

Two new functions (non-Pure; read back each), then one edit:

**`Split`** (no inputs):
```
bMirv = false                                                      // first: a MIRV splits once
If NOT IsValid(GM) → return
P = GetActorLocation                                               // store
n = GM.SpawnMirvChildren(P, SlotIndex, SplitCount, Speed, bLogEvents)   // store (ignored)
```

**`CheckSplit`** (no inputs):
```
If NOT bMirv → return
z = GetActorLocation.Z                                             // store
If z <= SplitHeight → Split()
```

**`Advance` (edit in place).** Read it. Its structure is: `If NOT bActive → return ; Hit = IsInsideExplosion() ; If Hit → Die ; return ; Loc ... Remaining ... Step ... If Step >= Remaining → Impact Else → SetActorLocation(Loc + Dir * Step)`. **After the `SetActorLocation(Loc + Dir * Step)` node in the Else branch, add one call node `CheckSplit()`** (that path ends there). Nothing else in `Advance` changes: not the `Hit` test, not the `Impact` branch.

**Check:** `BP_EnemyMissile` and `BP_DDGameMode` compile clean; `Split` and `CheckSplit` read back non-empty; `CheckSplit` is called from `Advance` only; `Split` from `CheckSplit` only; `SpawnMirvChildren` from `Split` only. **Save All.**

---

### Step 5 — `BP_DDGameMode`: two small edits

**5a. `StartWave` (edit in place).** Read it. It is `Wave = Max(WaveNumber, 1) ; WaveTotal = GetWaveCount(Wave) ; WaveToSpawn = WaveTotal ; Multiplier = ... ; RebuildBases() ; WaveState = 1 ; StateEndTime = ... ; NextSpawnTime = ... ; LogWaveStart()`. **Between `WaveToSpawn = WaveTotal` and the next node**, insert:
```
m = GetWaveMirvCount(Wave)                                         // non-Pure call; its output goes straight into the next node
MirvToSpawn = m
```
Nothing else changes.

**5b. `SpawnDue` (edit in place).** Read it. Its loop is `For i = 0 to NLocal - 1: e = SpawnRandomEnemy(NOT bDirectorSilent) ; If IsValid(e) → WaveToSpawn = WaveToSpawn - 1`. **On the True pin of `If IsValid(e)`, insert one call `MaybeMakeMirv(e)` BEFORE the `WaveToSpawn = WaveToSpawn - 1` node** (so `MaybeMakeMirv` reads `WaveToSpawn` before the decrement). The loop bound, the early `return`s and `NextSpawnTime` stay exactly as they are.

**Check:** `BP_DDGameMode` compiles clean; `GetWaveMirvCount` is called from `StartWave` only; `MaybeMakeMirv` from `SpawnDue` only; in `SpawnDue` the call sits between the `IsValid(e)` True pin and the decrement. **Save All.**

---

### Step 6 — `BP_DDPlayerController`: key `M`

Read the controller's event graph first. Create the key event **`M`** (`create_node` + `connect_pins`, **Pressed** pin only): `M` → `GM.SpawnMirvDebug(true)` (its `bLog` pin **true**; read it back). No other event changes.

**Check:** `BP_DDPlayerController` compiles clean; the keys are exactly `H`, `R`, `E`, `K`, `X`, `V`, `N`, `P`, `Delete`, `B`, `M` (+ Android Back); mouse and touch events unchanged; the startup chain unchanged. **Save All.**

---

### Step 7 — Run

1. **Save All.**
2. `StartPIE`. The title screen appears at once. **Wait about 20 seconds**; nobody taps, so no game starts.
3. `CaptureEditorImage` once at about 5 s (the title with the exit icon, as in 7.2).
4. `GetLogEntries` (filter `[DD]`), then `StopPIE`.
5. Fill in the report (section 8).

---

## 7. Acceptance tests

### A. Static checks
- [ ] **A1** New assets: **none**. Config: exactly the five new values; `DA_DDConfig` reads back `MirvStartWave = 2`, `WaveMirvCounts = [0, 1, 1, 2, 2, 2, 3, 2, 3, 3, 4, 3]`, `MirvMaxChildren = 3`, `MirvSplitHeightMin = 0.45`, `MirvSplitHeightMax = 0.60`. No other value changed. No Project Settings, input settings or `.ini` change.
- [ ] **A2** `BP_EnemyMissile`, `BP_DDGameMode`, `BP_DDPlayerController`, `PDA_DDConfig` compile with 0 errors and 0 warnings.
- [ ] **A3** `BP_EnemyMissile` has `bMirv` (false), `SplitHeight` (0), `SplitCount` (0), and `ArmMirv`, `Split`, `CheckSplit`, each read back non-empty. `Advance` calls `CheckSplit` once, after `SetActorLocation(Loc + Dir * Step)` in the Else branch; the `Hit` test and the `Impact` branch are unchanged.
- [ ] **A4** `BP_DDGameMode` has `MirvToSpawn` (0) and `GetWaveMirvCount`, `MakeMirv`, `SpawnMirvChildren`, `MaybeMakeMirv`, `SpawnMirvDebug`, each read back non-empty and matching Step 3 (the report lists the node count of each).
- [ ] **A5** `StartWave` sets `MirvToSpawn = GetWaveMirvCount(Wave)` right after `WaveToSpawn = WaveTotal`. `SpawnDue` calls `MaybeMakeMirv(e)` on the True pin of `If IsValid(e)`, before the decrement; its loop bound is still a stored local.
- [ ] **A6** `SpawnMirvChildren`: the target list excludes `ParentSlot`; each chosen slot is removed from the list; the loop bound and every random read are stored locals; the children are spawned with `SpawnEnemy` (not a new spawn path). `Split` clears `bMirv` first.
- [ ] **A7** Call audit (rule 12): the report lists, for `Split`, `CheckSplit`, `MakeMirv`, `MaybeMakeMirv`, `SpawnMirvDebug`, `SpawnMirvChildren` and the key `M`, every call node with its input pins and where each is wired from (no pin left at a default by mistake; `bLog` of the `M` key call is `true`).
- [ ] **A8** Keys: exactly `H`, `R`, `E`, `K`, `X`, `V`, `N`, `P`, `Delete`, `B`, `M` (+ Android Back). `M` calls `GM.SpawnMirvDebug(true)`.
- [ ] **A9** Unchanged: every function in rule 15; `SpawnRandomEnemy` and `SpawnEnemy` bodies; `Die`, `Impact`, `IsInsideExplosion`; the HUD; the startup chain; `StopSpawner`, `bDirectorSilent`.

### B. Log checks (agent; the one 20 s run, nobody taps)
- [ ] **B1** The startup lines as in the 7.2 run: camera, GameMode, config (`Width=2250.0 Height=1800.0`), bases 3, cities 6, PlayerController, view target, crosshair, HUD, `[DD][Area] Viewport=...`.
- [ ] **B2** Exactly one of `[DD][Save] Loaded high=<n>` or `[DD][Save] No file high=<n>`, followed by exactly one `[DD][Flow] Title high=<n>`.
- [ ] **B3** During the 20 s wait: no `Spawner started`, no `[DD][Wave]`, no `[DD][Enemy]` (so no `Mirv armed`, no `Split`), no `[DD][Score]`, no `[DD][Damage]`, no `[DD][Input] Fire request`, no `[DD][Game]`, and no `[DD][Flow]` line except the one `Title` line.
- [ ] **B4** No line containing `ERROR`, `Accessed None`, or `Blueprint Runtime Error`.

### C. Visual checks (editor image of the play view; NEEDS-HUMAN if you cannot capture it)
- [ ] **C1** The title screen is as in 7.2 (yellow `DOME DEFENSE`, `HIGH SCORE <n>`, exit icon top right, six cities, three bases with `10`). Nothing new on screen.

### D. Interactive checks (human, PIE) — all NEEDS-HUMAN
List every item as NEEDS-HUMAN in the report; do not run or claim them.
- [ ] **D1** `M` (during a game): a missile spawns, and soon after (about 20% of the way down) it splits into 3 warheads going to 3 different standing targets, none of them the parent's. Log order: `Mirv armed splitZ=... children=3`, `Spawn ...`, then three `Spawn ...`, then `Split n=3 ...`. All four end as `Killed` or `Impact`; kills score `25 * multiplier` each.
- [ ] **D2** Natural play: wave 1 has no `Mirv armed` line; from wave 2 the number of `Mirv armed` lines per wave equals the table (`1, 1, 2, 2, 2, 3, 2, 3, 3, 4, 3` for waves 2 to 12); each ends in a `Split n=<1..3>` line unless it was killed first.
- [ ] **D3** A MIRV shot down before the split produces no `Split` line and no warheads.
- [ ] **D4** The wave does not end while warheads are alive: `[DD][Wave] Complete` comes after the last warhead's `Killed` or `Impact`.
- [ ] **D5** Accounting from the log: per wave, `Spawn` lines = wave count + sum of `Split n=`; kills + impacts = `Spawn` lines.
- [ ] **D6** Nothing else changed: `K`, `X`, `B`, `N`, `V`, `E`, pause and resume (warheads freeze too), game over, bonus cities, the title and exit icon.
- [ ] **D7** Feel (no pass/fail): note whether the MIRV counts feel right. All five values are in `DA_DDConfig` (`WaveMirvCounts` is the main knob).

### D-DEVICE. Android (NEEDS-DEVICE)
- [ ] **T-1** On the device, splits are visible and the game runs smoothly with the extra warheads; nothing else changed.

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped.

---

## 8. Report format (the agent's final message)

```
## Phase 8a report
**Engine version:** x.y.z
**MCP tools used:** (list)

### Assets created or changed
(table: path | type | change | compiled clean)

### Config values read back
(the five values, as read from DA_DDConfig)

### Function inventory
(table: Blueprint | function | node count | read back non-empty)

### Call audit (rule 12)
(table: function | call | pin | wired from / value)

### Acceptance results
A1 PASS/FAIL ... through E1 PASS/FAIL/NEEDS-HUMAN/NEEDS-DEVICE

### Log excerpt
(all [DD] lines of the run, with timestamps)

### Screenshot notes
### Tool problems (errors, retries, editor restarts, writes that read back empty)
(every tool error or timeout, what was tried, whether the editor stopped responding or restarted; "none" if none)

### Deviations
(AliveScratch fallback, anything else)

### Open questions / risks
```

---

## 9. What this hands to later phases

| Provided | How a later phase uses it |
|---|---|
| `SpawnEnemy`-based warheads that bypass `WaveMaxActive` and `WaveToSpawn` | 8b: a flyer that launches missiles uses the same pattern (extras on top of the wave count, keeping the wave open). Decide there whether its missiles count against the cap. |
| `MaybeMakeMirv` in `SpawnDue` and `MirvToSpawn` | 8c: smart bombs get the same kind of per-wave quota and hook. |
| `GetWaveMirvCount` and `WaveMirvCounts` | The pattern for per-wave tables of special enemies. |
| Log accounting (3.7) | Later parts extend it: spawn lines = wave count + extras. |

**Known leftovers that stay:** `StopSpawner` (stops nothing), `bDirectorSilent` and every `If NOT bDirectorSilent` (always false now), the two disconnected spawn chains in `SpawnRandomEnemy` that read `EnemyBaseSpeed`, the HUD debug lines 2 to 4, the overlay `BP_PlayfieldDebug`.

---

## 10. Common pitfalls

1. **Rewriting an existing function or event graph with `write_graph_dsl`.** New functions only; existing graphs by small node edits.
2. **A write that reports success and reads back empty.** Read every new function back (rule 11).
3. **A pin left at its default** in a call you built (rule 12): `bLog` of the `M` key call, the arguments of `SpawnMirvChildren` in `Split`, `E` in `MakeMirv`.
4. **Reading a random number twice.** Store `f`, `k` (`MakeMirv`), `r` (`MaybeMakeMirv`) and the child index (`SpawnMirvChildren`) first.
5. **A loop bound that is a live expression.** Store `cnt` before the loop in `SpawnMirvChildren`.
6. **Calling `MaybeMakeMirv` after the decrement.** It must read `WaveToSpawn` before `WaveToSpawn - 1`, or the last missile of the wave could never be a MIRV and the probability is wrong.
7. **Letting a warhead use up `WaveToSpawn`.** Warheads are created in `SpawnMirvChildren` through `SpawnEnemy`, which does not touch the wave counters. Do not route them through `SpawnDue`.
8. **Splitting twice.** `Split` clears `bMirv` as its first node.
9. **Giving a warhead the parent's target.** The candidate list excludes `ParentSlot`.
10. **Reusing a target for two warheads.** Remove the chosen slot from the list.
11. **Making warheads or `E`-key missiles MIRVs.** Only `MaybeMakeMirv` (director spawns) and `SpawnMirvDebug` arm a MIRV.
12. **Editing a frozen function** (rule 15).
13. **Marking D or T items PASS.** You cannot start a game, press keys or tap. They are NEEDS-HUMAN and NEEDS-DEVICE.
14. **Cleaning up what this document does not list.**
