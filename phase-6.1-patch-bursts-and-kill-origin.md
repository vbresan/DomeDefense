# Phase 6.1 (patch) — Burst Size at the Cap and Kill Origin

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Type** | **Patch** to Phase 6. It changes existing behavior and adds tests. It adds no feature. |
| **Depends on** | Phase 6 (agent run: 31/31 old tests and 8/8 Phase 6 tests PASS; the human played seven waves to game over and the gameplay log was reviewed) |
| **Patch goal** | Two fixes. **(A)** A burst of two enemies is cut to one when the on-screen cap leaves room for exactly two (found in the log: bursts of 2, 2, 2, 1, 1). **(B)** An enemy killed by the explosion of **another enemy's landing** no longer scores points. Two new tests (T40, T41) cover both; the first run of the new tests must fail on purpose (red), the second must pass (green). |

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **No bars, no aspect caps.** The whole window is the playfield.
3. **Existing logic must survive** unless a step says to change it. Read each graph (`read_graph_dsl`) before editing it.
4. **Compile after every Blueprint change.** Every Blueprint ends with no errors and no warnings.
5. **Run each step's "Check"** before moving on. **Save All after each step's Check passes.**
6. **You cannot press keys, click, touch, or resize in PIE.** Group D checks are **NEEDS-HUMAN**; touch is **NEEDS-DEVICE**.
7. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
8. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after each run.
9. **This patch adds no config value and changes no config value.**
10. **NEVER REWRITE AN EXISTING FUNCTION OR EVENT GRAPH** with `write_graph_dsl` (it deletes all nodes and rebuilds; on big graphs this ran the editor out of memory). A **brand-new** function may be written once, in **one small write of about 40 nodes at most**; the new tests below are already split into small helpers for that reason. **Existing graphs are changed only by small node edits** (`create_node`, `connect_pins`, set a pin default, delete a single node).
11. **FROZEN FUNCTIONS: do not edit them.**
    - `BP_DDPlayerController`: `RunSelfTest`, `RunPhase2Tests`, `RunPhase3Tests`, `RunPhase4Tests`, `RunPhase4Rest`, `RunPhase5Tests`, `RunT32`, `RunT33`, `RunT34`, `RunT35`, `RunT36`, `RunT37`, `RunT38`, `RunT39`, `BonusCase`, `FreezeEnemies`, `SelfTestFlightStart`, `SelfTestFlightMid`, `SelfTestFlightEnd`, `SelfTestExplosionEnd`, `SelfTestEnemyStart`, `SelfTestEnemyEnd`.
    - **The only allowed edits in the controller:** new functions, and **three small edits in `RunPhase6Tests`** (Step 2).
12. **After any tool error, do not retry the same large write.** Stop that item, try one smaller piece at most, and if it still fails **stop and report** under *Tool problems*. **If the editor stops responding or restarts, stop and report.**
13. **The old tests must keep passing exactly as they are.** `[DD][Test] SelfTest done: 31/31 PASS` stays valid. The Phase 6 line changes from `n/8` to `n/10` (Step 2).
14. **Order is part of the spec.** Step 3 is a deliberate **red run** (the new tests must fail before the fixes exist). Do **not** apply the fixes before it.

---

## 2. Scope

### In scope
- `BP_DDGameMode.SpawnDue`: the loop bound is stored in a local before the loop.
- `BP_Explosion`: one new boolean variable `bPlayerMade`.
- `BP_EnemyMissile`: one new boolean variable `bPlayerKill`; small edits in `IsInsideExplosion` and `Die`.
- `BP_DDGameMode.DamageSlot`: marks the impact explosion as enemy-made.
- `BP_DDPlayerController`: new helpers `CapRoomCase`, `WaveSpeedCase`, `OriginCaseA`, `OriginCaseB`, `OriginCaseC`; new tests `RunT40`, `RunT41`; three small edits in `RunPhase6Tests`.

