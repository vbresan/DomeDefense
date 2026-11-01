# Phase 6 — Waves, Scoring and Bonus Cities

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Depends on** | Phase 5 (agent checks passed 31/31; the human played it with `X` and `V`, game over and restart, and accepted it; touch on a device is still open and is not repeated here) |
| **Phase goal** | The endless random spawner is replaced by **waves**. Each wave has a fixed number of enemy missiles, sent in bursts, and gets faster and worth more points as the waves go on. Every enemy missile you shoot down scores points. When a wave is over (all its enemy missiles are destroyed or have landed), a **tally** pays points for unused ABMs and surviving cities, the bases are refilled, and the next wave starts. Every 10,000 points banks a **bonus city**, which rebuilds one destroyed city at the end of a wave. The HUD shows score, wave, multiplier, banked bonus cities, a wave banner and the tally. A new debug key `N` skips to the next wave. |

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **No bars, no aspect caps.** The whole window is the playfield. Do not add any.
3. **Existing logic must survive** unless a step says to change or delete it. Read each Blueprint's current graphs (`read_graph_dsl`) before editing.
4. **Compile after every Blueprint change.** Every Blueprint ends with no errors and no warnings.
5. **Run each step's "Check"** before moving on. **Save All after each step's Check passes.**
6. **You cannot press keys, click, touch, or resize in PIE.** The self-test checks waves, scoring and bonus cities with **synthetic calls** (a function that takes the time as an input, so nothing is waited for). Interactive checks (group D) are **NEEDS-HUMAN**; touch is **NEEDS-DEVICE**.
7. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
8. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after each test. **Save before loading a level.**
9. **Read sections 4 and 5 before building graphs.**
10. **This phase adds exactly eight config values** (Step 1). Change no other config value.
11. **NEVER REWRITE AN EXISTING FUNCTION OR EVENT GRAPH.** `write_graph_dsl` replaces a graph by deleting all its nodes and rebuilding it. On a big graph this ran the editor out of memory in Phase 5 and the editor crashed. Therefore:
    - **A brand-new function** (it has no nodes yet) may be written with `write_graph_dsl`, in **one small write**. Keep each new function to **about 40 nodes at most**. If a function in this document looks bigger, it is already split into helpers: build the helpers as separate functions. (The 40 is a guess; if a write is slow or fails, build smaller pieces.)
    - **An existing graph** (anything listed as "edit in place") is changed **only** by small node edits: `create_node`, `connect_pins`, set a pin default value, delete a single node. **Never** `write_graph_dsl` on it. This includes the event graphs of `BP_DDGameMode` and `BP_DDPlayerController`: new events there are made with `create_node` and `connect_pins`.
12. **FROZEN FUNCTIONS: do not open them for writing, do not edit them, do not rewrite them.** All in `BP_DDPlayerController`:
    `RunSelfTest`, `RunPhase2Tests`, `RunPhase3Tests`, `RunPhase4Tests`, `RunPhase4Rest` (holds T22–T24), `RunPhase5Tests`, `SelfTestFlightStart`, `SelfTestFlightMid`, `SelfTestFlightEnd`, `SelfTestExplosionEnd`, `SelfTestEnemyStart`, `SelfTestEnemyEnd`.
    The **only** allowed changes in the controller are: new functions, new variables, the new key `N`, and inserting **one** call node (`RunPhase6Tests`) into the `BeginPlay` chain (7g). New tests go into **new functions**.
13. **After any tool error (DSL error, timeout, "not found"), do not retry the same large write.** Stop that item, try one smaller piece at most, and if it still fails, **stop and report** under *Tool problems*. **If the editor stops responding or restarts, stop and report.** Do not continue with a half-built graph.
14. **The old self-test must keep passing exactly as it is**: `[DD][Test] SelfTest done: 31/31 PASS` stays valid. The new tests (T32–T39) are counted in their **own** counter (`Test6Pass`) and have their own summary line (7f).

---

## 2. Scope

### In scope
- Config: eight new values (Step 1).
- `BP_DDGameMode`: score, wave and bonus-city state; the wave director (a time-stepped state machine plus one persistent loop); scoring; tally; rebuilding bases and cities; wave skip.
- `BP_DDGameMode.SpawnRandomEnemy`: speed now comes from the wave (one call inserted).
- `BP_EnemyMissile.Die`: calls the scoring function (one call inserted).
- `BP_DDPlayerController`: debug key `N`, new tests T32–T39 in new functions (total **31 + 8 = 39**), one inserted call in `BeginPlay`.
- `BP_DDHUD`: title line, score block, wave banner, wave tally.

### Out of scope (do NOT create)
- Game flow, menus, pause, high score, saving anything (Phase 7).
- MIRVs, bombers, smart bombs, UFOs, other weapons (Phase 8). `ScoreBomber` and `ScoreSmartBomb` are read but not used.
- Sounds, particles, imported art, any UMG.
- Any key besides `N` (new). Any config value besides the eight.
- Changing game over (3.9), the damage rules (Phase 5) or the nearest-base rule.
- Removing the dead event `EnemySpawnLoop` (it stays, unreferenced; cleanup is a later phase).
- Removing the self-test.

---

## 3. Design (decided; do not change)

### 3.1 Waves in one paragraph
A wave has a **count** of enemy missiles. The wave starts with a short **intro** (banner), then enemies are sent in **bursts** of `WaveBurstSize` every `EnemySpawnInterval` seconds, never more than `WaveMaxActive` on screen at once. When the wave has spawned all its missiles **and** none are left alive (each one was shot down or has landed), the wave **ends**: a **tally** screen pays the bonuses, destroyed bonus-city rebuilds happen, and after `WaveTallyTime` seconds the next wave starts. There is **no other** end condition (a destroyed city does not end a wave; game over is separate).

### 3.2 Per-wave numbers
All derived from config and the wave number `w` (1, 2, 3, …). **Wave 0 means "no wave yet"** (self-test, before the spawner starts).

| Value | Rule | Result with defaults |
|---|---|---|
| **Count** | `n = Length(WaveCounts)`; if `n <= 0` → 12; else `Max(WaveCounts[(Max(w,1) - 1) mod n], 1)` (the list repeats) | waves 1–12: 12, 15, 18, 12, 16, 14, 17, 10, 13, 16, 19, 12; wave 13 = 12 again |
| **Multiplier** | `Min(1 + (Max(w,1) - 1) / 2, Max(MaxWaveMultiplier, 1))` (integer division) | waves 1–12: ×1, ×1, ×2, ×2, ×3, ×3, ×4, ×4, ×5, ×5, ×6, ×6; then ×6 |
| **Enemy speed** | `s = EnemyBaseSpeed * WaveSpeedFactor ^ (Max(w,1) - 1)`; result `Min(s, Max(EnemyMaxSpeed, EnemyBaseSpeed))` | 130, 162.5, 203, 254, 317, 397, 496, then 600 (cap) |

`GetEnemySpeed(0)` returns `EnemyBaseSpeed` (this keeps the frozen test T21 true: it spawns enemies while the wave is 0).

### 3.3 Scoring
- A **kill** (an enemy destroyed by an explosion; this includes kills by impact explosions and chain reactions) scores `ScoreEnemyMissile * Multiplier`. Only while `Wave >= 1`. At wave 0 nothing scores (the self-test).
- An enemy that **lands** scores nothing. Enemies removed by `K` or by game over (`ClearEnemies`) score nothing.
- The **tally** (3.6) adds the wave-end points.
- `AddScore(Points)` ignores `Points <= 0`.
- **Bonus cities:** every time the score passes another multiple of `BonusCityScore` (10,000), one **bonus city** is banked: `earned = Score / BonusCityScore` (integer division); if `earned > BonusEarned` then `BonusCities += earned - BonusEarned` and `BonusEarned = earned`. (Jumping over two multiples in one `AddScore` banks two.)

### 3.4 Bonus cities
- `BonusCities` is a bank of "cities in stock". It is spent at the **end of a wave** (during `EndWave`, after the tally points were added, so a city banked by the tally points can be used at once): for each banked city, one **random destroyed city** is rebuilt (`BP_City.Rebuild`) and the bank goes down by one. If no destroyed city is left, the rest of the bank **stays banked**.
- **Cities are rebuilt only this way** (and by the debug key `V`). Bases are rebuilt every wave start (3.5).
- **A banked bonus city does not prevent game over**: when the last city falls, the game is over at once (Phase 5 rule, unchanged).

### 3.5 Wave life cycle (the "director")
`WaveState` values: **0 Idle** (no wave yet), **1 Intro**, **2 Running**, **3 Tally**, **4 Halted** (game over).

| State | Meaning | Leaves the state when |
|---|---|---|
| 0 Idle | before the spawner started; the self-test | `StartSpawner` calls `StartWave(1, now)` |
| 1 Intro | banner on screen, nothing spawns | `Now >= StateEndTime` (`WaveIntroTime` after the start) → 2 |
| 2 Running | bursts spawn; ends the wave when everything is done | `WaveToSpawn == 0` and no enemy alive → `EndWave` → 3 |
| 3 Tally | tally on screen | `Now >= StateEndTime` (`WaveTallyTime` after the end) → `StartWave(Wave + 1)` → 1 |
| 4 Halted | game over | `bGameOver` becomes false again (key `V`) → `StartWave(Wave)` (the **same** wave, from the intro) → 1 |

- **`StartWave(n, Now)`** sets the wave, count, multiplier, **rebuilds every base** (full ammo, destroyed bases come back), sets state 1.
- **Game over** (Phase 5) is unchanged: it clears every enemy and sets `bGameOver`. The director sees `bGameOver` at its next step and goes to state 4. After `V` (which clears `bGameOver`) the same wave restarts from its intro; **score and banked bonus cities are kept**. A click or tap after the lockout still **reloads the level** (everything, including the score, resets).
- **One director loop**, started by `StartSpawner`, runs forever: `Delay(0.1)` → `DirectorStepAt(Get Game Time In Seconds)` → repeat. It is never stopped and never started twice (`StartSpawner` is guarded by `bSpawnerRunning` and is called from exactly two places, as before).
- **All logic is in `DirectorStepAt(Now)`**, which takes the time as an input. The self-test calls it with made-up times (for example 1000.0, 1002.0). This is how waves are tested without waiting.

### 3.6 Spawning in a wave (`SpawnDue`)
Runs in state 2 on every step:
- Nothing if `WaveToSpawn <= 0` or `Now < NextSpawnTime`.
- `room = WaveMaxActive - Length(ActiveEnemies)`; `n = Min(Min(WaveBurstSize, WaveToSpawn), room)`; nothing if `n <= 0` (and `NextSpawnTime` is **not** changed then, so the burst goes out as soon as there is room).
- Otherwise `NextSpawnTime = Now + Max(EnemySpawnInterval, 0.2)` and `n` enemies are spawned with `SpawnRandomEnemy` (random standing target, speed of the current wave). `WaveToSpawn` goes down once per enemy that was really created.
- The first burst is allowed as soon as the intro is over (`NextSpawnTime` starts equal to the end of the intro).
- `bEnemyAutoSpawn = false` stops everything: `StartSpawner` does nothing, so there is no director, no waves and no score (as in Phase 4).

### 3.7 Wave end and tally (`EndWave`)
Runs once, when the wave is done:
1. State 3; `StateEndTime = Now + Max(WaveTallyTime, 0)`; `WavesCompleted + 1`.
2. `ammo` = the sum of `Ammo` of all **standing** bases; `cities` = standing cities (counted **before** any rebuild).
3. `ammoPts = ammo * ScoreUnusedAbm * Multiplier`; `cityPts = cities * ScoreSurvivingCity * Multiplier`. Stored for the HUD (`LastTally*`).
4. `AddScore(ammoPts + cityPts)` (may bank a bonus city).
5. Rebuild banked cities (3.4); `LastBonusRebuilt` = how many.
6. Log the line `Complete ...`.

### 3.8 Skipping (debug key `N`)
`N` → `GM.SkipWave()`: does nothing if `bGameOver` or `Wave < 1`; otherwise removes every enemy **without** explosions and score (`ClearEnemies(false)`) and starts the **next** wave at once (`StartWave(Wave + 1, now)`). No tally, no bonus, the score stays.

### 3.9 Debug keys (desktop only; they do not exist on Android)
Existing: `H` (overlay), `R` (refill), `E` (spawn enemy), `K` (clear enemies), `X` (destroy a city), `V` (restore all). **New: `N`.** No other keys.
Notes: `K` after the last missile of a wave has spawned ends the wave at once (no points for those enemies). `E` spawns a normal enemy at the current wave's speed; it counts as an active enemy for the wave end.

### 3.10 HUD
- Line 1 (title): `DOME DEFENSE | PHASE 6 | WAVES AND SCORE`. Lines 2 to 4 unchanged. The GAME OVER block unchanged.
- **Score block**, top centre, only while `Wave >= 1` (also during game over):
  - `SCORE <n>`: yellow `(1.0, 0.9, 0.2)`, scale 2.5, `Y = 15`.
  - `WAVE <n>   x<mult>`: white, scale 1.6, `Y = 55`.
  - `BONUS CITIES <n>`: green `(0.3, 1.0, 0.4)`, scale 1.6, `Y = 85`, only while `BonusCities > 0`.
- **Intro banner**, while `WaveState == 1` and not game over: `WAVE <n>` yellow, scale 5, `Y = SizeY * 0.30`; below it (`Y = SizeY * 0.30 + 90`) `POINTS x<mult>` white, scale 2.5.
- **Tally**, while `WaveState == 3` and not game over (`Y0 = SizeY * 0.25`):
  - `WAVE <n> COMPLETE` yellow, scale 4, `Y0`.
  - `ROCKETS <a>   +<ammoPts>` white, scale 2.5, `Y0 + 70`.
  - `CITIES <c>   +<cityPts>` white, scale 2.5, `Y0 + 115`.
  - `BONUS CITY REBUILT x<k>` green, scale 2.5, `Y0 + 160`, only when `k > 0`.
- Everything is centered horizontally with `Get Text Size`. All by canvas calls (no UMG).

### 3.11 Layering (unchanged from Phase 5)
Ground 0; bases, ammo numbers and cities +60; enemies +70; ABMs +80; explosions +90; crosshair +100; tap marker +110.

### 3.12 Log lines
Only real events log. The self-test sets `bDirectorSilent = true`, so wave and score events made by tests stay silent; test calls also pass `bLog = false`.

| Category | Line |
|---|---|
| `Wave` | `[DD][Wave] Start n=<w> count=<c> mult=<m> speed=<int>` (`speed` = rounded `GetEnemySpeed(w)`) |
| `Wave` | `[DD][Wave] Running n=<w>` (the intro is over) |
| `Wave` | `[DD][Wave] City rebuilt slot=<NAME>` (one per rebuilt city) |
| `Wave` | `[DD][Wave] Complete n=<w> ammo=<a> ammoPts=<p> cities=<c> cityPts=<p> rebuilt=<k> score=<s>` |
| `Wave` | `[DD][Wave] Skipped n=<old> next=<new>` (key `N`) |
| `Wave` | `[DD][Wave] Restart n=<w>` (after `V` following game over) |
| `Score` | `[DD][Score] Kill +<pts> score=<n>` |
| `Score` | `[DD][Score] Bonus city banked bank=<n> score=<n>` |

All Phase 0–5 lines stay as they are.

---

## 4. Conventions as built (these replace earlier text where they differ)

| Topic | As built |
|---|---|
| Camera | **Ortho Width is the vertical extent** (visible height). Do not change `ApplyArea`. |
| Visible area | `BP_DDGameMode` holds `VisibleMinX/MaxX/MinZ/MaxZ`. |
| Reading values | A value read in a later node is evaluated **when that node runs**. **Store a value in a variable (or a local) before you change what it depends on.** This also applies to **random nodes**: every read of `Random ... In Range` rolls a new number. |
| Config | Config functions and `GetConfig` are **not Pure**. The game mode uses its cached `Config`; the controller uses its cached `Cfg`. |
| Logging | Only through the static `DDLog(Message, Category, bIsError)`. Coordinates in logs are rounded integers. |
| Ticking | A Blueprint actor needs **Can Ever Tick = true** and **Start With Tick Enabled = true** for its Event Tick to run. |
| Self-test | `bSelfTest` (default true) starts a chain from `BeginPlay`. Phase 5's chain: `RunSelfTest` → `RunPhase3Tests` → `RunPhase4Tests` → `RunPhase5Tests` → `SelfTestFlightStart` → `Delay(0.5)` → `SelfTestFlightMid` → `Delay(2.5)` → `SelfTestFlightEnd` → `Delay(2.0)` → `SelfTestExplosionEnd` → `SelfTestEnemyStart` → `Delay(3.5)` → `SelfTestEnemyEnd`. `SelfTestEnemyEnd` prints `SelfTest done: n/31 PASS` and calls `GM.StartSpawner` last. |
| Test counters | `TestPassCount` counts T1–T31 (frozen). **`Test6Pass`** (new) counts T32–T39. |
| Test variable names | Older test variables keep their names. **Leave them.** New test variables use the prefix `Test`. |
| Synthetic tests | They call `Advance` or `DirectorStepAt` by hand and use actors whose tick is disabled (`SetActorTickEnabled(false)`). **Every enemy and every explosion a test creates is frozen and destroyed (or cleared) by the test itself.** An explosion destroyed from outside does not count as finished, so `ExplosionsFinished` is untouched. |
| Hooks | Comment boxes cannot be created. A "hook" is described by the node **after which** the new call goes. |
| Text of the HUD title | One space on each side of every pipe character. |
| Slow editor | "max tick rate 3" appears when the editor window is not focused. Do not tighten the tolerances given below. |
| Spawner | `StartSpawner` and `EnemySpawnLoop` are custom events in the `BP_DDGameMode` event graph. `StartSpawner` is called from exactly two places (`SelfTestEnemyEnd` and the `bSelfTest = false` branch of `BeginPlay`). **Phase 6 adds no third.** |
| Phase 5 state | `Cities`, `bGameOver`, `GameOverTime`, `GameOverCount`, `CityHits`, `BaseHits`, `LastDamageSlot`; functions `SlotToBaseIndex`, `SlotToCityIndex`, `CityIndexToSlot`, `IsSlotAlive`, `CountAliveCities`, `CountAliveBases`, `DamageSlot`, `CheckGameOver`, `GameOver`, `IsRestartAllowed`, `RestoreAll`, `DestroyRandomCity`, `SpawnCities`. `BP_City` and `BP_MissileBase` have `bDestroyed`, `Demolish`, `Rebuild` (`BP_MissileBase.Rebuild(StartAmmo)`). |

---

## 5. Tooling notes