### Out of scope (do NOT do)
- Any config change, any new key, any HUD change, any new asset.
- Changing `SpawnExplosion`'s inputs or outputs. Changing `BP_ABM`.
- Cleaning up leftovers (list in section 10). Removing the self-test.
- Changing how enemies are killed, how chain reactions work, or what an impact explosion can kill.

---

## 3. Design (decided; do not change)

### 3.1 Fix A: burst size at the cap (bug)
`SpawnDue` (Phase 6, 5a) was meant to spawn `n = Min(Min(WaveBurstSize, WaveToSpawn), room)` enemies, with `room = WaveMaxActive - Length(ActiveEnemies)`, computed **once** at the start of the burst. In the log it behaved as if `n` were recomputed on every loop pass: each spawned enemy lowered `room` and `WaveToSpawn`, which shortened the loop. With room for exactly two, only **one** was spawned. (The cap itself was never exceeded and all missiles of the wave were still sent; only the burst size at the boundary was wrong.)
**Rule:** `room` and `n` are stored in locals on the exec chain **before** the loop; the loop runs `n` times (`0 .. n - 1`) using the local.

### 3.2 Fix B: kill origin
Every explosion has an **origin**, stored in `BP_Explosion.bPlayerMade`:

| Explosion | `bPlayerMade` |
|---|---|
| ABM arrival (`BP_ABM.OnArrived`) | **true** (the default; `BP_ABM` is not touched) |
| Impact of an enemy on a slot (`DamageSlot`) | **false** |
| The explosion a dying enemy leaves behind (`Die`) | **inherits**: true if the enemy died inside at least one player-made explosion, otherwise false |
| Explosions created by tests with no flag set | true (the default) |

An enemy **scores** (`OnEnemyKilled`) only if, at the moment it dies, **at least one player-made explosion contains it**. If it dies only inside enemy-made explosions it is still killed (`EnemiesKilled + 1`, `LastKillPos`, the `Killed` log line, its own chain-reaction explosion) but **no points** are given and `LastKillPoints` is not changed.

Consequences, on purpose:
- A chain reaction **started by your ABM** still scores every kill in the chain (the explosions inherit "player-made").
- A chain **started by a landing** scores nothing, however long it gets.
- An enemy inside both kinds of explosion scores.
- Impact explosions can still kill enemies (Phase 5, unchanged).

### 3.3 Log lines
No new log line. The existing lines keep their format. (A kill without points shows a `[DD][Enemy] Killed ...` line **without** a `[DD][Score] Kill ...` line.)

---

## 4. What this patch changes in earlier specs (for the changelog)

| Earlier text | Now |
|---|---|
| Phase 6, 3.3: "A kill (an enemy destroyed by an explosion; this includes kills by impact explosions and chain reactions) scores ..." | **Replaced by 3.2 above.** Kills by landing explosions (and chains started by them) do not score. |
| Phase 6, 3.6 and 5a (`SpawnDue`): `n` computed once | **Fixed** (3.1): now actually computed once. |
| Phase 6 acceptance B10 (no `[DD][Score]` line in the 28 s window) | **Valid again**: with nobody firing there is no score line, because landing kills no longer score. |
| Phase 6 self-test: `Phase6 tests: n/8 PASS` | **`n/10`**. Total tests: 31 old + 10 = 41. The line `SelfTest done: n/31 PASS` is unchanged. |
| Phase 4 `IsInsideExplosion` and `Die`; Phase 5 `DamageSlot`; Phase 3 `BP_Explosion` | Small additions (Step 1 and Step 5). Nothing removed. |

---

## 5. Tooling notes