1. **Do not guess node IDs.** Use `find_node_types`. Call `get_graph_dsl_docs` once.
2. **Key events cannot be written as DSL.** Use `create_node` + `connect_pins`, **Pressed** pin only. Known ID pattern: `Input|KeyboardEvents|H`. Find the one for `N` with `find_node_types`.
3. **Latent nodes (`Delay`) only work in event graphs**, not in functions. Nothing in the controller adds a `Delay`. The only new `Delay` is the one in `DirectorLoop` (a custom event in the game-mode event graph, built with `create_node` + `connect_pins`).
4. **Non-Pure functions need exec wires.** Call each function in sequence, store each result in its own variable or local, then combine the variables.
5. **Sequence pins** are `then_0`, `then_1`. **No nodes after a Branch/return** in the same chain. Where the pseudo-code shows an `If` followed by more steps, either put the `If` inside a small helper function, or connect **both branch ends to the same next exec pin** (an exec input accepts several wires). A `Branch` that ends a loop body is fine.
6. **Spawn Actor From Class:** Collision Handling Override = **Always Spawn**. After spawning, call the actor's `Init...` function in the same exec chain.
7. **`Destroy Actor` runs `EndPlay` immediately.** In same-frame tests do **not** use `IsValid` on a just-destroyed actor; use `Length(GM.ActiveEnemies)` or `Length(GM.ActiveExplosions)`.
8. **`Set Actor Tick Enabled(false)`** keeps an actor still. A newly spawned explosion or enemy has not ticked yet when the spawning function returns, so freeze it **immediately** after the call. (The helper `FreezeEnemies` does this for every enemy in `GM.ActiveEnemies`.)
9. **Loops over an array that can change:** loop over a **copy**, or collect first and act afterwards.
10. **Integer remainder** is the node `Percent` (integer). **Integer division** is the `/` (integer) node (it truncates). `Power` takes floats.
11. **Variable defaults:** set the default on the variable **and** read it back. New variables are **not** Instance Editable unless a step says so.
12. **Properties not settable through ObjectTools:** `bIsPureFunc`, `BlueprintType`, `ActorLabel`. The variable-rename tool does not update references: do not rename anything.
13. **PIE tools:** `StartPIE`, `StopPIE`, `IsPIERunning`, `GetLogEntries`, `CaptureViewport`, `CaptureEditorImage`. Use `CaptureEditorImage` for the play view.
14. **`Switch on Int`** may be used for the director switch; four chained `Branch` nodes (each False pin to the next) are equally fine.
15. **Do not clean up:** leave unreferenced nodes and events (such as `EnemySpawnLoop`) where they are.

---

## 6. Implementation steps

### Step 1 — Config

Add to `PDA_DDConfig` (Instance Editable, category `Waves`):

| Variable | Type | Default |
|---|---|---|
| `WaveCounts` | array of integer | `[12, 15, 18, 12, 16, 14, 17, 10, 13, 16, 19, 12]` |
| `WaveBurstSize` | integer | `2` |
| `WaveMaxActive` | integer | `8` |
| `WaveSpeedFactor` | float | `1.25` |
| `EnemyMaxSpeed` | float | `600.0` |
| `WaveIntroTime` | float | `2.0` |
| `WaveTallyTime` | float | `3.0` |
| `BonusCityScore` | integer | `10000` |

Read all of them back on `DA_DDConfig`; set them on the instance if they do not show the defaults (the array must show all twelve numbers). **Also read back and record** these existing values: `ScoreEnemyMissile` (expected 25), `ScoreBomber` (100), `ScoreSmartBomb` (125), `ScoreUnusedAbm` (5), `ScoreSurvivingCity` (100), `MaxWaveMultiplier` (6), `EnemyBaseSpeed` (130), `EnemySpawnInterval` (2.5), `bEnemyAutoSpawn` (true), `AbmStartAmmo` (10). If one of the **scoring values or `MaxWaveMultiplier`** does not exist in `PDA_DDConfig` (they came from Phase 0), add it with the value shown, Instance Editable, category `Scoring`, and record it as a deviation. **Do not change any other value.**

`EnemySpawnInterval` now means **the time between bursts**. `bEnemyAutoSpawn` stays the master switch.

**Check:** the eight values read back as in the table; the ten recorded values are listed; `PDA_DDConfig` compiles clean. **Save All.**

---

### Step 2 — `BP_DDGameMode`: new variables

Read the existing graphs first. Keep everything. Add (not Instance Editable):

| Variable | Type | Default |
|---|---|---|
| `Score` | integer | 0 |
| `BonusCities` | integer | 0 |
| `BonusEarned` | integer | 0 |
| `Wave` | integer | 0 |
| `WaveState` | integer | 0 |
| `WaveTotal` | integer | 0 |
| `WaveToSpawn` | integer | 0 |
| `Multiplier` | integer | 1 |
| `StateEndTime` | float | 0 |
| `NextSpawnTime` | float | 0 |
| `bDirectorSilent` | boolean | false |
| `LastKillPoints` | integer | 0 |
| `LastTallyAmmo` | integer | 0 |
| `LastTallyAmmoPts` | integer | 0 |
| `LastTallyCities` | integer | 0 |
| `LastTallyCityPts` | integer | 0 |
| `LastBonusRebuilt` | integer | 0 |
| `WavesCompleted` | integer | 0 |

Compile; read every default back.

**Check:** compiles clean; 18 variables exist with these defaults. **Save All.**

---

### Step 3 — `BP_DDGameMode`: tables, score and reset functions

**Build the functions in the order listed** (a function can only call functions that already exist). Each is a **new** function: write it in one small write. All are **non-Pure**.

**`GetWaveCount`** (input `WaveNumber` integer; output `Count` integer):
```
n = Length(Config.WaveCounts)                                     // store
If n <= 0 → return 12
w = Max(WaveNumber, 1)                                            // store
i = (w - 1) Percent n                                             // store
return Max(Config.WaveCounts[i], 1)
```

**`GetMultiplier`** (input `WaveNumber`; output `Mult` integer):
```
w = Max(WaveNumber, 1)                                            // store
m = 1 + (w - 1) / 2                                               // integer division; store
return Min(m, Max(Config.MaxWaveMultiplier, 1))
```

**`GetEnemySpeed`** (input `WaveNumber`; output `Speed` float):
```
w = Max(WaveNumber, 1)                                            // store
s = Config.EnemyBaseSpeed * Power(Config.WaveSpeedFactor, w - 1)  // store (w - 1 converted to float)
return Min(s, Max(Config.EnemyMaxSpeed, Config.EnemyBaseSpeed))
```

**`AddScore`** (input `Points` integer):
```
If Points <= 0 → return
Score = Score + Points
If Config.BonusCityScore <= 0 → return
earned = Score / Config.BonusCityScore                            // integer division; store
If earned > BonusEarned:
    BonusCities = BonusCities + (earned - BonusEarned)
    BonusEarned = earned
    If NOT bDirectorSilent → DDLog("Bonus city banked bank=" + BonusCities + " score=" + Score, "Score", false)
```
(Both branch ends and both early `return`s end the function; the nested `If` ends it, nothing follows.)

**`OnEnemyKilled`** (input `bLog` boolean):
```
If Wave < 1 → return                                              // the self-test and the time before the first wave: no points
pts = Config.ScoreEnemyMissile * Multiplier                       // store
LastKillPoints = pts
AddScore(pts)
If bLog → DDLog("Kill +" + pts + " score=" + Score, "Score", false)
```

**`ResetTally`** (no inputs): `LastKillPoints = 0 ; LastTallyAmmo = 0 ; LastTallyAmmoPts = 0 ; LastTallyCities = 0 ; LastTallyCityPts = 0 ; LastBonusRebuilt = 0`.

**`ResetGame`** (no inputs):
```
Score = 0 ; BonusCities = 0 ; BonusEarned = 0 ; Wave = 0 ; WaveState = 0 ; WaveTotal = 0 ; WaveToSpawn = 0
Multiplier = 1 ; StateEndTime = 0 ; NextSpawnTime = 0 ; WavesCompleted = 0 ; bDirectorSilent = false
ResetTally()
```
(`ResetGame` does **not** touch enemies, explosions, cities, bases, `bGameOver`, or any Phase 4/5 counter.)

**Check:** all seven functions compile clean (`GetWaveCount`, `GetMultiplier`, `GetEnemySpeed`, `AddScore`, `OnEnemyKilled`, `ResetTally`, `ResetGame`). **Save All.**

---

### Step 4 — `BP_DDGameMode`: rebuild helpers

New functions, one small write each, in this order (non-Pure):

**`TotalStandingAmmo`** (output `Total` integer): local `t = 0`; for each index `i` of `Bases` → `B = Bases[i]` ; if `IsValid(B)` AND NOT `B.bDestroyed` → `t = t + B.Ammo`. Return `t`.

**`RebuildBases`** (no inputs): for each index `i` of `Bases` → `B = Bases[i]` ; if `IsValid(B)` → `B.Rebuild(Config.AbmStartAmmo)`.

**`PickRandomDestroyedCity`** (output `Index` integer):
```
Dead = local array of integers (empty)
For each index i of Cities: C = Cities[i] ; If IsValid(C) AND C.bDestroyed → Dead.Add(i)
n = Length(Dead)                                                  // store
If n == 0 → return -1
k = Random Integer In Range(0, n - 1)                             // store in a variable FIRST
return Dead[k]
```
**Fallback:** if a local array cannot be created, use a game-mode variable `DeadScratch` (array of integer, not Instance Editable), cleared at the start. Record it.

**`RebuildCities`** (output `Count` integer):
```
k0 = BonusCities ; done = 0                                       // two locals, stored first
For i = 0 to k0 - 1:                                              // no iterations when k0 <= 0
    idx = PickRandomDestroyedCity()                               // store
    If idx >= 0:
        Cities[idx].Rebuild() ; done = done + 1
        sl = CityIndexToSlot(idx)                                 // store
        If NOT bDirectorSilent → DDLog("City rebuilt slot=" + GetSlotName(sl), "Wave", false)
BonusCities = BonusCities - done                                  // after the loop (Completed pin)
return done
```

**Check:** all four compile clean. **Save All.**

---

### Step 5 — `BP_DDGameMode`: the director

#### 5a. New functions (one small write each, in this order, non-Pure)

**`LogWaveStart`** (no inputs):
```
If bDirectorSilent → return
sp = GetEnemySpeed(Wave)                                          // store
DDLog("Start n=" + Wave + " count=" + WaveTotal + " mult=" + Multiplier + " speed=" + Round(sp), "Wave", false)
```