1. **Do not guess node IDs.** Use `find_node_types`. Call `get_graph_dsl_docs` once if needed.
2. **A value read by a later node is evaluated when that node runs.** This includes loop bounds: a loop whose last index is a live expression re-reads it on every pass. **Store a value in a local or variable before the loop (or before anything that changes what it depends on).**
3. **A function's returned object** (for example the explosion returned by `SpawnExplosion`) may be wired **directly** into the target pin of the very next nodes in the same exec chain; no local is needed. Do **not** read `GM.LastExplosion` for it later, because it changes with every new explosion. Where a test needs `GM.LastExplosion`, it reads and stores what it needs **in the same step** that created the explosion.
4. **Branches inside a chain:** no nodes after a Branch/return; connect **both branch ends to the same next exec pin** (an exec input accepts several wires). A `Branch` that ends a loop body is fine.
5. **Non-Pure functions need exec wires.** Call each function in sequence, store each result in its own variable or local, then combine.
6. **`Destroy Actor` runs `EndPlay` immediately.** In same-frame tests use `Length(GM.ActiveEnemies)` or `Length(GM.ActiveExplosions)`, not `IsValid`.
7. **Synthetic tests** freeze every enemy and explosion they create (`SetActorTickEnabled(false)`) right after the call. They are cleaned up by the test itself.
8. **A new function cannot return an object** (tool limit seen in Phase 6): helpers below return only booleans.
9. **PIE tools:** `StartPIE`, `StopPIE`, `IsPIERunning`, `GetLogEntries`, `CaptureViewport`, `CaptureEditorImage`.
10. **Do not clean up:** leave unreferenced nodes and events (section 10) where they are.

---

## 6. Implementation steps

### Step 1 — Two new variables (no logic yet)

- `BP_Explosion`: variable **`bPlayerMade`**, boolean, default **`true`**, not Instance Editable.
- `BP_EnemyMissile`: variable **`bPlayerKill`**, boolean, default **`false`**, not Instance Editable.

Read both defaults back.

**Check:** both Blueprints compile clean; the defaults read back as above. **Save All.**

---

### Step 2 — New helpers, tests, and the `RunPhase6Tests` edit (controller)

Only **new** functions (one small write each, in this order) and three small node edits. All non-Pure.

**Conventions for the helpers and tests below**
- `Y = Cfg.PlayPlaneY`; `V = 130.0`; `M, G, H = Cfg.ExplosionMaxRadius, Cfg.ExplosionGrowTime, Cfg.ExplosionHoldTime`; `a = 0.625 * M`. Position tolerance 0.5.
- **Reset R** (start of every helper): `GM.ClearEnemies(false) ; GM.RestoreAll(false) ; GM.ResetGame() ; GM.bDirectorSilent = true` (in this order; `ResetGame` sets `bDirectorSilent` back to false, so it is set to true **after** it).
- **End E** (end of every helper, after the result is stored): destroy **every `BP_Explosion`** (`Get All Actors Of Class(BP_Explosion)` → For Each → `Destroy Actor`; no explosion exists at this point of the self-test except the ones the helper made, and they are frozen) ; `GM.ClearEnemies(false) ; GM.RestoreAll(false) ; GM.ResetGame()`. (Same pattern as T30 uses for `BP_ABM`.)
- Every helper stores its boolean result **in a local before E runs**, then outputs it.
- All tests are silent (`bLog = false`).

**`CapRoomCase`** (output `bOK` boolean) — room for exactly two:
```
R
mx = Cfg.WaveMaxActive ; t0 = 1000.0 ; intro = Max(Cfg.WaveIntroTime, 0.0)
Repeat (mx - 2) times:
    e = GM.SpawnEnemy((0, Y, 1800), GM.GetSlotPos(8), V, 8, false) ; e.SetActorTickEnabled(false)
GM.StartWave(1, t0) ; GM.WaveState = 2
tot = GM.GetWaveCount(1) ; exp = Min(Min(Cfg.WaveBurstSize, tot), 2)           // store both
GM.DirectorStepAt(t0 + intro + 1.0) ; FreezeEnemies
n = Length(GM.ActiveEnemies) ; ts = GM.WaveToSpawn                             // store
bOK = (n == mx - 2 + exp) AND (ts == tot - exp)
E
```
(Assumes `WaveMaxActive >= 3`. With the defaults: 6 enemies prefilled, expected burst 2, so `n == 8`.)