**`StartWave`** (inputs `WaveNumber` integer, `Now` float):
```
Wave = Max(WaveNumber, 1)
WaveTotal = GetWaveCount(Wave)                                    // store the result in a local first, then set
WaveToSpawn = WaveTotal
Multiplier = GetMultiplier(Wave)
RebuildBases()
WaveState = 1
StateEndTime = Now + Max(Config.WaveIntroTime, 0)
NextSpawnTime = StateEndTime
LogWaveStart()
```
(`StartWave` does not clear enemies and does not touch `Score` or `BonusCities`.)

**`SpawnDue`** (input `Now` float):
```
If WaveToSpawn <= 0 → return
If Now < NextSpawnTime → return
room = Config.WaveMaxActive - Length(ActiveEnemies)               // store
n = Min(Min(Config.WaveBurstSize, WaveToSpawn), room)             // store
If n <= 0 → return
NextSpawnTime = Now + Max(Config.EnemySpawnInterval, 0.2)
For i = 0 to n - 1:
    e = SpawnRandomEnemy(NOT bDirectorSilent)                     // store
    If IsValid(e) → WaveToSpawn = WaveToSpawn - 1
```
(Three early `return`s, one loop; the loop body ends in a `Branch`.)

**`ComputeTally`** (no inputs):
```
ammo = TotalStandingAmmo() ; cities = CountAliveCities()          // two locals, stored in order
LastTallyAmmo = ammo ; LastTallyCities = cities
LastTallyAmmoPts = ammo * Config.ScoreUnusedAbm * Multiplier
LastTallyCityPts = cities * Config.ScoreSurvivingCity * Multiplier
```

**`LogWaveComplete`** (no inputs):
```
If bDirectorSilent → return
DDLog("Complete n=" + Wave + " ammo=" + LastTallyAmmo + " ammoPts=" + LastTallyAmmoPts + " cities=" + LastTallyCities
      + " cityPts=" + LastTallyCityPts + " rebuilt=" + LastBonusRebuilt + " score=" + Score, "Wave", false)
```

**`EndWave`** (input `Now` float):
```
WaveState = 3
StateEndTime = Now + Max(Config.WaveTallyTime, 0)
WavesCompleted = WavesCompleted + 1
ComputeTally()                                                    // BEFORE any rebuild
AddScore(LastTallyAmmoPts + LastTallyCityPts)
k = RebuildCities()                                               // store
LastBonusRebuilt = k
LogWaveComplete()
```

**`CheckWaveEnd`** (input `Now` float):
```
If WaveToSpawn > 0 → return
If Length(ActiveEnemies) > 0 → return
EndWave(Now)
```

**`StepIntro`** (input `Now`): `If Now >= StateEndTime → WaveState = 2 ; If NOT bDirectorSilent → DDLog("Running n=" + Wave, "Wave", false)`.

**`StepRunning`** (input `Now`): `SpawnDue(Now)` then `CheckWaveEnd(Now)`.

**`StepTally`** (input `Now`): `If Now >= StateEndTime → StartWave(Wave + 1, Now)`.

**`StepHalted`** (input `Now`): `If NOT bGameOver → StartWave(Wave, Now) ; If NOT bDirectorSilent → DDLog("Restart n=" + Wave, "Wave", false)`. (Both statements sit on the True branch of the same `If`; nothing follows the branch.)

**`DirectorStepAt`** (input `Now` float):
```
If bGameOver:
    If WaveState != 0 → WaveState = 4
    return
Switch on WaveState:
    1 → StepIntro(Now)
    2 → StepRunning(Now)
    3 → StepTally(Now)
    4 → StepHalted(Now)
    (0 and anything else → nothing)
```

**`SkipWave`** (no inputs):
```
If bGameOver → return
If Wave < 1 → return
ClearEnemies(false)
old = Wave                                                        // store
t = Get Game Time In Seconds                                      // store
StartWave(old + 1, t)
If NOT bDirectorSilent → DDLog("Skipped n=" + old + " next=" + Wave, "Wave", false)
```

**Check:** each function compiles clean after it is written. `StartWave`, `EndWave` and `SkipWave` are called from nowhere else yet. **Save All.**

#### 5b. The loop and the `StartSpawner` edit (event graph; **small node edits only**)

**New custom event `DirectorLoop`** (in the event graph of `BP_DDGameMode`; use `create_node` + `connect_pins`, about four nodes):
```
Custom Event DirectorLoop → Delay(0.1) → DirectorStepAt(Now = Get Game Time In Seconds) → DirectorLoop
```
(The last node is a call to the same custom event, as `EnemySpawnLoop` does.) There is **no** "is the spawner running" check: this loop never stops.

**Edit `StartSpawner` in place.** Read it first; it is:
```
If Config.bEnemyAutoSpawn AND NOT bSpawnerRunning:
    bSpawnerRunning = true
    DDLog("Spawner started interval=...ms", "Enemy", false)
    EnemySpawnLoop
```
Change **only** the part after the `DDLog`:
```
    bSpawnerRunning = true
    DDLog("Spawner started interval=...ms", "Enemy", false)       // existing, unchanged
    ResetGame()                                                   // NEW
    StartWave(1, Get Game Time In Seconds)                        // NEW
    DirectorLoop                                                  // NEW; replaces the EnemySpawnLoop call
```
Delete **only** the call node `EnemySpawnLoop` inside `StartSpawner`. **Do not delete or change the `EnemySpawnLoop` event itself** (it stays, unreferenced). Do not touch `StopSpawner`.

**Check:** compiles clean. `DirectorLoop` is called from `StartSpawner` and from itself only. `EnemySpawnLoop` is called from nowhere. `StartSpawner` is still called from exactly two places. **Save All.**

---

### Step 6 — Two hooks (small edits in place)

#### 6a. `SpawnRandomEnemy` (edit in place)
Read it. It currently ends with:
`... Tg = GetSlotPos(Slot) → SpawnEnemy(Start, Tg, Config.EnemyBaseSpeed, Slot, bLog) → return`.
**Insert** a call `GetEnemySpeed(Wave)` between the `GetSlotPos` call and the `SpawnEnemy` call (break that one exec wire; the new call goes in between). **Store** the output in a local `Spd`. Wire `Spd` to the **Speed** pin of `SpawnEnemy`, **replacing** the link from `Config.EnemyBaseSpeed`. Nothing else changes.

At wave 0 `GetEnemySpeed` returns `EnemyBaseSpeed`, so every earlier test is unaffected.

#### 6b. `BP_EnemyMissile.Die` (edit in place)
Read it. It runs: `bActive = false ; P = GetActorLocation ; GM.EnemiesKilled + 1 ; GM.LastKillPos = P ; GM.SpawnExplosion(P, bLogEvents) ; log ; Destroy Actor`.
**Insert** `GM.OnEnemyKilled(bLogEvents)` **right after** `GM.LastKillPos = P` and before `GM.SpawnExplosion`. (Break that one exec wire; the new call goes in between.) `Impact`, `Advance` and everything else stay.

At wave 0 `OnEnemyKilled` returns at once, so every earlier test is unaffected.