**`WaveSpeedCase`** (output `bOK`) — the enemy speed follows the wave:
```
R
GM.Wave = 2                                                                    // set directly; StartWave is not used here
e = GM.SpawnRandomEnemy(false) → immediately: e.SetActorTickEnabled(false) ; sp = e.Speed      // sp stored in a float local at once
want = GM.GetEnemySpeed(2)                                                     // store
bOK = Abs(sp - want) <= 0.001 AND sp >= Cfg.EnemyBaseSpeed
E
```

**`OriginCaseA`** (output `bOK`) — an enemy killed by an **enemy-made** explosion: no points, origin inherited:
```
R ; GM.Wave = 3 ; GM.Multiplier = 2
sA0 = GM.Score ; kA0 = GM.EnemiesKilled
XA = GM.SpawnExplosion((0, Y, 900), false)
    → immediately, on XA: XA.bPlayerMade = false ; XA.SetActorTickEnabled(false) ; XA.Advance(G + H * 0.5)      // full radius at (0, 900)
EA = GM.SpawnEnemy((a, Y, 900), GM.GetSlotPos(4), V, 4, false) ; EA.SetActorTickEnabled(false)
EA.Advance(0)                                                                  // inside XA: dies
    → immediately: oA = GM.LastExplosion.bPlayerMade (store in a boolean local) ; GM.LastExplosion.SetActorTickEnabled(false)
sA1 = GM.Score ; kA1 = GM.EnemiesKilled ; lkA = GM.LastKillPoints              // store each
bOK = sA1 == sA0 AND kA1 == kA0 + 1 AND lkA == 0 AND NOT oA
E
```

**`OriginCaseB`** (output `bOK`) — an enemy killed by a **player-made** explosion: points, origin inherited:
```
R ; GM.Wave = 3 ; GM.Multiplier = 2
sB0 = GM.Score
XB = GM.SpawnExplosion((0, Y, 900), false)                                     // default flag, nothing set
    → immediately: oBX = XB.bPlayerMade (store) ; XB.SetActorTickEnabled(false) ; XB.Advance(G + H * 0.5)
EB = GM.SpawnEnemy((a, Y, 900), GM.GetSlotPos(4), V, 4, false) ; EB.SetActorTickEnabled(false)
EB.Advance(0)
    → immediately: oB = GM.LastExplosion.bPlayerMade (store) ; GM.LastExplosion.SetActorTickEnabled(false)
sB1 = GM.Score ; lkB = GM.LastKillPoints                                       // store each
bOK = oBX AND sB1 == sB0 + Cfg.ScoreEnemyMissile * 2 AND lkB == Cfg.ScoreEnemyMissile * 2 AND oB
E
```

**`OriginCaseC`** (output `bOK`) — a real impact makes an **enemy-made** explosion:
```
R ; GM.Wave = 3 ; GM.Multiplier = 2
sC0 = GM.Score ; cC0 = GM.CityHits
EC = GM.SpawnEnemy((-800, Y, 1800), GM.GetSlotPos(3), V, 3, false) ; EC.SetActorTickEnabled(false)
EC.Advance(1000.0)                                                             // far longer than the flight: impact on CITY3
    → immediately: oC = GM.LastExplosion.bPlayerMade (store) ; GM.LastExplosion.SetActorTickEnabled(false)
sC1 = GM.Score ; cC1 = GM.CityHits                                             // store each
bOK = NOT oC AND sC1 == sC0 AND cC1 == cC0 + 1
E
```

**`RunT40`** (no inputs) — "burst room and wave speed":
```
GM.ClearEnemies(false) ; GM.RestoreAll(false) ; GM.ResetGame() ; GM.bDirectorSilent = true
c1 = CapRoomCase() ; c2 = WaveSpeedCase()                                       // store each before the next call
ok = c1 AND c2                                                                  // stored in a boolean local
GM.ClearEnemies(false) ; GM.RestoreAll(false) ; GM.ResetGame()
If ok → Test6Pass++
DDLog("T40 burst room and wave speed: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)
```