**Check:** both compile clean. `OnEnemyKilled` is called only from `Die` (and from the new tests' paths through `Die`). **Save All.**

---

### Step 7 — `BP_DDPlayerController`

Read the existing graphs first. **Frozen functions (rules 11 and 12) are not opened for writing.**

#### 7a. Debug key
With `create_node` + `connect_pins` (Pressed pin only): **`N`** → `GM.SkipWave()`. `H`, `R`, `E`, `K`, `X`, `V` stay. No other keys.

#### 7b. New variable
`Test6Pass` (integer, 0; not Instance Editable).

#### 7c. New helper `FreezeEnemies` (non-Pure, no inputs)
```
Copy = copy of GM.ActiveEnemies
For each E in Copy: If IsValid(E) → E.SetActorTickEnabled(false)
```

#### 7d. Test conventions (read before 7e)
- Sample values are **function locals**; store every result in its own local at the point shown, before running the next line. `Y = Cfg.PlayPlaneY`; `V = 130.0` (a literal test speed); `M, G, H` are `Cfg.ExplosionMaxRadius`, `Cfg.ExplosionGrowTime`, `Cfg.ExplosionHoldTime`. Position tolerances 0.5 units. Synthetic times (`t0 = 1000.0` and so on) are plain numbers; they have nothing to do with the real game clock.
- Each test function starts with the **preamble P** and ends with the **closing C**:
  ```
  P: GM.ClearEnemies(false) ; GM.RestoreAll(false) ; GM.ResetGame() ; GM.bDirectorSilent = true      // in this order
  C: GM.ClearEnemies(false) ; GM.RestoreAll(false) ; GM.ResetGame()
  ```
  (`ResetGame` sets `bDirectorSilent` back to `false`, so `P` sets it to `true` **after** `ResetGame`.)
- **Every enemy and explosion a test creates is frozen right after the call** (`SetActorTickEnabled(false)` or `FreezeEnemies`) and removed by `C` (enemies) or destroyed by the test (explosions).
- The tests assume the config values are sane: `WaveIntroTime > 0`, `WaveTallyTime > 0`, `WaveBurstSize >= 1`, `WaveMaxActive >= 2`, `BonusCityScore >= 2`, `WaveSpeedFactor >= 1`, `MaxWaveMultiplier <= 20`. All hold with the defaults.
- Each test ends with: `If ok → Test6Pass++` and `DDLog("Tnn name: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)`.
- All tests are silent (`bLog = false`, `bDirectorSilent = true`).

#### 7e. New test functions (each non-Pure, no inputs; build them one at a time, each in its own small write)

**`RunT32` — wave tables**
```
P
n = Length(Cfg.WaveCounts) ; cap = Max(Cfg.MaxWaveMultiplier, 1)
cntOK = true
For w = 1 to 13:
    c = GM.GetWaveCount(w)
    If n > 0 → e = Max(Cfg.WaveCounts[(w - 1) Percent n], 1) Else e = 12
    cntOK = cntOK AND c == e
wrapOK = (n <= 0) OR (GM.GetWaveCount(n + 1) == GM.GetWaveCount(1))
zeroOK = GM.GetWaveCount(0) == GM.GetWaveCount(1)
tbl = [1, 1, 2, 2, 3, 3, 4, 4, 5, 5, 6, 6]                         // a literal array
mulOK = true
For w = 1 to 12: m = GM.GetMultiplier(w) ; mulOK = mulOK AND m == Min(tbl[w - 1], cap)
m0 = GM.GetMultiplier(0) ; m40 = GM.GetMultiplier(40)
edgeOK = m0 == 1 AND m40 == Min(20, cap)
base = Cfg.EnemyBaseSpeed ; capS = Max(Cfg.EnemyMaxSpeed, base)
s0 = GM.GetEnemySpeed(0) ; s1 = GM.GetEnemySpeed(1) ; s2 = GM.GetEnemySpeed(2) ; s50 = GM.GetEnemySpeed(50)
spOK = |s0 - base| <= 0.001 AND |s1 - base| <= 0.001
       AND |s2 - Min(base * Cfg.WaveSpeedFactor, capS)| <= 0.01
       AND s50 <= capS + 0.01 AND s50 >= s2 - 0.01
ok = cntOK AND wrapOK AND zeroOK AND mulOK AND edgeOK AND spOK
C
If ok → Test6Pass++
DDLog("T32 wave tables: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)
```

**`RunT33` — scoring and bonus banking**
```
P
B = Cfg.BonusCityScore
// wave 0: nothing scores
GM.OnEnemyKilled(false) ; a0 = GM.Score ; lk0 = GM.LastKillPoints
// wave 3 with multiplier 2 (set directly; StartWave is not used here)
GM.Wave = 3 ; GM.Multiplier = 2
GM.OnEnemyKilled(false) ; a1 = GM.Score ; lk1 = GM.LastKillPoints ; bk1 = GM.BonusCities
// bonus banking from a clean slate
GM.Score = 0 ; GM.BonusEarned = 0 ; GM.BonusCities = 0
GM.AddScore(B - 1)    ; s2 = GM.Score ; bk2 = GM.BonusCities
GM.AddScore(1)        ; s3 = GM.Score ; bk3 = GM.BonusCities ; be3 = GM.BonusEarned
GM.AddScore(2 * B)    ; s4 = GM.Score ; bk4 = GM.BonusCities ; be4 = GM.BonusEarned
GM.AddScore(0) ; GM.AddScore(-5) ; s5 = GM.Score ; bk5 = GM.BonusCities
GM.AddScore(B / 2)    ; s6 = GM.Score ; bk6 = GM.BonusCities
ok = a0 == 0 AND lk0 == 0
     AND a1 == Cfg.ScoreEnemyMissile * 2 AND lk1 == Cfg.ScoreEnemyMissile * 2 AND bk1 == 0
     AND s2 == B - 1 AND bk2 == 0
     AND s3 == B AND bk3 == 1 AND be3 == 1
     AND s4 == 3 * B AND bk4 == 3 AND be4 == 3
     AND s5 == 3 * B AND bk5 == 3
     AND s6 == 3 * B + B / 2 AND bk6 == 3
C
If ok → Test6Pass++
DDLog("T33 scoring and bonus banking: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)
```

**`RunT34` — pacing (stepped)**
```
P
t0 = 1000.0 ; iv = Max(Cfg.EnemySpawnInterval, 0.2) ; intro = Max(Cfg.WaveIntroTime, 0.0) ; mx = Cfg.WaveMaxActive
GM.StartWave(1, t0)
tot = GM.GetWaveCount(1) ; bs = Min(Cfg.WaveBurstSize, tot) ; first = Min(bs, mx)
st0 = GM.WaveState ; w0 = GM.Wave ; m0 = GM.Multiplier ; wt0 = GM.WaveTotal ; ts0 = GM.WaveToSpawn
GM.DirectorStepAt(t0 + intro * 0.5)        ; n1 = Length(GM.ActiveEnemies) ; st1 = GM.WaveState         // still the intro
GM.DirectorStepAt(t0 + intro + 0.01)       ; n2 = Length(GM.ActiveEnemies) ; st2 = GM.WaveState         // intro over: Running, nothing spawned yet
GM.DirectorStepAt(t0 + intro + 0.02)       ; FreezeEnemies ; n3 = Length(GM.ActiveEnemies) ; ts3 = GM.WaveToSpawn   // first burst
e0 = GM.ActiveEnemies[0] ; spd = e0.Speed                                                                // first enemy's speed
GM.DirectorStepAt(t0 + intro + 0.03)       ; n4 = Length(GM.ActiveEnemies)                              // too early for a burst
GM.DirectorStepAt(t0 + intro + 0.02 + iv + 0.01) ; FreezeEnemies ; n5 = Length(GM.ActiveEnemies) ; ts5 = GM.WaveToSpawn   // second burst
second = Min(Min(Cfg.WaveBurstSize, tot - first), mx - first)
ok = st0 == 1 AND w0 == 1 AND m0 == 1 AND wt0 == tot AND ts0 == tot
     AND n1 == 0 AND st1 == 1 AND n2 == 0 AND st2 == 2
     AND n3 == first AND ts3 == tot - first
     AND |spd - GM.GetEnemySpeed(1)| <= 0.001 AND |spd - Cfg.EnemyBaseSpeed| <= 0.001
     AND n4 == n3
     AND n5 == first + second AND ts5 == tot - first - second
C
If ok → Test6Pass++
DDLog("T34 wave pacing: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)
```
(Store `e0` before reading `spd`. Read `ActiveEnemies[0]` only after `n3` was stored; with `first >= 1` it exists.)

**`RunT35` — active cap (stepped)**
```
P
t0 = 1000.0 ; iv = Max(Cfg.EnemySpawnInterval, 0.2) ; intro = Max(Cfg.WaveIntroTime, 0.0) ; mx = Cfg.WaveMaxActive
Repeat (mx - 1) times:
    e = GM.SpawnEnemy((0, Y, 1800), GM.GetSlotPos(8), V, 8, false) ; e.SetActorTickEnabled(false)
GM.StartWave(1, t0) ; GM.WaveState = 2
tot = GM.GetWaveCount(1)
pre = Length(GM.ActiveEnemies)
GM.DirectorStepAt(t0 + intro + 1.0)            ; FreezeEnemies ; nA = Length(GM.ActiveEnemies) ; tsA = GM.WaveToSpawn   // room for exactly one
GM.DirectorStepAt(t0 + intro + 1.0 + iv + 0.5) ; nB = Length(GM.ActiveEnemies) ; tsB = GM.WaveToSpawn ; stB = GM.WaveState   // no room: nothing
ok = pre == mx - 1 AND nA == mx AND tsA == tot - 1 AND nB == mx AND tsB == tsA AND stB == 2
C
If ok → Test6Pass++
DDLog("T35 active cap: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)
```

**`RunT36` — tally and next wave (stepped)**
```
P
t0 = 1000.0 ; tally = Cfg.WaveTallyTime ; tE = t0 + 10.0
m = Min(2, Max(Cfg.MaxWaveMultiplier, 1))                        // the multiplier of waves 3 and 4
GM.StartWave(3, t0) ; GM.WaveState = 2 ; GM.WaveToSpawn = 0 ; GM.BonusEarned = 100000      // no bonus city can bank here
GM.Cities[0].Demolish() ; GM.Cities[1].Demolish()                                            // 4 cities stand
GM.Bases[0].SetAmmo(3) ; GM.Bases[1].SetAmmo(4) ; GM.Bases[2].Demolish()                     // standing ammo: 3 + 4 = 7
sBefore = GM.Score ; cBefore = GM.WavesCompleted
GM.DirectorStepAt(tE)
st1 = GM.WaveState ; la = GM.LastTallyAmmo ; lc = GM.LastTallyCities ; lap = GM.LastTallyAmmoPts ; lcp = GM.LastTallyCityPts
sc1 = GM.Score ; wc1 = GM.WavesCompleted ; w1 = GM.Wave ; br1 = GM.LastBonusRebuilt ; ca1 = GM.CountAliveCities()
GM.DirectorStepAt(tE + tally * 0.5)  ; st2 = GM.WaveState ; w2 = GM.Wave
GM.DirectorStepAt(tE + tally + 0.5)
st3 = GM.WaveState ; w3 = GM.Wave ; m3 = GM.Multiplier ; ts3 = GM.WaveToSpawn ; tot3 = GM.WaveTotal ; ca3 = GM.CountAliveCities()
bOK = true ; For i in 0..2: b = GM.Bases[i] ; bOK = bOK AND NOT b.bDestroyed AND b.Ammo == Cfg.AbmStartAmmo
ok = st1 == 3 AND la == 7 AND lc == 4
     AND lap == 7 * Cfg.ScoreUnusedAbm * m AND lcp == 4 * Cfg.ScoreSurvivingCity * m
     AND sc1 == sBefore + lap + lcp AND wc1 == cBefore + 1 AND w1 == 3 AND br1 == 0 AND ca1 == 4
     AND st2 == 3 AND w2 == 3
     AND st3 == 1 AND w3 == 4 AND m3 == m AND ts3 == tot3 AND tot3 == GM.GetWaveCount(4) AND bOK AND ca3 == 4
C
If ok → Test6Pass++
DDLog("T36 wave tally and next wave: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)
```
(Store each value in its own local **right after the step shown**, so that every check reads the state at the moment it is meant for.)

**Helper `BonusCase`** (non-Pure; inputs `nDestroy`, `bank`, `expRebuilt`, `expBank` integers; output `bOK` boolean):
```
GM.ClearEnemies(false) ; GM.RestoreAll(false) ; GM.ResetGame() ; GM.bDirectorSilent = true
GM.BonusEarned = 1000000                                        // no new bank from the tally points
For i = 0 to nDestroy - 1: GM.Cities[i].Demolish()
GM.StartWave(1, 1000.0) ; GM.WaveState = 2 ; GM.WaveToSpawn = 0 ; GM.BonusCities = bank
GM.DirectorStepAt(1001.0)
r = GM.LastBonusRebuilt ; bk = GM.BonusCities ; alive = GM.CountAliveCities() ; st = GM.WaveState ; total = Length(GM.Cities)
bOK = r == expRebuilt AND bk == expBank AND alive == total - nDestroy + expRebuilt AND st == 3
GM.ClearEnemies(false) ; GM.RestoreAll(false) ; GM.ResetGame()
```

**`RunT37` — bonus rebuild**
```
P
c1 = BonusCase(3, 2, 2, 0)        // 3 destroyed, 2 banked: both rebuilt, bank empty
c2 = BonusCase(1, 5, 1, 4)        // 1 destroyed, 5 banked: one rebuilt, 4 stay banked
c3 = BonusCase(0, 3, 0, 3)        // nothing destroyed: nothing rebuilt, 3 stay banked
c4 = BonusCase(2, 0, 0, 0)        // nothing banked: nothing rebuilt
ok = c1 AND c2 AND c3 AND c4
C
If ok → Test6Pass++
DDLog("T37 bonus city rebuild: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)
```
(`P` runs once more at the start; `BonusCase` also runs its own preamble and closing. Store each `c` before the next call.)

**`RunT38` — skip and game-over halt (stepped)**
```
P
GM.StartWave(2, 1000.0) ; GM.Score = 500
E1 = GM.SpawnEnemy((0, Y, 1800), GM.GetSlotPos(0), V, 0, false) ; E1.SetActorTickEnabled(false)
E2 = GM.SpawnEnemy((0, Y, 1800), GM.GetSlotPos(4), V, 4, false) ; E2.SetActorTickEnabled(false)
E3 = GM.SpawnEnemy((0, Y, 1800), GM.GetSlotPos(8), V, 8, false) ; E3.SetActorTickEnabled(false)
nSp = Length(GM.ActiveEnemies) ; kB = GM.EnemiesKilled ; xB = GM.ExplosionCount ; sB = GM.Score
GM.SkipWave()
w1 = GM.Wave ; st1 = GM.WaveState ; m1 = GM.Multiplier ; n1 = Length(GM.ActiveEnemies)
k1 = GM.EnemiesKilled ; x1 = GM.ExplosionCount ; s1 = GM.Score ; ts1 = GM.WaveToSpawn ; tot1 = GM.WaveTotal
// game over halts the director
GM.GameOver(false)
GM.DirectorStepAt(2000.0) ; stA = GM.WaveState ; nA = Length(GM.ActiveEnemies) ; tsA = GM.WaveToSpawn
GM.DirectorStepAt(2001.0) ; stB = GM.WaveState ; nB = Length(GM.ActiveEnemies) ; tsB = GM.WaveToSpawn
// V brings the same wave back
GM.RestoreAll(false)
GM.DirectorStepAt(2002.0) ; stC = GM.WaveState ; wC = GM.Wave ; tsC = GM.WaveToSpawn ; totC = GM.WaveTotal
ok = nSp >= 3 AND w1 == 3 AND st1 == 1 AND m1 == Min(2, Max(Cfg.MaxWaveMultiplier, 1)) AND n1 == 0
     AND k1 == kB AND x1 == xB AND s1 == sB AND ts1 == tot1 AND tot1 == GM.GetWaveCount(3)
     AND stA == 4 AND nA == 0 AND tsA == ts1 AND stB == 4 AND nB == 0 AND tsB == ts1
     AND stC == 1 AND wC == 3 AND tsC == totC AND totC == GM.GetWaveCount(3)
C
If ok → Test6Pass++
DDLog("T38 skip and game over halt: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)
```
(`nSp >= 3` instead of `== 3`: the number is only a sanity check. Store every value at the point shown.)

**`RunT39` — kill scoring (stepped)**
```
P
GM.Wave = 3 ; GM.Multiplier = 2                                   // direct; StartWave is not used here
a = 0.625 * M
sBefore = GM.Score ; kBefore = GM.EnemiesKilled ; cBefore = GM.CityHits ; nBefore = Length(GM.ActiveEnemies) ; aBefore = Length(GM.ActiveExplosions)
EX = GM.SpawnExplosion((0, Y, 900), false) ; EX.SetActorTickEnabled(false) ; EX.Advance(G + H * 0.5)      // full radius at (0, 900)
s4 = GM.GetSlotPos(4)
E1 = GM.SpawnEnemy((a, Y, 900), s4, V, 4, false) ; E1.SetActorTickEnabled(false)                         // inside EX
E1.Advance(0)                                                      // dies: scores, spawns EY
EY = GM.LastExplosion ; EY.SetActorTickEnabled(false)
sKill = GM.Score ; lk = GM.LastKillPoints ; kKill = GM.EnemiesKilled
E2 = GM.SpawnEnemy((-800, Y, 1800), GM.GetSlotPos(3), V, 3, false) ; E2.SetActorTickEnabled(false)       // far from every explosion
E2.Advance(1000.0)                                                 // lands on CITY3: damage, no score
EZ = GM.LastExplosion ; EZ.SetActorTickEnabled(false)
sLand = GM.Score ; cLand = GM.CityHits ; kLand = GM.EnemiesKilled
Destroy Actor (EX) ; Destroy Actor (EY) ; Destroy Actor (EZ)
nEnd = Length(GM.ActiveEnemies) ; aEnd = Length(GM.ActiveExplosions)
ok = sKill == sBefore + Cfg.ScoreEnemyMissile * 2 AND lk == Cfg.ScoreEnemyMissile * 2 AND kKill == kBefore + 1
     AND sLand == sKill AND cLand == cBefore + 1 AND kLand == kKill
     AND nEnd == nBefore AND aEnd == aBefore
C                                                                  // RestoreAll brings CITY3 back; ResetGame resets the score
If ok → Test6Pass++
DDLog("T39 kill scoring: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)
```
(The same pattern as the frozen T23 and T22. E1's death creates the explosion `EY` at radius 0; E2 starts far from `EX` and `EY`, so it flies on to its target.)

**Check after each test function:** it compiles clean; it contains no `Delay`; it does not call `RestartGame`, `StartSpawner`, `StopSpawner` or `DirectorLoop`. **Save All after every two tests.**

#### 7f. New function `RunPhase6Tests` (non-Pure, no inputs)
```
Test6Pass = 0
RunT32() ; RunT33() ; RunT34() ; RunT35() ; RunT36() ; RunT37() ; RunT38() ; RunT39()
GM.ClearEnemies(false) ; GM.RestoreAll(false) ; GM.ResetGame()      // leave a clean state: full bases, six cities, silent flag off
DDLog("Phase6 tests: " + Test6Pass + "/8 PASS", "Test", Test6Pass != 8)
```
(`TestPassCount` is **not** touched. The frozen line `SelfTest done: n/31 PASS` stays exactly as it is.)

#### 7g. `Event BeginPlay`: insert one call
Insert **one** call node `RunPhase6Tests()` **right after `RunPhase5Tests()`** and before `SelfTestFlightStart()`, by breaking that one exec wire. The chain is now:

`Branch(bSelfTest)`
- **True** → `Delay(0.5)` → `UpdateArea(Get Viewport Size)` → `Delay(0.5)` → `RunSelfTest()` → `RunPhase3Tests()` → `RunPhase4Tests()` → `RunPhase5Tests()` → **`RunPhase6Tests()`** → `SelfTestFlightStart()` → `Delay(0.5)` → `SelfTestFlightMid()` → `Delay(2.5)` → `SelfTestFlightEnd()` → `Delay(2.0)` → `SelfTestExplosionEnd()` → `SelfTestEnemyStart()` → `Delay(3.5)` → `SelfTestEnemyEnd()`
- **False** → `GM.StartSpawner`

**Check:** compiles clean; keys `H`, `R`, `E`, `K`, `X`, `V`, `N` exist and no others; the chain matches 7g; `StartSpawner` is still called from exactly two places; the frozen functions were not edited. **Save All.**

---

### Step 8 — `BP_DDHUD`

Read the current graphs first. **Edit in place with small node edits; the existing draw code stays.** In the text below `GM` means `OwnerPC.GM`; every new function starts with the guard `IsValid(OwnerPC)` AND `IsValid(OwnerPC.GM)` (otherwise it returns).

**8a. Title.** Change line 1 to: `"DOME DEFENSE | PHASE 6 | WAVES AND SCORE"`. Lines 2 to 4 and the GAME OVER block stay.

**8b. New functions** (one small write each, in this order, non-Pure):

- **`DrawCentered`** (inputs `Text` string, `Color` linear color, `SizeX` float, `Y` float, `Scale` float):
  ```
  (w, h) = Get Text Size(Text, Font = None, Scale = Scale)
  Draw Text(Text, Color, X = SizeX * 0.5 - w * 0.5, Y = Y, Font = None, Scale = Scale, Scale Position = false)
  ```
- **`DrawScoreBlock`** (input `SizeX`):
  ```
  If GM.Wave < 1 → return
  DrawCentered("SCORE " + GM.Score, (1.0, 0.9, 0.2), SizeX, 15, 2.5)
  DrawCentered("WAVE " + GM.Wave + "   x" + GM.Multiplier, White, SizeX, 55, 1.6)
  If GM.BonusCities > 0 → DrawCentered("BONUS CITIES " + GM.BonusCities, (0.3, 1.0, 0.4), SizeX, 85, 1.6)
  ```
- **`DrawBanner`** (inputs `SizeX`, `SizeY`):
  ```
  If GM.WaveState != 1 OR GM.bGameOver → return
  DrawCentered("WAVE " + GM.Wave, (1.0, 0.9, 0.2), SizeX, SizeY * 0.30, 5.0)
  DrawCentered("POINTS x" + GM.Multiplier, White, SizeX, SizeY * 0.30 + 90, 2.5)
  ```
- **`DrawTally`** (inputs `SizeX`, `SizeY`):
  ```
  If GM.WaveState != 3 OR GM.bGameOver → return
  Y0 = SizeY * 0.25
  DrawCentered("WAVE " + GM.Wave + " COMPLETE", (1.0, 0.9, 0.2), SizeX, Y0, 4.0)
  DrawCentered("ROCKETS " + GM.LastTallyAmmo + "   +" + GM.LastTallyAmmoPts, White, SizeX, Y0 + 70, 2.5)
  DrawCentered("CITIES " + GM.LastTallyCities + "   +" + GM.LastTallyCityPts, White, SizeX, Y0 + 115, 2.5)
  If GM.LastBonusRebuilt > 0 → DrawCentered("BONUS CITY REBUILT x" + GM.LastBonusRebuilt, (0.3, 1.0, 0.4), SizeX, Y0 + 160, 2.5)
  ```
- **`DrawWaveInfo`** (inputs `SizeX`, `SizeY`): `DrawScoreBlock(SizeX)` ; `DrawBanner(SizeX, SizeY)` ; `DrawTally(SizeX, SizeY)`.

**8c. One inserted call.** In `Event ReceiveDrawHUD`, insert `DrawWaveInfo(SizeX, SizeY)` **between the last `Draw Text` of line 4 and the `If GM.bGameOver` branch** (break that one wire and route it through the new call). `SizeX` and `SizeY` are the outputs of `Event ReceiveDrawHUD`.

**Fallback:** if `Get Text Size` is not available, use `w = 8.5 * Scale * Length(Text)` and `h = 14 * Scale`. Record it.

**Check:** compiles clean; the HUD still draws lines 1 to 4 and the GAME OVER block. **Save All.**

---

### Step 9 — Save and run

1. **Save All** (save the level before any level load).
2. `StartPIE`. The self-test runs for about **10 seconds** (a little longer than in Phase 5: eight more synchronous tests, no new waits), then the spawner starts and the first wave begins. Wait about **28 seconds** in total.
3. `CaptureEditorImage` at about 6 s (the live test is running, as in Phase 5), then about **11 s, 14 s, 20 s and 26 s** (the intro banner, then enemies in pairs, then more enemies). The times depend on when `SelfTest done` was printed; read the actual times from the log afterwards and say in the report which capture fell where.
4. `GetLogEntries` (filter `[DD]`), then `StopPIE`.
5. Fill in the report (section 8).

---

## 7. Acceptance tests

### A. Static checks
- [ ] **A1** New assets: **none** (no new Blueprint, material or data asset).
- [ ] **A2** Every changed Blueprint compiles with 0 errors and 0 warnings.
- [ ] **A3** `DA_DDConfig` shows the eight new values (array of twelve); the ten recorded existing values are listed; no other value changed (any added scoring value is a recorded deviation).
- [ ] **A4** `BP_DDGameMode` has the 18 new variables and the functions `GetWaveCount`, `GetMultiplier`, `GetEnemySpeed`, `AddScore`, `OnEnemyKilled`, `ResetTally`, `ResetGame`, `TotalStandingAmmo`, `RebuildBases`, `PickRandomDestroyedCity`, `RebuildCities`, `LogWaveStart`, `StartWave`, `SpawnDue`, `ComputeTally`, `LogWaveComplete`, `EndWave`, `CheckWaveEnd`, `StepIntro`, `StepRunning`, `StepTally`, `StepHalted`, `DirectorStepAt`, `SkipWave`, and the custom event `DirectorLoop`.
- [ ] **A5** `StartSpawner` ends with `ResetGame()`, `StartWave(1, ...)`, `DirectorLoop` after the existing log; the call to `EnemySpawnLoop` inside it is gone; the event `EnemySpawnLoop` still exists, unreferenced. `StartSpawner` is called from exactly two places. `StopSpawner` is called from nowhere new.
- [ ] **A6** `SpawnRandomEnemy` passes `GetEnemySpeed(Wave)` to `SpawnEnemy`. `BP_EnemyMissile.Die` calls `GM.OnEnemyKilled(bLogEvents)` once, after `GM.LastKillPos = P` and before `SpawnExplosion`. `OnEnemyKilled` is called from nowhere else.
- [ ] **A7** The controller has key events `H`, `R`, `E`, `K`, `X`, `V`, `N` and no other keys; the new functions `FreezeEnemies`, `BonusCase`, `RunT32`..`RunT39`, `RunPhase6Tests`; the variable `Test6Pass`; the chain of 7g. **No frozen function was edited** (the agent says so explicitly in the report). No new `Delay` in the controller.
- [ ] **A8** `BP_DDHUD` has the new title and the functions `DrawCentered`, `DrawScoreBlock`, `DrawBanner`, `DrawTally`, `DrawWaveInfo`, called once from `ReceiveDrawHUD` before the GAME OVER branch.
- [ ] **A9** No actors were added to the level. No UMG.

### B. Log checks (agent)
Order is not guaranteed. Wait about 28 seconds. "Before done" means before the line `SelfTest done`.

- [ ] **B1** The Phase 0 to 5 startup lines still appear (GameMode, Config OK `Width=2250.0 Height=1800.0 ...`, Camera, PlayerController, View target, Crosshair spawned, HUD, PlayfieldDebug, `[DD][Area] Viewport=...`, `Bases spawned n=3`, `Cities spawned n=6`).
- [ ] **B2** `T1` to `T6`, `T7 fire path`, `T8` to `T31` are PASS exactly as in Phase 5 (including `T22`–`T24`, `T25 enemy live impact`, `T26 enemy killed by explosion: PASS killZ=<n>`).
- [ ] **B3** `[DD][Test] SelfTest done: 31/31 PASS` **exactly once**.
- [ ] **B4** `T32 wave tables: PASS`, `T33 scoring and bonus banking: PASS`, `T34 wave pacing: PASS`, `T35 active cap: PASS`, `T36 wave tally and next wave: PASS`, `T37 bonus city rebuild: PASS`, `T38 skip and game over halt: PASS`, `T39 kill scoring: PASS`, and **`[DD][Test] Phase6 tests: 8/8 PASS`**, all **before done**.
- [ ] **B5** **No** line with `[DD][Wave]` or `[DD][Score]` anywhere before done (the new tests are silent).
- [ ] **B6** The Phase 5 live-segment lines are unchanged and appear before done: `[DD][Enemy] Spawn slot=CITY3 fromX=-300 toX=-252`, `[DD][Enemy] Spawn slot=BASE-CENTER fromX=0 toX=0`, `[DD][Input] Fire request base=CENTER X=0 Z=900 n=1 ammo=9`, `[DD][ABM] Arrived base=CENTER X=0 Z=900`, `[DD][Enemy] Killed slot=BASE-CENTER ...`, `[DD][Enemy] Impact slot=CITY3 X=-252 Z=100`, exactly one `[DD][Damage] City destroyed slot=CITY3 left=5`, exactly four `[DD][Explosion] Start` and four `End` lines. **No other `[DD][Damage]` line and no `[DD][Game]` line before done.**
- [ ] **B7** `[DD][Enemy] Spawner started interval=2500ms` exactly once, **after** done.
- [ ] **B8** After done: `[DD][Wave] Start n=1 count=12 mult=1 speed=130` **exactly once**, and `[DD][Wave] Running n=1` exactly once, **1.5 to 3.5 seconds after** the `Start` line (read the log timestamps).
- [ ] **B9** After `Running n=1`: at least **four** `[DD][Enemy] Spawn slot=...` lines (automatic enemies); the first two carry the same timestamp or are less than 1.0 s apart (one burst of two); later bursts are about 2.5 s apart.
- [ ] **B10** **No** `[DD][Score]` line and **no** `[DD][Wave] Complete` line anywhere (nobody fires; a first wave cannot end within the window). `[DD][Damage]` lines **after** done are allowed (an automatic enemy can land from about 15 s after done); if one appears it must have the form `City destroyed slot=... left=<n>` or `Base destroyed slot=... left=<n>`.
- [ ] **B11** No line containing `ERROR`, `Accessed None`, or `Blueprint Runtime Error`.

### C. Visual checks (editor image of the play view; NEEDS-HUMAN if you cannot capture it)
- [ ] **C1** At rest (before 6 s): black background, no bars, brown ground, **six green city blocks**, three gray-blue bases with numbers `10`. No score text yet.
- [ ] **C2** In a capture right after done (within about 2 s): the HUD shows **`SCORE 0`** and **`WAVE 1   x1`** at the top centre, and the large banner **`WAVE 1`** with **`POINTS x1`** below it. If no capture caught it, mark **NEEDS-HUMAN**; do **not** claim it.
- [ ] **C3** In a capture about 4 s or later after done: the banner is gone, the score block stays, at least one **red dot** is on screen. If not caught, **NEEDS-HUMAN**.
- [ ] **C4** HUD line 1 reads `DOME DEFENSE | PHASE 6 | WAVES AND SCORE`. No GAME OVER text is on screen.

### D. Interactive checks (human, PIE; click inside the viewport first; `Shift+F1` frees the mouse, `Esc` stops PIE; **wait about 10 seconds after Play so the self-test is finished; do not click or press keys before that**)
- [ ] **D1** Wave 1: banner `WAVE 1 / POINTS x1` for about 2 seconds, then enemies come **in pairs about every 2.5 seconds**. Never more than 8 on screen at once.
- [ ] **D2** Shooting an enemy adds **+25** to the score (log `[DD][Score] Kill +25 score=<n>`). An enemy that lands adds nothing. A chain reaction scores every enemy it kills.
- [ ] **D3** `N` skips to the next wave at once (enemies vanish without explosions, banner appears, log `[DD][Wave] Skipped n=1 next=2`). Press `N` again to reach **wave 3**: banner `POINTS x2`, and a kill now gives **+50**.
- [ ] **D4** **Wave end:** after the last missile of a wave is destroyed or has landed (use `K` after the last spawn to be quick) the tally appears for about 3 seconds: `WAVE n COMPLETE`, `ROCKETS <a> +<pts>`, `CITIES <c> +<pts>`. The points are added to the score, matching the `[DD][Wave] Complete ...` log line (ammo × 5 × multiplier, cities × 100 × multiplier). Then the next wave begins: **bases are rebuilt and refilled to 10** (also a base that was destroyed), the wave has the next count (wave 2: 15 missiles) and the missiles are **faster** (log `Start n=2 count=15 mult=1 speed=162` or `163`).
- [ ] **D5** **Bonus city** (temporary test): set `BonusCityScore` to **100** in `DA_DDConfig`, play, press `X` once (one city becomes rubble), shoot four enemies (+100): log `Bonus city banked bank=1`, HUD shows `BONUS CITIES 1`. At the end of the wave the tally says `BONUS CITY REBUILT x1`, the log has `City rebuilt slot=...`, and the rubble is a green city again. **Set `BonusCityScore` back to 10000 afterwards.** With nothing destroyed at the wave end, the bank stays (no city to rebuild).
- [ ] **D6** **Game over unchanged:** when the last city falls, enemies vanish, **GAME OVER** appears, no new enemies. **A banked bonus city does not save you.** Press `V`: everything is rebuilt, the **same wave restarts from its banner**, the **score is kept** (log `[DD][Wave] Restart n=<w>`). A click after the 2-second lockout reloads the level and the **score is back to zero**.
- [ ] **D7** Layout: resize the window (wide, 5:4, nearly square): the score block, banner and tally stay centered and readable; cities, bases and the ground stay as before.
- [ ] **D8** Phase 2 to 5 behavior unchanged: base choice, ammo numbers, `R`, `E`, `K`, `H`, `X`, `V`, crosshair, explosions, chain reactions, damage and rubble.
- [ ] **D9** Feel (no pass/fail): note whether the pairs every 2.5 s, 12 missiles in wave 1, the speed ramp and the point values feel right. All are in `DA_DDConfig` (`WaveCounts`, `WaveBurstSize`, `WaveMaxActive`, `EnemySpawnInterval`, `WaveSpeedFactor`, `EnemyMaxSpeed`, `WaveIntroTime`, `WaveTallyTime`, `BonusCityScore`, and the `Score...` values).

### D-DEVICE. Touch (carried from Phase 1.1; still open)
- [ ] **T-1** A tap launches from the nearest base as before.
- [ ] **T-2** A second finger is ignored.
- [ ] **T-3** One tap makes exactly one launch.
- [ ] **T-4** After game over, a tap during the first 2 seconds does nothing; a tap after that restarts the game.
- [ ] **T-5** The score, wave banner and tally are readable on the phone at landscape size.

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped.

---

## 8. Report format (the agent's final message)

```
## Phase 6 report
**Engine version:** x.y.z
**MCP tools used:** (list)

### Assets created or changed
(table: path | type | change | compiled clean)

### Config values read
(the eight new values; ScoreEnemyMissile, ScoreBomber, ScoreSmartBomb, ScoreUnusedAbm, ScoreSurvivingCity, MaxWaveMultiplier, EnemyBaseSpeed, EnemySpawnInterval, bEnemyAutoSpawn, AbmStartAmmo)

### Acceptance results
A1 PASS/FAIL ... through E1 PASS/FAIL/NEEDS-HUMAN/NEEDS-DEVICE

### Log excerpt
(all [DD] lines from PIE, including every [DD][Test], [DD][Wave], [DD][Score], [DD][Enemy], [DD][Damage], [DD][Game] and [DD][Explosion] line, with timestamps)

### Screenshot notes
(what the HUD, the banner and the enemies looked like; when each capture was taken relative to "SelfTest done")

### Tool problems (errors, retries, editor restarts)
(every tool error or timeout, what was tried, whether the editor stopped responding or restarted; "none" if none)

### Deviations
### Open questions / risks
```

---

## 9. What this hands to Phase 7 and Phase 8

| Provided | How a later phase uses it |
|---|---|
| `GM.ResetGame`, `StartWave(n, Now)`, `DirectorLoop`, `DirectorStepAt` | Phase 7 (game flow) starts and restarts the game through these instead of reloading the level. |
| `WaveState` (0–4), `bGameOver`, `GameOverTime`, `IsRestartAllowed` | Phase 7 replaces reload-on-click with proper states (menu, playing, game over, pause) and keeps these as the facts. |
| `GM.Score`, `BonusCities`, `WavesCompleted`, `Wave` | Phase 7: high score, the game-over screen, saved results. |
| Debug keys `X`, `V`, `N` | Stay until Phase 7 decides what happens to debug keys; the self-test is dropped then, too. |
| `SpawnDue` | Phase 8: this is where a wave's composition grows (MIRVs, bombers, smart bombs); `WaveToSpawn` then counts several kinds. |
| `OnEnemyKilled(bLog)` | Phase 8: gets a weapon type and uses `ScoreBomber` and `ScoreSmartBomb` (they are in the config already). |
| `GetWaveCount`, `GetMultiplier`, `GetEnemySpeed` | Tuning tables; Phase 8 adds per-weapon speed or count tables the same way. |
| `EnemySpawnLoop` (unreferenced) | Dead code; remove it in a cleanup step. |

---

## 10. Notes for the human (not agent steps)

1. **Restart Unreal Editor before you give this to the agent**, and keep the larger page file you set up. The Phase 5 crash was the editor running out of memory while rewriting a large graph; this spec avoids that (small new functions, no rewrites of existing graphs, Save All after each step) and tells the agent to stop and report on errors.
2. **Wave 1 is 12 missiles at the current speed (130)**, in pairs every 2.5 seconds, 2 seconds after the banner. Each wave then gets 25% faster, up to 600. Multiplier ×1 for waves 1–2, ×2 for 3–4, and so on up to ×6. Everything is in `DA_DDConfig`.
3. **`EnemySpawnInterval` now means the time between bursts** (pairs), not between single enemies.
4. **A wave ends only when all its missiles are gone** (shot down or landed). A destroyed city does not end it. Game over is unchanged: the last city falling ends the game at once, and a banked bonus city does not save you.
5. **Cities come back only through bonus cities** (one per 10,000 points, spent at the end of a wave on a random rubble city) and through `V`. **Bases are rebuilt and refilled at the start of every wave.**
6. **`V` after game over restarts the same wave from its banner and keeps the score.** A click or tap after the lockout still reloads the level (score back to zero). Phase 7 changes that.
7. **`N` skips the wave** (no tally, no bonus, no points). `K` after the last missile has spawned ends the wave quickly (no points for the removed enemies).
8. **The self-test now has 39 tests.** The old line still says `31/31`; the new tests report `Phase6 tests: 8/8` on a separate line. It still runs for about 10 seconds at every Play and is dropped later.
9. **`EnemySpawnLoop` is dead code** left in the game mode for a later cleanup.
10. **Touch is still untested.** **Commit** the accepted Phase 5 state **before** running this phase, so there is a clean point to return to.

---

## 11. Common pitfalls

1. **Rewriting an existing function or event graph with `write_graph_dsl`.** This crashed the editor in Phase 5. New functions only; existing graphs by small node edits.
2. **Editing a frozen function** (rule 12). If a frozen test fails, stop and report; do not "fix" it.
3. **Changing `GetEnemySpeed(0)`.** At wave 0 it must return `EnemyBaseSpeed` exactly: the frozen T21 checks it.
4. **Scoring at wave 0.** `OnEnemyKilled` must return at once while `Wave < 1`; otherwise the frozen live segment would score points before the game starts.
5. **Counting the new tests in `TestPassCount`.** They use `Test6Pass`; the frozen line must still say `31/31`.
6. **A second `StartSpawner` call site, or a second `DirectorLoop` start.** `StartSpawner` is the only starter, guarded by `bSpawnerRunning`. Two loops would double every spawn.
7. **`RestoreAll` or a test starting the director.** They must not. The director exists only after `StartSpawner`; the tests call `DirectorStepAt` by hand.
8. **Forgetting the silent flag.** Each test sets `bDirectorSilent = true` after `ResetGame`; `ResetGame` sets it back to `false`. A test that leaves it `true` silences the real game.
9. **Reading a random value twice** in `PickRandomDestroyedCity` (store the index first).
10. **Test enemies or explosions left ticking or alive.** Freeze them right after the call and clear or destroy them; otherwise an enemy can land or die between tests.
11. **Using `Delay` in a function.** Only the event `DirectorLoop` has one.
12. **Counting cities after the rebuild.** `ComputeTally` runs **before** `RebuildCities`.
13. **Spending the bank before the tally points are added.** `AddScore` comes first, then `RebuildCities`.
14. **Calling `DamageSlot`, `RestartGame` or `StopSpawner` from the new code.** Nothing in this phase needs them (the tests use `DamageSlot` only through `Impact`, as in T39).
15. **Leaving the old `EnemySpawnLoop` call in `StartSpawner`.** Then the old random spawner would run on top of the waves.
16. **Leaving a test with something destroyed.** Every test ends with the closing `C` (cities and bases standing, score and wave reset).