**`RunT41`** (no inputs) — "kill origin":
```
GM.ClearEnemies(false) ; GM.RestoreAll(false) ; GM.ResetGame() ; GM.bDirectorSilent = true
a1 = OriginCaseA() ; b1 = OriginCaseB() ; c1 = OriginCaseC()                    // store each before the next call
ok = a1 AND b1 AND c1                                                           // stored in a boolean local
GM.ClearEnemies(false) ; GM.RestoreAll(false) ; GM.ResetGame()
If ok → Test6Pass++
DDLog("T41 kill origin: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)
```

**Edit `RunPhase6Tests`** (in place, three small edits; read it first; it currently ends `RunT39() → GM.ClearEnemies(false) ; GM.RestoreAll(false) ; GM.ResetGame() → DDLog("Phase6 tests: " + Test6Pass + "/8 PASS", "Test", Test6Pass != 8)`):
1. Insert calls `RunT40()` and `RunT41()` **after `RunT39()`** and before the cleanup line.
2. Change the number in the comparison from `8` to `10` (`Test6Pass != 10`).
3. Change the text `"/8 PASS"` to `"/10 PASS"`.

Everything else in the function stays.

**Check:** every new function compiles clean; none contains a `Delay`; none calls `RestartGame`, `StartSpawner`, `StopSpawner`, `DirectorLoop` or `DamageSlot` directly. `RunPhase6Tests` calls `RunT32` … `RunT41` in order. **Save All.**

---

### Step 3 — RED run (before any fix)

`StartPIE`. Wait until `[DD][Test] SelfTest done` appears (about 10 s), then `GetLogEntries` (filter `[DD]`) and `StopPIE`. **Expected, exactly:**

| Line | Expected |
|---|---|
| `T1` … `T31` | PASS |
| `T32` … `T39` | PASS |
| `T40 burst room and wave speed` | **FAIL** (case A: only one enemy is spawned) |
| `T41 kill origin` | **FAIL** (cases A and C) |
| `Phase6 tests` | `8/10 PASS` (an error-level line; expected in this run) |
| `SelfTest done` | `31/31 PASS` |

If **anything else** differs (an old test fails, `T40` or `T41` passes, a runtime error appears), **stop and report**; do not continue to the fixes. Record the lines for the report.

---

### Step 4 — Fix A in `SpawnDue`

Read `SpawnDue`. **Edit in place, small node edits only.**
1. After the lines that compute `room` and `n`, **store** `room` in a local `RoomLocal` and `n` in a local `NLocal` (set nodes on the exec chain, before the loop; if `room` and `n` were already stored in locals, find out which pin the loop actually uses).
2. Make the loop's last index read **`NLocal - 1`** (and nothing that depends on `WaveToSpawn` or `Length(ActiveEnemies)`).
3. The two early `return`s and the `NextSpawnTime` line stay as they are. `NextSpawnTime` is set **before** the loop.

Then **check every other new loop of Phase 6** (`RebuildCities`, `TotalStandingAmmo`, `ComputeTally`, `FreezeEnemies`, `PickRandomDestroyedCity`): a loop bound must never depend on something the loop body changes. Report what you found, even if nothing needed changing.

**Check:** `SpawnDue` compiles clean; the loop bound is a stored local. **Save All.**

---

### Step 5 — Fix B (kill origin)

Small node edits in place. Read each function first.

**5a. `BP_EnemyMissile.IsInsideExplosion`.** It currently does: `Loc = GetActorLocation ; Hit = false (local) ; For Each E in GM.ActiveExplosions: If IsValid(E): r = E.ContainsPoint(Loc) ; If r → Hit = true ; return Hit`. Add:
- **before the loop:** `bPlayerKill = false`;
- **inside the loop**, right after `Hit = true` (when an explosion contains the point): `If E.bPlayerMade → bPlayerKill = true`. (Both ends of that branch end the loop body.)

`Hit` is returned exactly as before.

**5b. `BP_EnemyMissile.Die`.** It currently runs: `bActive = false ; P = GetActorLocation ; GM.EnemiesKilled + 1 ; GM.LastKillPos = P ; GM.OnEnemyKilled(bLogEvents) ; GM.SpawnExplosion(P, bLogEvents) ; log ; Destroy Actor`. Change:
- Put a **Branch on `bPlayerKill`** in front of `GM.OnEnemyKilled(bLogEvents)`: True → `OnEnemyKilled` → next; **False → the same next exec pin** (the `SpawnExplosion` call). (Both branch ends go to the same pin.)
- **Directly after** `GM.SpawnExplosion(P, bLogEvents)`, wire the explosion it **returns** into a `Set bPlayerMade` node on that explosion, value = `bPlayerKill`. No `IsValid` check is needed (`SpawnExplosion` always spawns). Do not read `GM.LastExplosion` here.

**5c. `BP_DDGameMode.DamageSlot`.** It calls `SpawnExplosion(P, bLog)` and then `CheckGameOver(bLog)`. **Directly after `SpawnExplosion`**, wire the explosion it returns into a `Set bPlayerMade` node on that explosion, value = **`false`**. Nothing else changes.

**Fallback (5b, 5c):** if the returned object cannot be wired to the next node, use a variable of type `BP_Explosion` reference on the same Blueprint (`NewExplosionRef`), set from the return value right after the call and used by the `Set` node. Record it.

**Check:** the three Blueprints compile clean. `bPlayerMade` is written only in `Die`, `DamageSlot` and the tests; `bPlayerKill` only in `IsInsideExplosion`. `SpawnExplosion` and `BP_ABM` are unchanged. **Save All.**

---

### Step 6 — GREEN run

`StartPIE`. Wait about **28 seconds** in total. Take `CaptureEditorImage` once at about **12 s after `SelfTest done`** (to see that nothing broke; enemies on screen, score block). `GetLogEntries` (filter `[DD]`), then `StopPIE`. Fill in the report (section 8).

---

## 7. Acceptance tests

### A. Static checks
- [ ] **A1** New assets: **none**. No config value changed.
- [ ] **A2** Every changed Blueprint compiles with 0 errors and 0 warnings.
- [ ] **A3** `BP_Explosion.bPlayerMade` (boolean, default true) and `BP_EnemyMissile.bPlayerKill` (boolean, default false) exist.
- [ ] **A4** `SpawnDue`: the loop bound is a stored local, and `NextSpawnTime` is set before the loop. The agent reports its review of the other new loops (Step 4).
- [ ] **A5** `IsInsideExplosion` sets `bPlayerKill` as in 5a. `Die` calls `OnEnemyKilled` only when `bPlayerKill` is true and sets the new explosion's `bPlayerMade` from `bPlayerKill`. `DamageSlot` sets the impact explosion's `bPlayerMade = false`. `SpawnExplosion` and `BP_ABM` are unchanged.
- [ ] **A6** The controller has the new functions `CapRoomCase`, `WaveSpeedCase`, `OriginCaseA`, `OriginCaseB`, `OriginCaseC`, `RunT40`, `RunT41`; `RunPhase6Tests` calls `RunT32` … `RunT41` and uses `10`. **No frozen function was edited** (the agent says so explicitly). The key list is unchanged (`H`, `R`, `E`, `K`, `X`, `V`, `N`).
- [ ] **A7** No actors added to the level. No new `Delay` anywhere.

### B. Log checks (agent)
- [ ] **B1 (red run)** The red-run table of Step 3 matched exactly (lines recorded in the report). If it did not, the agent stopped there.
- [ ] **B2 (green run)** The Phase 0 to 6 startup lines still appear (as in the Phase 6 report), `T1` … `T31` PASS, `[DD][Test] SelfTest done: 31/31 PASS` exactly once.
- [ ] **B3** `T32` … `T39` PASS, **`T40 burst room and wave speed: PASS`**, **`T41 kill origin: PASS`**, **`[DD][Test] Phase6 tests: 10/10 PASS`**, all before `SelfTest done`.
- [ ] **B4** No `[DD][Wave]` or `[DD][Score]` line before `SelfTest done`. The Phase 5 live-segment lines are unchanged (CITY3 spawn and impact, BASE-CENTER spawn and kill, four explosion starts and four ends, one `City destroyed slot=CITY3 left=5`).
- [ ] **B5** After done: `[DD][Enemy] Spawner started interval=2500ms` once; `[DD][Wave] Start n=1 count=12 mult=1 speed=130` once; `[DD][Wave] Running n=1` once, 1.5 to 3.5 s later.
- [ ] **B6** After `Running n=1`: the first **eight** `[DD][Enemy] Spawn` lines form **four pairs** (the two lines of a pair share a timestamp or are less than 100 ms apart; pairs are about 2.5 s apart). The **fourth pair** is the regression check: before this patch the fourth burst was a single enemy.
- [ ] **B7** **No `[DD][Score]` line anywhere** in the window (nobody fires). If a `[DD][Enemy] Killed` line appears (a landing explosion killed a partner), there is **no** `[DD][Score] Kill` line for it.
- [ ] **B8** No `[DD][Wave] Complete` line in the window. `[DD][Damage]` lines after done are allowed in the usual form.
- [ ] **B9** No line containing `ERROR`, `Accessed None`, or `Blueprint Runtime Error` in the green run.

### C. Visual checks
No visual change in this patch. The one capture only has to show the game running (enemies, score block `SCORE 0`, six cities). Mark **PASS** or **NEEDS-HUMAN**.

### D. Interactive checks (human, PIE; click inside the viewport first; wait about 10 s after Play for the self-test)
**New for this patch**
- [ ] **D1** Wave 1 pairs: the first four pairs of enemies come out as **pairs** (not a pair, a pair, a pair, a single).
- [ ] **D2** **Landing kills do not score:** when two enemies of a pair land on the same city, the landing explosion may kill the other one; the log shows `Killed ...` for it and **no** `[DD][Score] Kill` line, and the score does not change.
- [ ] **D3** **Your chain still scores:** shoot an enemy so that its own explosion reaches another enemy; **both** kills show `[DD][Score] Kill +25` (×multiplier).
- [ ] **D4** An enemy that dies inside **both** one of your explosions and a landing explosion scores (hard to stage; note it only if it happens).

**Still open from Phase 6 (never exercised in the log)**
- [ ] **D5** `N` skips the wave (`[DD][Wave] Skipped n=<old> next=<new>`); wave 3 shows `POINTS x2`.
- [ ] **D6** `V` after game over restarts the **same wave** from its banner, **keeps the score** (`[DD][Wave] Restart n=<w>`); a click after the 2-second lockout reloads the level and the score is 0.
- [ ] **D7** The banner, the tally (`WAVE n COMPLETE`, `ROCKETS ...`, `CITIES ...`, `BONUS CITY REBUILT x<k>`) and the score block look right; resize the window (wide, 5:4, nearly square) and everything stays centered.
- [ ] **D8** `K` after the last missile has spawned ends the wave quickly.

### D-DEVICE. Touch (still open since Phase 1.1)
- [ ] **T-1** A tap launches from the nearest base. **T-2** A second finger is ignored. **T-3** One tap, one launch. **T-4** After game over, a tap in the first 2 s does nothing; later it restarts. **T-5** Score, banner and tally are readable on the phone.

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped.

---

## 8. Report format (the agent's final message)

```
## Phase 6.1 report
**Engine version:** x.y.z
**MCP tools used:** (list)

### Assets created or changed
(table: path | type | change | compiled clean)

### Red run (Step 3)
(the [DD][Test] lines from the red run, with timestamps; did it match the expected table?)

### Step 4 review
(what the SpawnDue loop used before; what was changed; result of the review of the other new loops)

### Acceptance results
A1 PASS/FAIL ... through E1 PASS/FAIL/NEEDS-HUMAN/NEEDS-DEVICE

### Green run log excerpt
(all [DD] lines from the green run, with timestamps: every [DD][Test], [DD][Wave], [DD][Score], [DD][Enemy], [DD][Damage] and [DD][Explosion] line)

### Screenshot notes
### Tool problems (errors, retries, editor restarts)
(every tool error or timeout, what was tried, whether the editor stopped responding or restarted; "none" if none)

### Deviations
### Open questions / risks
```

---

## 9. What this hands to later phases

| Provided | How a later phase uses it |
|---|---|
| `BP_Explosion.bPlayerMade` | Phase 8: other weapons (smart bombs, MIRV warheads) decide whether their explosions are player-made or enemy-made; scoring already respects the flag. |
| `BP_EnemyMissile.bPlayerKill`, the gate in `Die` | Phase 8: `OnEnemyKilled` gets a weapon type behind the same gate. |
| `CapRoomCase`, `WaveSpeedCase`, `OriginCase*` | Examples of small test helpers; the self-test is dropped in Phase 7. |

---

## 10. Notes for the human (not agent steps)

1. **Commit order.** Commit the accepted Phase 6 state **before** giving this patch to the agent. Then commit the result of this patch as one commit. Suggested message: `Phase 6.1: fix burst size at the active cap; kills by landing explosions no longer score (explosion origin); tests T40 and T41`.
2. **Files this patch touches (for the diff):** `BP_DDGameMode` (`SpawnDue`, `DamageSlot`), `BP_Explosion` (+1 variable), `BP_EnemyMissile` (+1 variable, `IsInsideExplosion`, `Die`), `BP_DDPlayerController` (new helpers and tests, `RunPhase6Tests`). Nothing in config, HUD, assets or the level.
3. **The red run is deliberate.** It proves the new tests can fail. You will see `FAIL` lines and a `Phase6 tests: 8/10` error line in the agent's first run; that is expected.
4. **Self-test count is now 41** (31 old + 10). The old line still says `31/31`; the Phase 6 line says `n/10`.
5. **Known leftovers, not touched here (for a later cleanup phase):** the dead event `EnemySpawnLoop` (never started); two disconnected spawn chains in `SpawnRandomEnemy` that still read `EnemyBaseSpeed` (the live path uses `GetEnemySpeed`); the unused function `CacheLastExplosion` and the helper `DestroyLastExplosion`; unused string nodes in `RunT36`; `StopSpawner` no longer stops anything (the director loop never stops).
6. **Tuning, not a bug:** in your seven-wave run, wave 7 (speed 496, flight time about 3.6 s) overwhelmed you. If it is too steep, lower `WaveSpeedFactor` or `EnemyMaxSpeed` in `DA_DDConfig`.
7. **Patch specs from now on.** Corrections come as numbered patch specs (`phase-6.1-...`, `phase-6.2-...`), each with a "what this changes in earlier specs" table (section 4) and a suggested commit message, so the repository history shows every change. The agent never gets loose correction messages.
8. **Touch is still untested.**

---

## 11. Common pitfalls

1. **Applying the fixes before the red run.** The red run proves the tests see the bugs. Do Step 3 first.
2. **A loop bound that is a live expression.** Store `n` in a local before the loop (fix A), and check the other loops.
3. **Reading `GM.LastExplosion` later** instead of using the returned explosion (or reading it in the same step). It changes with every new explosion.
4. **Setting `bPlayerMade` to false by default.** The default is **true**; only `DamageSlot` and the inheritance in `Die` produce false. Otherwise ABM kills, `BP_ABM` and the frozen tests would change.
5. **Gating the whole of `Die`.** Only `OnEnemyKilled` is gated by `bPlayerKill`. The counter, `LastKillPos`, the chain-reaction explosion and the log stay for every kill.
6. **Forgetting to reset `bPlayerKill`.** `IsInsideExplosion` sets it to false before the loop, every time.
7. **Editing a frozen function** to "make a test fit". If an old test fails, stop and report.
8. **A test helper that leaves explosions alive.** Every helper ends with E (destroy all `BP_Explosion`, clear enemies, `RestoreAll`, `ResetGame`).
9. **Using `Delay` in a function**, or starting the director from a test. The tests call `DirectorStepAt` by hand.
10. **Treating the `8/10` line and the `FAIL` lines of the red run as problems.** They are the expected result of Step 3.
