# Phase 5 — Damage, Cities and Game Over

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Depends on** | Phase 4 (agent checks passed 26/26; the human played it and accepted it; touch on a device is still open and is not repeated here) |
| **Phase goal** | Enemy impacts now matter. The six cities become real actors (green blocks). An enemy that reaches a city destroys it; an enemy that reaches a base destroys the base and its ammo drops to 0. A destroyed city or base turns into flat dark rubble, and an orange explosion flashes at the impact point. New enemies aim only at slots that still stand. When the last city falls, the game is over: enemies are cleared, no new ones appear, a large **GAME OVER** is shown, and a click or tap (after a short lockout) reloads the level. Two desktop debug keys speed up testing. |

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **No bars, no aspect caps.** The whole window is the playfield. Do not add any.
3. **Existing logic must survive** unless a step says to change or delete it. Read each Blueprint's current graphs (`read_graph_dsl`) before editing.
4. **Compile after every Blueprint change.** Every Blueprint ends with no errors and no warnings.
5. **Run each step's "Check"** before moving on.
6. **You cannot press keys, click, touch, or resize in PIE.** The self-test checks damage and game over with **synthetic calls** (no waiting) and one short live segment. Interactive checks (group D) are **NEEDS-HUMAN**; touch is **NEEDS-DEVICE**. The restart itself reloads the level, so the agent **never** triggers it (see 3.9).
7. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
8. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after each test. **Save before loading a level.**
9. **Read sections 4 and 5 before building graphs.**
10. **This phase adds exactly one config value** (Step 1). Change no other config value.

---

## 2. Scope

### In scope
- Config: `GameOverRestartDelay`.
- Material instances `MI_DD_City` and `MI_DD_Rubble`.
- `BP_City`: intact block, rubble block, `Demolish`, `Rebuild`.
- `BP_MissileBase`: rubble block, destroyed state, `SetAmmo` guard.
- `BP_DDGameMode`: city spawning, damage, game over, restore, target filtering, debug helper.
- `BP_EnemyMissile.Impact`: calls the damage function.
- `BP_DDPlayerController`: debug keys `X` and `V`, game-over guard in `RequestFire`, `RestartGame`, changed Phase 4 tests, new tests T27–T31 (total **31**).
- `BP_DDHUD`: title line and the GAME OVER text.
- `BP_PlayfieldDebug`: remove the base and city boxes (the real actors replace them).

### Out of scope (do NOT create)
- Scoring, waves, speed ramps, ammo restock, rebuilding bases or cities between waves, bonus cities (Phase 6).
- Game flow, menus, pause, high scores (Phase 7).
- MIRV, bombers, smart bombs, sounds, particles, destruction animations, imported art.
- Any UMG. Any key besides `X` and `V`. Any config value besides `GameOverRestartDelay`.
- Damage from any source except an enemy impact (explosions and ABMs never damage cities or bases; the debug key `X` goes through the same impact damage path, see 3.10).

---

## 3. Design (decided; do not change)

### 3.1 Slots and their state
The nine ground slots of Phase 4 stay exactly as they are (0 BASE-LEFT, 1–3 CITY1–3, 4 BASE-CENTER, 5–7 CITY4–6, 8 BASE-RIGHT; `GetSlotPos`, `GetSlotName`). Each slot is **standing** or **destroyed**:

| Slot | Object | Object index |
|---|---|---|
| 0, 4, 8 | `GM.Bases[i]` | 0, 1, 2 |
| 1, 2, 3, 5, 6, 7 | `GM.Cities[i]` | 0, 1, 2, 3, 4, 5 |

`IsSlotAlive(Slot)` is true when the object exists and its `bDestroyed` is false. Any other slot number (below 0 or above 8) is **not** alive.

### 3.2 What an impact does
When an enemy reaches its target (`BP_EnemyMissile.Impact`), after the Phase 4 counters and log, the enemy is destroyed and then `GM.DamageSlot(Slot, bLog)` runs:
- **Slot standing:** the object is demolished (city → rubble; base → rubble and `Ammo = 0`), the matching counter rises (`CityHits` or `BaseHits`), `LastDamageSlot = Slot`, and **one explosion** is created at the slot position (`GM.SpawnExplosion`, same size and timing as any explosion). Then the game-over check runs.
- **Slot already destroyed (rubble):** **nothing happens**. No explosion, no counter, no visual change. (A log line only if `bLog`.)

### 3.3 Cities (placeholder look)
- `BP_City`, spawned at runtime by the game mode at `(CityX[i], PlayPlaneY, GroundTopZ)`.
- **Intact:** one green block, 120 wide × 40 deep × 80 tall (cube scale `(1.2, 0.4, 0.8)`), standing on the ground (relative Z 40), material `MI_DD_City` (the Phase 0 city green).
- **Rubble:** one flat dark-gray block, 140 wide × 40 deep × 14 tall (cube scale `(1.4, 0.4, 0.14)`, relative Z 7), material `MI_DD_Rubble`.
- Exactly **one** of the two is visible at any time. Nothing else changes (no animation).

### 3.4 Bases
- A destroyed base shows its own rubble block (120 wide × 40 deep × 14 tall, same material `MI_DD_Rubble`) instead of its body.
- Its ammo is **0** and **stays 0**: `SetAmmo` on a destroyed base always ends with `Ammo = 0` (so `R`, `RefillBases` and `SetAmmo(7)` cannot revive it). The number under it reads `0`.
- The nearest-base rule needs **no change**: a base with `Ammo = 0` is already skipped. If all bases are gone, a click logs `No ammo` and does nothing.
- Bases are brought back only by `RestoreAll` (key `V`) or by a level reload. (Phase 6 will rebuild them between waves.)

### 3.5 Targeting
`SpawnRandomEnemy` picks its target **uniformly among the standing slots** (those where `IsSlotAlive` is true), at spawn time. An enemy already in flight keeps its target; if that slot has fallen meanwhile, it lands on the rubble with no effect (3.2).

### 3.6 Impact explosions
They are ordinary explosions. They **can kill other enemies** (and those create their own explosions: chain reaction), exactly like ABM explosions. They do not damage cities, bases or ABMs.

### 3.7 Game over
- **Trigger:** all six cities are destroyed. Losing bases alone does **not** end the game (without bases you simply cannot shoot).
- **Effects, once:** `bGameOver = true`, `GameOverTime = now`, `GameOverCount + 1`, every enemy removed **without explosions** (`ClearEnemies(false)`). From now on `SpawnRandomEnemy` spawns **nothing** (the automatic spawner keeps its timer but does nothing; no `StopSpawner` call, see 3.9). ABMs already flying and explosions already running finish normally.
- **While game over:** `RequestFire` launches nothing (3.9). The HUD shows GAME OVER (3.11).

### 3.8 Counting standing things
`CountAliveCities` and `CountAliveBases` count objects with `bDestroyed = false`. They are used by the game-over check and by logs.

### 3.9 Restart
- `GM.IsRestartAllowed()` is true when `bGameOver` is true **and** at least `Config.GameOverRestartDelay` (2.0 s) have passed since `GameOverTime`. This lockout stops a click made just as the last city falls from restarting the game by accident.
- `RequestFire` (called by mouse click and by touch): **if `bGameOver`** → if `IsRestartAllowed` → `RestartGame()`; in both cases **return** without firing. Otherwise the Phase 4 body runs unchanged.
- `RestartGame` (controller): reloads the current level (`Open Level` by name, using `Get Current Level Name`). Everything resets because a new game mode and controller are created. The 10-second self-test runs again after a reload (it disappears when the self-test is dropped).
- **Why the spawner is not stopped:** `StopSpawner` followed by `StartSpawner` while the old `Delay` is still pending would create **two** spawn loops. Instead the loop keeps running and `SpawnRandomEnemy` simply returns nothing during game over. After `V` (which clears `bGameOver`) the same loop resumes by itself.

### 3.10 Debug keys (desktop only; they do not exist on Android)
- **`X`** destroys one random **standing city** through the normal damage path (explosion, counters, game-over check). If none stands: logs and does nothing.
- **`V`** (`RestoreAll`) rebuilds **every** city, rebuilds **every** base with full ammo, and clears game over. It does **not** touch the spawner and does not reload anything.
- Existing keys stay: `H` (overlay), `R` (refill; skips destroyed bases), `E` (spawn enemy; spawns nothing during game over), `K` (clear enemies). `N` stays reserved for Phase 6.

### 3.11 HUD
- Line 1 (title): `DOME DEFENSE | PHASE 5 | CITIES AND GAME OVER`. Lines 2 to 4 unchanged.
- While `GM.bGameOver`: **GAME OVER** in large red text, horizontally centered, at about 40% of the screen height. After the lockout (`IsRestartAllowed`), a second line **TAP OR CLICK TO RESTART** in white below it. Both are drawn by `BP_DDHUD` canvas calls (no UMG).

### 3.12 Layering (Y offset of visual components)
| Layer | Y offset |
|---|---|
| Ground | 0 |
| Bases (body, rubble) and ammo number | +60 |
| **Cities (body, rubble)** | **+60** |
| Enemies | +70 |
| ABMs | +80 |
| Explosions | +90 |
| Crosshair | +100 |
| Tap marker | +110 |

### 3.13 Log lines
Only real events log; synthetic test calls pass `bLog = false` and stay silent. The live test enemy logs.

| Category | Line |
|---|---|
| `Core` | `[DD][Core] Cities spawned n=6` |
| `Damage` | `[DD][Damage] City destroyed slot=<NAME> left=<int>` (`left` = cities still standing) |
| `Damage` | `[DD][Damage] Base destroyed slot=<NAME> left=<int>` (`left` = bases still standing) |
| `Damage` | `[DD][Damage] Already destroyed slot=<NAME>` |
| `Damage` | `[DD][Damage] No surviving city` (key `X` with nothing left) |
| `Game` | `[DD][Game] GAME OVER cities=0 bases=<int>` |
| `Game` | `[DD][Game] Restored cities=<int> bases=<int>` (from `V`) |
| `Game` | `[DD][Game] Restart requested` (from `RestartGame`) |

---

## 4. Conventions as built (these replace earlier text where they differ)

| Topic | As built |
|---|---|
| Camera | **Ortho Width is the vertical extent** (visible height). Do not change `ApplyArea`. |
| Visible area | `BP_DDGameMode` holds `VisibleMinX/MaxX/MinZ/MaxZ`. Enemy spawn positions use it. |
| Reading values | A value read in a later node is evaluated **when that node runs**. **Store a value in a variable (or a local) before you change what it depends on.** This also applies to **random nodes**: every read of `Random ... In Range` rolls a new number. |
| Config | Config functions and `GetConfig` are **not Pure**. The game mode uses its cached `Config`; the controller uses its cached `Cfg`. |
| Logging | Only through the static `DDLog(Message, Category, bIsError)`. Coordinates in logs are rounded integers. |
| Ticking | A Blueprint actor needs **Can Ever Tick = true** and **Start With Tick Enabled = true** for its Event Tick to run. `BP_City` does not tick: leave both **false**. |
| Self-test | `bSelfTest` (default true) starts a chain from `BeginPlay`. `RunSelfTest` runs T1–T6, T8, T9 and calls `RunPhase2Tests`; `RunPhase3Tests` runs T16–T18; `RunPhase4Tests` runs T20–T21 and calls the helper **`RunPhase4Rest`**, which holds **T22–T24** (the agent split it because one graph write timed out; edit T22 where it lives). `SelfTestFlightStart/Mid/End` run T14–T15; `SelfTestExplosionEnd` runs T19; `SelfTestEnemyStart/End` run the live segment (T25, T26) and end with `SelfTest done`. Sample values inside a test function are **function locals**. |
| Test variable names | Older test variables (`C1`–`C6`, `A1`, `A2`, `T1Text`, `T2Text`, `MBefore`, `ABefore`) keep their names. **Leave them.** New test variables use the prefix `Test`. |
| Synthetic tests | They call `Advance` by hand on actors whose tick is disabled (`SetActorTickEnabled(false)`). **Every explosion a test creates is frozen and destroyed by the test itself** (an explosion destroyed from outside does not count as finished, so `ExplosionsFinished` is untouched). |
| Hooks | Comment boxes cannot be created. A "hook" is described by the node **after which** the new call goes. |
| Text of the HUD title | One space on each side of every pipe character. |
| Slow editor | "max tick rate 3" appears when the editor window is not focused. Do not tighten the tolerances given below. |
| Spawner | `StartSpawner` is called from exactly two places (`SelfTestEnemyEnd` and the `bSelfTest = false` branch of `BeginPlay`). **Phase 5 adds no third.** |

---

## 5. Tooling notes

1. **Do not guess node IDs.** Use `find_node_types`. Call `get_graph_dsl_docs` once.
2. **Key events cannot be written as DSL.** Use `create_node` + `connect_pins`, **Pressed** pin only. Known ID pattern: `Input|KeyboardEvents|H`. Find the ones for `X` and `V` with `find_node_types`.
3. **Latent nodes (`Delay`) only work in event graphs**, not in functions. Nothing in this phase adds a `Delay`.
4. **Non-Pure functions need exec wires.** Call each function in sequence, store each result in its own variable or local, then combine the variables.
5. **Sequence pins** are `then_0`, `then_1`. **No nodes after a Branch/return** in the same chain. Where the pseudo-code shows an `If` followed by more steps, either put the `If` inside a small helper function, or connect **both branch ends to the same next exec pin** (an exec input accepts several wires). A `Branch` that ends a loop body is fine.
6. **Spawn Actor From Class:** Collision Handling Override = **Always Spawn**. After spawning, call the actor's `Init...` function in the same exec chain.
7. **`Destroy Actor` runs `EndPlay` immediately.** In same-frame tests do **not** use `IsValid` on a just-destroyed actor; use `Length(GM.ActiveEnemies)` or `Length(GM.ActiveExplosions)`.
8. **`Set Actor Tick Enabled(false)`** keeps an actor still. A newly spawned explosion has not ticked yet when the spawning function returns, so freeze it **immediately** after the call.
9. **Loops over an array that can change:** loop over a **copy**, or collect first and act afterwards.
10. **Visibility:** use the component node `Set Visibility(New Visibility, Propagate To Children = false)` and read it back with `Is Visible`. Read every property you set back.
11. **Properties not settable through ObjectTools:** `bIsPureFunc`, `BlueprintType`, `ActorLabel`. The variable-rename tool does not update references: do not rename anything.
12. **PIE tools:** `StartPIE`, `StopPIE`, `IsPIERunning`, `GetLogEntries`, `CaptureViewport`, `CaptureEditorImage`. Use `CaptureEditorImage` for the play view.

---

## 6. Implementation steps

### Step 1 — Config

Add to `PDA_DDConfig` (Instance Editable, category `Game`):

| Variable | Type | Default |
|---|---|---|
| `GameOverRestartDelay` | float | `2.0` |

Read it back on `DA_DDConfig`; set it on the instance if it does not show `2.0`. Also read back and record `AbmStartAmmo` (expected 10), `EnemyBaseSpeed` (130), `EnemySpawnInterval` (2.5). **Do not change any other value.**

**Check:** `DA_DDConfig.GameOverRestartDelay = 2.0`; `PDA_DDConfig` compiles clean.

---

### Step 2 — Materials

Create two Material Instances (parent `M_DD_Unlit`) in `/Game/DomeDefense/Materials/`:

| Asset | `Color` |
|---|---|
| `MI_DD_City` | `(0.10, 0.80, 0.20, 1.0)` (green) |
| `MI_DD_Rubble` | `(0.28, 0.28, 0.30, 1.0)` (dark gray) |

**Check:** both exist; parent is `M_DD_Unlit`; colors read back.

---

### Step 3 — `BP_City`

Create Blueprint class `BP_City` (parent `Actor`) in `/Game/DomeDefense/Actors/`.

**Components** (root is a `Scene` component `Root`):

| Name | Type | Settings |
|---|---|---|
| `Body` | StaticMeshComponent | mesh `/Engine/BasicShapes/Cube`; relative location (0, **60**, 40); relative scale (1.2, 0.4, 0.8); material `MI_DD_City`; NoCollision |
| `Rubble` | StaticMeshComponent | mesh `/Engine/BasicShapes/Cube`; relative location (0, **60**, 7); relative scale (1.4, 0.4, 0.14); material `MI_DD_Rubble`; NoCollision |

(Read back the collision-enabled flag of both, not only the preset.)

**Class defaults:** Can Ever Tick = **false**, Start With Tick Enabled = **false**.

**Variables** (not Instance Editable): `CityIndex` (integer, 0), `bDestroyed` (boolean, false).

**Functions** (all non-Pure):

- **`ApplyVisual`** (no inputs): `Body.SetVisibility(NOT bDestroyed)` ; `Rubble.SetVisibility(bDestroyed)`. (Store `NOT bDestroyed` and `bDestroyed` in locals first if the graph needs it.)
- **`InitCity`** (input `Index` integer): `CityIndex = Index ; bDestroyed = false ; ApplyVisual()`.
- **`Demolish`** (no inputs): `bDestroyed = true ; ApplyVisual()`.
- **`Rebuild`** (no inputs): `bDestroyed = false ; ApplyVisual()`.
- **`GetBodyShown`** (output `bShown` boolean): `bShown = Body.IsVisible()`.
- **`GetRubbleShown`** (output `bShown` boolean): `bShown = Rubble.IsVisible()`.

**Check:** compiles clean. Read back both components' relative Y = 60, scales, materials, collision disabled; both tick flags false.

---

### Step 4 — `BP_MissileBase`

Read the current graphs first. **Keep everything** (`SetAmmoDisplay` stays the only writer of `AmmoText`; `ConsumeAmmo`, `GetAmmoDisplayText`, `LaunchPos` unchanged).

**4a. New component** `Rubble`: StaticMeshComponent, mesh `/Engine/BasicShapes/Cube`, relative location (0, **60**, 7), relative scale (1.2, 0.4, 0.14), material `MI_DD_Rubble`, NoCollision. Make sure it is **not** hidden by a component default that the code cannot undo (the code controls visibility, see `ApplyVisual`).

**4b. New variable** (not Instance Editable): `bDestroyed` (boolean, false).

**4c. New and changed functions** (non-Pure):

- **`ApplyVisual`** (no inputs): `Body.SetVisibility(NOT bDestroyed)` ; `Rubble.SetVisibility(bDestroyed)`.
- **`SetAmmo`** (input `NewAmmo` integer) — **changed**:
  ```
  If bDestroyed → Ammo = 0
  Else          → Ammo = Max(NewAmmo, 0)
  SetAmmoDisplay(Ammo)
  ```
  (Both branch ends connect to the same `SetAmmoDisplay` call.)
- **`Demolish`** (no inputs): `bDestroyed = true ; SetAmmo(0) ; ApplyVisual()`.
- **`Rebuild`** (input `StartAmmo` integer): `bDestroyed = false ; ApplyVisual() ; SetAmmo(StartAmmo)`.
- **`InitBase`** — **add one call**: `bDestroyed = false ; ApplyVisual()` **before** the existing `SetAmmo(StartAmmo)`. Keep the rest.
- **`GetBodyShown`** (output `bShown`): `Body.IsVisible()`. **`GetRubbleShown`** (output `bShown`): `Rubble.IsVisible()`.

**Check:** compiles clean. `AmmoText` is still written only inside `SetAmmoDisplay` (search the graph).

---

### Step 5 — `BP_DDGameMode`

Read the existing graphs first. Keep everything.

#### 5a. New variables (not Instance Editable)

| Variable | Type | Default |
|---|---|---|
| `Cities` | array of `BP_City` reference | empty |
| `bGameOver` | boolean | false |
| `GameOverTime` | float | 0 |
| `GameOverCount` | integer | 0 |
| `CityHits` | integer | 0 |
| `BaseHits` | integer | 0 |
| `LastDamageSlot` | integer | -1 |

Compile.

#### 5b. Mapping, state and counting (all non-Pure)

**`SlotToBaseIndex`** (input `Slot` integer; output `Index` integer): `0 → 0`, `4 → 1`, `8 → 2`, otherwise `-1`.

**`SlotToCityIndex`** (input `Slot`; output `Index`): `1 → 0`, `2 → 1`, `3 → 2`, `5 → 3`, `6 → 4`, `7 → 5`, otherwise `-1`.

**`CityIndexToSlot`** (input `Index`; output `Slot`): `0 → 1`, `1 → 2`, `2 → 3`, `3 → 5`, `4 → 6`, `5 → 7`, otherwise `-1`.

**`IsSlotAlive`** (input `Slot` integer; output `bAlive` boolean):
```
bi = SlotToBaseIndex(Slot) ; ci = SlotToCityIndex(Slot)          // store each in its own local
If bi >= 0 AND bi < Length(Bases):
    B = Bases[bi] ; return IsValid(B) AND NOT B.bDestroyed
If ci >= 0 AND ci < Length(Cities):
    C = Cities[ci] ; return IsValid(C) AND NOT C.bDestroyed
return false
```
(Three separate `return` nodes on separate branches.)

**`CountAliveCities`** (output `Count` integer): local `n = 0`; for each index `i` of `Cities` → `C = Cities[i]` ; if `IsValid(C)` AND NOT `C.bDestroyed` → `n = n + 1`. Return `n`.

**`CountAliveBases`** (output `Count`): the same over `Bases`.

**Check:** compiles clean.

#### 5c. Damage and game over (all non-Pure)

**`DamageSlot`** (inputs `Slot` integer, `bLog` boolean; output `bDamaged` boolean):
```
bAlive = IsSlotAlive(Slot)                                        // store
If NOT bAlive:
    If bLog AND Slot >= 0 AND Slot <= 8 → DDLog("Already destroyed slot=" + GetSlotName(Slot), "Damage", false)
    return false
bi = SlotToBaseIndex(Slot) ; ci = SlotToCityIndex(Slot)           // store
If bi >= 0:
    Bases[bi].Demolish() ; BaseHits = BaseHits + 1 ; LastDamageSlot = Slot
    If bLog → left = CountAliveBases() ; DDLog("Base destroyed slot=" + GetSlotName(Slot) + " left=" + left, "Damage", false)
Else:
    Cities[ci].Demolish() ; CityHits = CityHits + 1 ; LastDamageSlot = Slot
    If bLog → left = CountAliveCities() ; DDLog("City destroyed slot=" + GetSlotName(Slot) + " left=" + left, "Damage", false)
// both branches continue here (same next exec pin):
P = GetSlotPos(Slot)                                              // store
SpawnExplosion(P, bLog)
CheckGameOver(bLog)
return true
```
(`left` is read **after** `Demolish`, so it already excludes the destroyed object.)

**`CheckGameOver`** (input `bLog` boolean):
```
If bGameOver → return
n = CountAliveCities()                                            // store
If n == 0 → GameOver(bLog)
```

**`GameOver`** (input `bLog` boolean):
```
bGameOver = true
GameOverTime = Get Game Time In Seconds
GameOverCount = GameOverCount + 1
ClearEnemies(false)                                               // no explosions, no kill or impact counting
If bLog → nb = CountAliveBases() ; DDLog("GAME OVER cities=0 bases=" + nb, "Game", false)
```
(Do **not** call `StopSpawner` here: see 3.9.)

**`IsRestartAllowed`** (output `bAllowed` boolean):
```
Now = Get Game Time In Seconds
bAllowed = bGameOver AND ((Now - GameOverTime) >= Config.GameOverRestartDelay)
```

**Check:** compiles clean.

#### 5d. Restore and debug helper (non-Pure)

**`RestoreAll`** (input `bLog` boolean):
```
For each index i of Cities: C = Cities[i] ; If IsValid(C) → C.Rebuild()
For each index i of Bases:  B = Bases[i]  ; If IsValid(B) → B.Rebuild(Config.AbmStartAmmo)
bGameOver = false
If bLog → nc = CountAliveCities() ; nb = CountAliveBases() ; DDLog("Restored cities=" + nc + " bases=" + nb, "Game", false)
```
(`RestoreAll` does **not** call `StartSpawner`, `StopSpawner` or `ClearEnemies`. The counters `CityHits`, `BaseHits`, `GameOverCount` are cumulative and are **not** reset.)

**`DestroyRandomCity`** (input `bLog` boolean; output `bDone` boolean):
```
Alive = local array of integers (empty)
For each index i of Cities: C = Cities[i] ; If IsValid(C) AND NOT C.bDestroyed → Alive.Add(i)
n = Length(Alive)
If n == 0:
    If bLog → DDLog("No surviving city", "Damage", false)
    return false
k = Random Integer In Range(0, n - 1)                             // store in a variable FIRST
ci = Alive[k]
Slot = CityIndexToSlot(ci)                                        // store
r = DamageSlot(Slot, bLog)                                        // store
return r
```
**Fallback:** if a local array cannot be created, use a game-mode variable `AliveScratch` (array of integer, not Instance Editable), cleared at the start. Record it.

**Check:** compiles clean.

#### 5e. Spawn the cities

**`SpawnCities`** (no inputs):
```
Cities.Clear()
For i = 0 to Length(Config.CityX) - 1:
    C = Spawn Actor From Class(BP_City, Location = (Config.CityX[i], Config.PlayPlaneY, Config.GroundTopZ), Collision = Always Spawn)
    C.InitCity(i)
    Cities.Add(C)
DDLog("Cities spawned n=" + Length(Cities), "Core", false)
```
**`Event BeginPlay` addition:** call `SpawnCities()` **right after** the existing `SpawnBases()`.

**Check:** compiles clean; both calls exist in `BeginPlay`.

#### 5f. Change `SpawnRandomEnemy` (input `bLog`; output `Enemy`)

New body (replaces the Phase 4 body):
```
If bGameOver → return none                                        // nothing spawns during game over
Alive = local array of integers (empty)
For Slot = 0 to 8: a = IsSlotAlive(Slot) ; If a → Alive.Add(Slot)
n = Length(Alive)
If n == 0 → return none
k = Random Integer In Range(0, n - 1)                             // store in a variable FIRST
Slot = Alive[k]                                                   // store
X = Random Float In Range(VisibleMinX, VisibleMaxX)               // store in a variable FIRST
Start = (X, Config.PlayPlaneY, VisibleMaxZ)
Tg = GetSlotPos(Slot)
return SpawnEnemy(Start, Tg, Config.EnemyBaseSpeed, Slot, bLog)
```
**Fallback:** the same `AliveScratch` variable as in 5d if a local array is not possible.

**Check:** compiles clean. `StartSpawner`, `EnemySpawnLoop`, `StopSpawner` and `SpawnEnemy` are unchanged.

---

### Step 6 — `BP_EnemyMissile.Impact`: the hook

Read the current `Impact`. Keep everything in it and **add at the end**:

```
// existing: bActive = false ; SetActorLocation(Target) ; counters ; LastImpact* ; the Impact log
G = GM ; Slot = SlotIndex ; bL = bLogEvents          // NEW: three locals, stored BEFORE Destroy Actor
Destroy Actor                                        // existing
If IsValid(G) → G.DamageSlot(Slot, bL)               // NEW: the Phase 5 hook, after Destroy Actor
```
**Why this order:** `DamageSlot` can end the game, and game over clears every enemy. If this enemy were still alive and in `ActiveEnemies` at that moment, it would be destroyed twice. The locals keep the values valid after the actor is gone. Do **not** move `DamageSlot` before `Destroy Actor`.

`Die` and `Advance` stay unchanged.

**Check:** compiles clean; `DamageSlot` is called only from `Impact`, `DestroyRandomCity` and the controller's test functions.

---

### Step 7 — `BP_DDPlayerController`

Read the existing graphs first. Keep everything else (self-test chain order, `RunPhase4Rest`, `SelfTest*` functions, `LaunchAt`, `SelectBase`, `ConsiderBase`).

#### 7a. Debug keys
With `create_node` + `connect_pins` (Pressed pin only):
- **`X`** → `GM.DestroyRandomCity(true)`
- **`V`** → `GM.RestoreAll(true)`

`H`, `R`, `E`, `K` stay. No other keys.

#### 7b. New variables (not Instance Editable)

| Variable | Type | Default |
|---|---|---|
| `bRestarting` | boolean | false |
| `TestEnemyCityHitsBefore` | integer | 0 |

#### 7c. New function `RestartGame` (non-Pure, no inputs)
```
If bRestarting → return
bRestarting = true
DDLog("Restart requested", "Game", false)
Open Level (by Name)(Level Name = Get Current Level Name(Remove Prefix String = true))
```
(Nothing in the self-test ever calls `RestartGame`.)

#### 7d. Change `RequestFire` (inputs `ScreenX`, `ScreenY` unchanged)
Insert **at the very top, before the debounce**:
```
If GM.bGameOver:
    ar = GM.IsRestartAllowed()                                   // store
    If ar → RestartGame()
    return                                                       // both paths return; nothing is fired
```
(Two nested branches whose ends all `return`. The Phase 4 body continues from the **False** pin of the first branch, unchanged.)

#### 7e. Change the Phase 4 tests (edit them where they live)

**T22 enemy flight and impact (stepped)** — in `RunPhase4Rest`. The impact now hits a **standing** base (slot 8), which is destroyed and creates an explosion. Replace the T22 block with:
```
T22 enemy flight and impact (stepped)
    iBefore = GM.EnemyImpacts ; kBefore = GM.EnemiesKilled ; xBefore = GM.ExplosionCount ; nBefore = Length(GM.ActiveEnemies)
    bhBefore = GM.BaseHits ; aBefore = Length(GM.ActiveExplosions)
    St = (-800, Y, 1800) ; Tg = GM.GetSlotPos(8)
    E = GM.SpawnEnemy(St, Tg, V, 8, false) ; E.SetActorTickEnabled(false)
    d0 = Distance(St, Tg) ; dir = (Tg - St) / d0
    E.Advance(2.0) ; p1 = E.GetActorLocation ; exp1 = St + dir * (V * 2.0)
    E.Advance(1.0) ; p2 = E.GetActorLocation ; exp2 = St + dir * (V * 3.0)
    midOK = (Length(GM.ActiveEnemies) == nBefore + 1) AND (GM.EnemyImpacts == iBefore) AND (GM.BaseHits == bhBefore)
    E.Advance(1000.0)                                   // far longer than the remaining flight: impact on the standing right base
    EX = GM.LastExplosion ; EX.SetActorTickEnabled(false)             // freeze the impact explosion at once
    iNow = GM.EnemyImpacts ; ipos = GM.LastImpactPos ; islot = GM.LastImpactSlot ; nNow = Length(GM.ActiveEnemies)
    kNow = GM.EnemiesKilled ; xNow = GM.ExplosionCount ; bhNow = GM.BaseHits ; dslot = GM.LastDamageSlot
    aliveNow = GM.IsSlotAlive(8)                        // store each of these in its own local
    Destroy Actor (EX) ; GM.RestoreAll(false)           // clean up: explosion gone, everything standing again
    aEnd = Length(GM.ActiveExplosions) ; aliveEnd = GM.IsSlotAlive(8)
    ok = Distance(p1, exp1) <= 0.5 AND Distance(p2, exp2) <= 0.5 AND midOK
         AND iNow == iBefore + 1 AND Distance(ipos, Tg) <= 0.5 AND islot == 8
         AND nNow == nBefore AND kNow == kBefore
         AND xNow == xBefore + 1 AND bhNow == bhBefore + 1 AND dslot == 8 AND NOT aliveNow
         AND aEnd == aBefore AND aliveEnd
    If ok → TestPassCount++
    DDLog("T22 enemy flight and impact: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)
```
(The only differences from Phase 4: the explosion and base-damage conditions, the freeze, the cleanup. The flight checks are the same.)

**T21, T23, T24 stay exactly as they are** (no impact happens in them; all slots stand when they run).

**Live segment, `SelfTestEnemyStart`** — two changes:
1. Add at the top: `TestEnemyCityHitsBefore = GM.CityHits`.
2. Enemy A now starts lower so that its impact explosion is surely finished before `SelfTestEnemyEnd` even at 3 fps. Change its start from `(-300, Y, 300)` to **`(-300, Y, 200)`**:
   ```
   TestEnemyA = GM.SpawnEnemy((-300, Y, 200), GM.GetSlotPos(3), 130.0, 3, true)    // flight to CITY3: about 0.85 s
   ```
   Enemy B, the ABM launch and everything else in that function stay as they are. The log line is still `Spawn slot=CITY3 fromX=-300 toX=-252`.

**Live segment, `SelfTestEnemyEnd`** — changes (the `Delay(3.5)` before it stays):
- **T25** gets a damage condition. Replace its block with:
  ```
  aImpact  = (GM.EnemyImpacts == TestEnemyImpactsBefore + 1)
  aPos     = Distance(GM.LastImpactPos, GM.GetSlotPos(3)) <= 0.5 AND GM.LastImpactSlot == 3
  aGone    = NOT IsValid(TestEnemyA)
  cityA    = GM.Cities[2]                                             // store
  aDamage  = (GM.CityHits == TestEnemyCityHitsBefore + 1) AND IsValid(cityA) AND cityA.bDestroyed
             AND NOT GM.IsSlotAlive(3) AND GM.LastDamageSlot == 3
  ok25 = aImpact AND aPos AND aGone AND aDamage
  If ok25 → TestPassCount++
  DDLog("T25 enemy live impact: " + (ok25 ? "PASS" : "FAIL"), "Test", NOT ok25)
  ```
- **T26**: only `xOK` changes. Three explosions now exist in this segment (the ABM's, enemy B's own, and the impact explosion on CITY3):
  ```
  xOK = (GM.ExplosionCount == TestEnemyExplBefore + 3) AND (GM.ExplosionsFinished == TestEnemyFinishedBefore + 3)
  ```
  All other T26 conditions stay.
- **End of the function:** insert **`GM.RestoreAll(false)`** as the first line of the closing block, **before** `GM.RefillBases(false)`. Change the summary line to `DDLog("SelfTest done: " + TestPassCount + "/31 PASS", "Test", TestPassCount != 31)`. `GM.StartSpawner` stays the last call.

(Why the numbers work: A's flight is about 0.85 s, so it lands around 1 s after the start, even at 3 fps; its explosion lives 1.3 s and ends before about 2.7 s. `SelfTestEnemyEnd` runs at 3.5 s. A's explosion is at `X = -252`, far from B and from the ABM explosion, so it kills nothing.)

#### 7f. New function `RunPhase5Tests` (non-Pure, no inputs)

Sample values are function locals; store each result before using it. `Y = Cfg.PlayPlaneY`; `V = 130.0` (a literal test speed). All position tolerances are 0.5 units. **Every explosion a test creates is frozen right after the call and destroyed by the test.** Each test ends with everything standing again. All tests are silent (`bLog = false`).

```
T27 city damage (stepped)
    cBefore = GM.CityHits ; bBefore = GM.BaseHits ; xBefore = GM.ExplosionCount ; aBefore = Length(GM.ActiveExplosions)
    alive0 = GM.IsSlotAlive(3)                                  // true at the start
    r1 = GM.DamageSlot(3, false) ; EX = GM.LastExplosion ; EX.SetActorTickEnabled(false)
    c = GM.Cities[2]                                            // slot 3 is city index 2
    alive1 = GM.IsSlotAlive(3) ; ch1 = GM.CityHits ; ls = GM.LastDamageSlot ; x1 = GM.ExplosionCount ; xp = GM.LastExplosionPos
    bodyShown = c.GetBodyShown() ; rubbleShown = c.GetRubbleShown()
    side = GM.IsSlotAlive(2) AND GM.IsSlotAlive(5) AND GM.IsSlotAlive(4)      // neighbours untouched
    r2 = GM.DamageSlot(3, false)                                // the same slot again: nothing may happen
    ch2 = GM.CityHits ; x2 = GM.ExplosionCount ; go = GM.bGameOver
    Destroy Actor (EX) ; GM.RestoreAll(false)
    ok = alive0 AND r1 AND NOT alive1 AND ch1 == cBefore + 1 AND ls == 3 AND x1 == xBefore + 1
         AND Distance(xp, GM.GetSlotPos(3)) <= 0.5 AND NOT bodyShown AND rubbleShown AND side
         AND NOT r2 AND ch2 == ch1 AND x2 == x1 AND NOT go
         AND GM.BaseHits == bBefore AND Length(GM.ActiveExplosions) == aBefore
    If ok → TestPassCount++
    DDLog("T27 city damage: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)

T28 base damage (stepped)
    bBefore = GM.BaseHits ; xBefore = GM.ExplosionCount ; aBefore = Length(GM.ActiveExplosions)
    r = GM.DamageSlot(0, false) ; EX = GM.LastExplosion ; EX.SetActorTickEnabled(false)
    B = GM.Bases[0]
    d1 = B.Ammo ; d2 = B.GetAmmoDisplayText() ; d3 = B.bDestroyed ; bodyShown = B.GetBodyShown() ; rubbleShown = B.GetRubbleShown()
    bh = GM.BaseHits ; ls = GM.LastDamageSlot ; x1 = GM.ExplosionCount
    B.SetAmmo(7) ; d4 = B.Ammo                                  // a destroyed base cannot get ammo
    GM.RefillBases(false) ; d5 = B.Ammo ; d6 = GM.Bases[1].Ammo ; d7 = GM.Bases[2].Ammo
    sel = SelectBase((-1000, Y, 900))                           // Left would be nearest; it is destroyed, so Center (index 1)
    go = GM.bGameOver
    Destroy Actor (EX) ; GM.RestoreAll(false)
    ok = r AND d1 == 0 AND d2 == "0" AND d3 AND NOT bodyShown AND rubbleShown
         AND bh == bBefore + 1 AND ls == 0 AND x1 == xBefore + 1
         AND d4 == 0 AND d5 == 0 AND d6 == Cfg.AbmStartAmmo AND d7 == Cfg.AbmStartAmmo
         AND sel == 1 AND NOT go AND Length(GM.ActiveExplosions) == aBefore
    If ok → TestPassCount++
    DDLog("T28 base damage: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)

T29 target filtering (stepped)
    GM.Cities[0].Demolish() ; GM.Cities[1].Demolish() ; GM.Cities[2].Demolish() ; GM.Cities[3].Demolish() ; GM.Cities[4].Demolish()
    GM.Bases[0].Demolish() ; GM.Bases[1].Demolish()             // direct and quiet: no explosions, no game-over check
    // standing now: slot 7 (city index 5) and slot 8 (right base)
    nBefore = Length(GM.ActiveEnemies) ; allOK = true ; saw7 = false ; saw8 = false
    Repeat 40 times (For Loop 0..39):
        e = GM.SpawnRandomEnemy(false) ; e.SetActorTickEnabled(false)
        s = e.SlotIndex
        slotOK   = (s == 7 OR s == 8)
        targetOK = Distance(e.Target, GM.GetSlotPos(s)) <= 0.5
        allOK = allOK AND slotOK AND targetOK
        If s == 7 → saw7 = true ; If s == 8 → saw8 = true
    cnt = Length(GM.ActiveEnemies)
    GM.ClearEnemies(false) ; GM.RestoreAll(false)
    ok = allOK AND saw7 AND saw8 AND cnt == nBefore + 40 AND Length(GM.ActiveEnemies) == nBefore
         AND GM.IsSlotAlive(0) AND GM.IsSlotAlive(1)
    If ok → TestPassCount++
    DDLog("T29 target filtering: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)

T30 game over (stepped)
    gBefore = GM.GameOverCount ; xBefore = GM.ExplosionCount ; aBefore = Length(GM.ActiveExplosions)
    sBefore = GM.EnemiesSpawned ; fcBefore = FireCount ; enBefore = Length(GM.ActiveEnemies)
    E1 = GM.SpawnEnemy((0, Y, 1800),   GM.GetSlotPos(8), V, 8, false) ; E1.SetActorTickEnabled(false)
    E2 = GM.SpawnEnemy((300, Y, 1800), GM.GetSlotPos(0), V, 0, false) ; E2.SetActorTickEnabled(false)
    nSpawn = Length(GM.ActiveEnemies)                           // enBefore + 2 (both still standing targets: bases 8 and 0)
    GM.Cities[0].Demolish() ; GM.Cities[1].Demolish() ; GM.Cities[2].Demolish() ; GM.Cities[3].Demolish() ; GM.Cities[4].Demolish()
    pre = GM.bGameOver                                          // five cities down is NOT game over
    r6 = GM.DamageSlot(7, false) ; EX = GM.LastExplosion ; EX.SetActorTickEnabled(false)     // the sixth city: game over
    g = GM.bGameOver ; gc = GM.GameOverCount ; ne = Length(GM.ActiveEnemies) ; xN = GM.ExplosionCount
    basesOK = GM.IsSlotAlive(0) AND GM.IsSlotAlive(4) AND GM.IsSlotAlive(8)                  // game over came from the cities only
    allowedNow = GM.IsRestartAllowed()                          // lockout: false right after game over
    sp = GM.SpawnRandomEnemy(false) ; spN = Length(GM.ActiveEnemies) ; spC = GM.EnemiesSpawned
    RequestFire(0, 0)                                           // inside the lockout: must neither fire nor restart
    fcAfter = FireCount ; stillOver = GM.bGameOver ; rs = bRestarting
    Destroy every BP_ABM (Get All Actors Of Class) ; Dismiss every BP_TapMarker ; GM.RefillBases(false)     // only matters if the guard is broken
    FireCount = 0 ; LastFirePos = (0,0,0) ; LastFireTime = -1000
    Destroy Actor (EX) ; GM.RestoreAll(false)
    ok = NOT pre AND r6 AND g AND gc == gBefore + 1 AND nSpawn == enBefore + 2 AND ne == 0 AND xN == xBefore + 1
         AND basesOK AND NOT allowedNow AND NOT IsValid(sp) AND spN == 0 AND spC == sBefore + 2
         AND fcAfter == fcBefore AND stillOver AND NOT rs
         AND NOT GM.bGameOver AND Length(GM.ActiveExplosions) == aBefore
    If ok → TestPassCount++
    DDLog("T30 game over: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)

T31 restore all (stepped)
    GM.Cities[0].Demolish() ; GM.Cities[5].Demolish() ; GM.Bases[1].Demolish()
    GM.Bases[0].SetAmmo(3)                                      // a partly used, still standing base
    GM.GameOver(false)
    g1 = GM.bGameOver
    GM.RestoreAll(false)
    g2 = GM.bGameOver
    cityOK = true ; For i in 0..5: c = GM.Cities[i] ; cityOK = cityOK AND IsValid(c) AND NOT c.bDestroyed AND c.GetBodyShown() AND NOT c.GetRubbleShown()
    baseOK = true ; For i in 0..2: b = GM.Bases[i] ; baseOK = baseOK AND NOT b.bDestroyed AND b.Ammo == Cfg.AbmStartAmmo
                                    AND b.GetAmmoDisplayText() == ("" + Cfg.AbmStartAmmo) AND b.GetBodyShown() AND NOT b.GetRubbleShown()
    slotOK = true ; For i in 0..8: s = GM.IsSlotAlive(i) ; slotOK = slotOK AND s
    badOK  = NOT GM.IsSlotAlive(9) AND NOT GM.IsSlotAlive(-1)
    ok = g1 AND NOT g2 AND cityOK AND baseOK AND slotOK AND badOK AND NOT GM.IsRestartAllowed()
    If ok → TestPassCount++
    DDLog("T31 restore all: " + (ok ? "PASS" : "FAIL"), "Test", NOT ok)

GM.ClearEnemies(false) ; GM.RestoreAll(false) ; GM.RefillBases(false)          // leave a clean state
```
(Store every `g`, `gc`, `ne`, `xN`, `sp`, `spN`, `spC`, `fcAfter` value in its own local at the point shown, before running the next line. In T30 the game-over call happens inside `DamageSlot`; the test never calls `GameOver` directly. T31 does call `GameOver(false)` directly, on purpose.)

**Check:** `RunPhase5Tests` compiles; there is no call to `RestartGame`, `StartSpawner` or `StopSpawner` in it.

#### 7g. `Event BeginPlay`: the self-test chain
Insert `RunPhase5Tests()` **right after `RunPhase4Tests()`**. The chain is now:

`Branch(bSelfTest)`
- **True** → `Delay(0.5)` → `UpdateArea(Get Viewport Size)` → `Delay(0.5)` → `RunSelfTest()` → `RunPhase3Tests()` → `RunPhase4Tests()` → **`RunPhase5Tests()`** → `SelfTestFlightStart()` → `Delay(0.5)` → `SelfTestFlightMid()` → `Delay(2.5)` → `SelfTestFlightEnd()` → `Delay(2.0)` → `SelfTestExplosionEnd()` → `SelfTestEnemyStart()` → `Delay(3.5)` → `SelfTestEnemyEnd()`
- **False** → `GM.StartSpawner`

**Check:** compiles clean; keys `H`, `R`, `E`, `K`, `X`, `V` exist and no others; `RequestFire` starts with the game-over branch; the chain matches 7g; `StartSpawner` is still called from exactly two places.

---

### Step 8 — `BP_DDHUD` and `BP_PlayfieldDebug`

**8a. `BP_DDHUD`.** Change **line 1** to: `"DOME DEFENSE | PHASE 5 | CITIES AND GAME OVER"`. Keep lines 2 to 4. **Add after line 4** (guard: `OwnerPC` valid and `OwnerPC.GM` valid; below, `GM` means `OwnerPC.GM`):
```
If GM.bGameOver:
    t1 = "GAME OVER"
    (w1, h1) = Get Text Size(t1, Font = None, Scale = 6.0)
    Draw Text(t1, Color (1.0, 0.25, 0.25), X = SizeX * 0.5 - w1 * 0.5, Y = SizeY * 0.40 - h1 * 0.5, Font = None, Scale = 6.0, Scale Position = false)
    ar = GM.IsRestartAllowed()
    If ar:
        t2 = "TAP OR CLICK TO RESTART"
        (w2, h2) = Get Text Size(t2, Font = None, Scale = 2.0)
        Draw Text(t2, White, X = SizeX * 0.5 - w2 * 0.5, Y = SizeY * 0.40 + h1 * 0.5 + 30, Font = None, Scale = 2.0, Scale Position = false)
```
(`SizeX` and `SizeY` are the outputs of `Event ReceiveDrawHUD`.)

**Fallback:** if `Get Text Size` is not available, use the approximate widths `w1 = 9 * 8.5 * 6.0` and `w2 = 23 * 8.5 * 2.0`, and heights `h1 = 14 * 6.0`. Record it.

**8b. `BP_PlayfieldDebug`.** The base boxes (cyan) and the city boxes (green) of the overlay are drawn by `Event Tick`. They would now sit on top of the real actors (and a green box over rubble would lie). **Delete these two elements** (Phase 1.1 table, rows 6 and 7). Keep rows 1 to 5 (visible-area frame, guaranteed-zone frame, ground line, two axis gizmos), `bShowOverlay`, `ToggleOverlay` and the `BeginPlay` log.

**Fallback:** if the nodes cannot be removed cleanly, add a boolean variable `bShowSlotMarkers` (default **false**) and put a `Branch` in front of both elements so they draw only when it is true. Record it.

**Check:** both compile clean; the overlay still draws the yellow frame, the orange frame, the white ground line and the two gizmos.

---

### Step 9 — Save and run

1. **Save All** (save the level before any level load).
2. `StartPIE`. The self-test runs for about **10 seconds**, then the spawner starts. Wait about **18 seconds**.
3. Take `CaptureEditorImage` about once per second from about 6 s to 10 s (the live enemy A reaches CITY3 at about 7 s and its rubble stays until about 9.5 s), one at about 12 s (the city is green again), and one at about 14 s and one at about 17 s (an automatic enemy is on screen).
4. `GetLogEntries` (filter `[DD]`), then `StopPIE`.
5. Fill in the report (section 8).

---

## 7. Acceptance tests

### A. Static checks
- [ ] **A1** New assets: `BP_City`, `MI_DD_City`, `MI_DD_Rubble`. No others.
- [ ] **A2** Every changed Blueprint compiles with 0 errors and 0 warnings.
- [ ] **A3** `DA_DDConfig.GameOverRestartDelay = 2.0`; `AbmStartAmmo`, `EnemyBaseSpeed`, `EnemySpawnInterval` recorded; no other value changed.
- [ ] **A4** `BP_City` has `Body` and `Rubble` (both Y 60, no collision), the functions `ApplyVisual`, `InitCity`, `Demolish`, `Rebuild`, `GetBodyShown`, `GetRubbleShown`, and both tick flags false.
- [ ] **A5** `BP_MissileBase` has `Rubble`, `bDestroyed`, `ApplyVisual`, `Demolish`, `Rebuild`, `GetBodyShown`, `GetRubbleShown`, the guarded `SetAmmo`, and `AmmoText` is still written only inside `SetAmmoDisplay` (search the graphs).
- [ ] **A6** `BP_DDGameMode` has the seven new variables and the functions `SlotToBaseIndex`, `SlotToCityIndex`, `CityIndexToSlot`, `IsSlotAlive`, `CountAliveCities`, `CountAliveBases`, `DamageSlot`, `CheckGameOver`, `GameOver`, `IsRestartAllowed`, `RestoreAll`, `DestroyRandomCity`, `SpawnCities`; `BeginPlay` calls `SpawnBases` then `SpawnCities`.
- [ ] **A7** `DamageSlot` is called only from `BP_EnemyMissile.Impact` (after `Destroy Actor`, with locals), `DestroyRandomCity`, and the controller's test functions. `GameOver` is called only from `CheckGameOver` and the controller's `RunPhase5Tests`. `StopSpawner` is **not** called from anywhere new. `StartSpawner` is called from exactly two places.
- [ ] **A8** The controller has key events `H`, `R`, `E`, `K`, `X`, `V` and no other keys. `RequestFire` begins with the `bGameOver` branch. `RestartGame` exists and is called only from `RequestFire`.
- [ ] **A9** `BP_PlayfieldDebug` no longer draws base or city boxes (or hides them behind `bShowSlotMarkers = false`, recorded as a deviation).
- [ ] **A10** No actors were added to the level (cities spawn at runtime). `BP_ABM`, `BP_Explosion` and the self-test chain order (apart from the new `RunPhase5Tests`) are unchanged.

### B. Log checks (agent)
Order is not guaranteed. Wait about 18 seconds.

- [ ] **B1** The Phase 0 to 4 startup lines still appear (GameMode, Config OK `Width=2250.0 Height=1800.0 ...`, Camera, PlayerController, View target, Crosshair spawned, HUD, PlayfieldDebug, `[DD][Area] Viewport=...`), `Bases spawned n=3`, and **`[DD][Core] Cities spawned n=6`**.
- [ ] **B2** `T1` to `T6`, `T7 fire path`, and `T8` to `T21` are PASS exactly as in Phase 4 (including `T16`–`T19` and the Phase 3 explosion lines for X=1000 Z=1700).
- [ ] **B3** `T22 enemy flight and impact: PASS`, `T23 kill, chain reaction and control: PASS`, `T24 clear enemies: PASS`.
- [ ] **B4** `T27 city damage: PASS`, `T28 base damage: PASS`, `T29 target filtering: PASS`, `T30 game over: PASS`, `T31 restore all: PASS`.
- [ ] **B5** `[DD][Enemy] Spawn slot=CITY3 fromX=-300 toX=-252` and `[DD][Enemy] Spawn slot=BASE-CENTER fromX=0 toX=0`, each once, **before** `SelfTest done`. The synthetic enemies of T21 to T24 and T29 to T30 log nothing.
- [ ] **B6** `[DD][Input] Fire request base=CENTER X=0 Z=900 n=1 ammo=9` and `[DD][ABM] Arrived base=CENTER X=0 Z=900`.
- [ ] **B7** `[DD][Enemy] Killed slot=BASE-CENTER X=0 Z=<n>` with `n` between 740 and 960, and `[DD][Enemy] Impact slot=CITY3 X=-252 Z=100`, each once.
- [ ] **B8** Exactly **one** `[DD][Damage] City destroyed slot=CITY3 left=5` line, and **no other** `[DD][Damage]` line in the whole play window (the synthetic tests are silent, and an automatic enemy cannot land before about 25 s).
- [ ] **B9** Before `SelfTest done`, exactly **four** `[DD][Explosion] Start` lines (at X=1000 Z=1700; at X=0 Z=900; the enemy's own at X=0 Z=`n` from B7; and the impact explosion at **X=-252 Z=100 max=160**) and exactly **four** `[DD][Explosion] End` lines, including `End X=-252 Z=100 peak=<n> life=<n>ms` with `peak` between 80 and 160.
- [ ] **B10** `T25 enemy live impact: PASS` and `T26 enemy killed by explosion: PASS killZ=<n>`
- [ ] **B11** `[DD][Test] SelfTest done: 31/31 PASS`
- [ ] **B12** **No** line with the category `[DD][Game]` anywhere (game over, restore and restart happen only on human input; the self-test is silent).
- [ ] **B13** `[DD][Enemy] Spawner started interval=2500ms` exactly once, **after** `SelfTest done`; then at least one more `[DD][Enemy] Spawn slot=...` line.
- [ ] **B14** No line containing `ERROR`, `Accessed None`, or `Blueprint Runtime Error`.

### C. Visual checks (editor image of the play view; NEEDS-HUMAN if you cannot capture it)
- [ ] **C1** At rest (before 6 s or after 10 s): black background, no bars, frames, brown ground, **six green city blocks** and three gray-blue bases with numbers `10`. **No cyan and no green outline boxes.**
- [ ] **C2** In at least one capture between about 7 s and 9.5 s, the third city from the left (X about -252) is **flat dark-gray rubble** instead of a green block, and an orange disc is around it. If no capture caught it, mark **NEEDS-HUMAN**; do **not** claim it.
- [ ] **C3** In a capture after about 10 s, all six cities are **green blocks again** (the test restored them).
- [ ] **C4** In a capture at about 14 s or later, at least one **red dot** is on screen. If not caught, **NEEDS-HUMAN**.
- [ ] **C5** HUD line 1 reads `DOME DEFENSE | PHASE 5 | CITIES AND GAME OVER`. No GAME OVER text is on screen.

### D. Interactive checks (human, PIE; click inside the viewport first; `Shift+F1` frees the mouse, `Esc` stops PIE; **wait about 10 seconds after Play so the self-test is finished; do not click or press keys before that**)
- [ ] **D1** Let an enemy reach a city (or press `X`): the green block is replaced by **flat dark rubble**, an **orange explosion** flashes at the impact point, and the log shows `City destroyed ... left=<n>`.
- [ ] **D2** Let an enemy reach a base: the base body is replaced by rubble, its number reads **0**, it never fires again, and `R` does **not** bring it back. A click near it is fired by the next nearest base. If all bases are gone, a click does nothing (log: `No ammo`) and the game continues until the cities fall.
- [ ] **D3** Targeting: after a city is rubble, newly spawned enemies never head for it (watch several, or press `E` repeatedly). An enemy already flying toward a slot that falls meanwhile lands on the rubble with nothing happening (no explosion; log `Already destroyed` only for real enemies).
- [ ] **D4** An impact explosion that overlaps another enemy kills it, and that kill spawns its own explosion.
- [ ] **D5** Game over: press `X` until all six cities are rubble (or let enemies do it). The moment the last city falls: all enemies disappear without explosions, no new enemy appears, **GAME OVER** appears large and red in the middle, and a click in the **first 2 seconds does nothing**. After 2 seconds **TAP OR CLICK TO RESTART** appears; a click reloads the level (the 10-second self-test runs again; do not click during it) and the game starts fresh with six cities, three bases, ammo 10.
- [ ] **D6** `V` rebuilds all cities and bases (ammo 10) and removes GAME OVER **without reloading**; enemies start coming again within about 2.5 s. Pressing `V` during normal play is harmless. `X` and `V` log `[DD][Damage]` and `[DD][Game]` lines.
- [ ] **D7** Layering: red enemies are drawn over cities and rubble; explosions over both; the blue ABM dots over enemies.
- [ ] **D8** Resize the window (wide, 5:4, nearly square): cities, rubble and bases stay on the ground at their positions, and the GAME OVER text stays centered.
- [ ] **D9** Phase 2 to 4 behavior unchanged: base choice, ammo numbers, `R`, `E`, `K`, `H`, crosshair, explosions, chain reactions.
- [ ] **D10** Feel (no pass/fail): note how fast the game is lost at the current speed and interval.

### D-DEVICE. Touch (carried from Phase 1.1; still open)
- [ ] **T-1** A tap launches from the nearest base as before.
- [ ] **T-2** A second finger is ignored.
- [ ] **T-3** One tap makes exactly one launch.
- [ ] **T-4** After game over, a tap during the first 2 seconds does nothing; a tap after that restarts the game. The GAME OVER text is readable on the phone.

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped.

---

## 8. Report format (the agent's final message)

```
## Phase 5 report
**Engine version:** x.y.z
**MCP tools used:** (list)

### Assets created or changed
(table: path | type | change | compiled clean)

### Config values read
(GameOverRestartDelay, AbmStartAmmo, EnemyBaseSpeed, EnemySpawnInterval)

### Acceptance results
A1 PASS/FAIL ... through E1 PASS/FAIL/NEEDS-HUMAN/NEEDS-DEVICE

### Log excerpt
(all [DD] lines from PIE, including every [DD][Test], [DD][Enemy], [DD][Damage], [DD][Game] and [DD][Explosion] line)

### Screenshot notes
(what the cities, rubble and the orange disc looked like, when each capture was taken)

### Deviations
### Open questions / risks
```

---

## 9. What this hands to Phase 6

| Provided | How Phase 6 uses it |
|---|---|
| `GM.DamageSlot(Slot, bLog)` | The single entry point for damage. Phase 6 leaves it alone. |
| `GM.IsSlotAlive`, `CountAliveCities`, `CountAliveBases` | End-of-wave bonus (surviving cities), bonus-city rules, wave logic. |
| `BP_City.Rebuild`, `BP_MissileBase.Rebuild(StartAmmo)`, `GM.RestoreAll` | Rebuilding bases and awarding bonus cities between waves (call the per-object `Rebuild`, not `RestoreAll`). |
| `GM.bGameOver`, `GameOver`, `IsRestartAllowed`, `GameOverTime` | Phase 7 replaces the reload-on-click with proper game flow and keeps these as the state. |
| `SpawnRandomEnemy` blocking during game over | The wave spawner must respect `bGameOver` the same way. |
| `GM.CityHits`, `BaseHits`, `LastDamageSlot`, `GameOverCount` | Test hooks. |
| Debug keys `X`, `V` | Stay until Phase 7; `N` (next wave) is added in Phase 6. |
| Self-test chain (sync tests, then live segments) | Phase 6 adds scoring and wave tests the same way. |

---

## 10. Notes for the human (not agent steps)

1. **The game can now be lost.** The first waves are easy at the current speed and spawn interval; tune `EnemyBaseSpeed` and `EnemySpawnInterval` in `DA_DDConfig`, or wait for the wave phase.
2. **Nothing is rebuilt during play.** Destroyed cities and bases stay rubble until `V` or a restart. `R` skips destroyed bases. Rebuilding and bonus cities arrive with Phase 6.
3. **Game over counts only cities.** With all bases destroyed and at least one city standing, you cannot shoot but the game goes on.
4. **The spawner idles at game over** instead of stopping. This is on purpose: it lets `V` resume the game without a second spawn loop.
5. **A restart reloads the level**, so the 10-second self-test runs again; do not click during it. It is dropped later. `X` and `V` are desktop debug keys and do not exist on Android.
6. **The overlay (`H`) no longer draws base and city boxes**: the real actors replace them.
7. **Cities and rubble are placeholder blocks.** The art pass replaces them (the intact and rubble components, `ApplyVisual`); the logic stays.
8. **Live segment changed slightly:** the test enemy aimed at CITY3 now starts lower (Z 200 instead of 300) so its explosion finishes before the check. The log line `fromX=-300 toX=-252` is unchanged.
9. **Touch is still untested.** **Commit** the accepted state when you are happy.

---

## 11. Common pitfalls

1. **Calling `DamageSlot` before `Destroy Actor` in `Impact`.** A game over then clears the enemy that is just landing, and it is destroyed twice. Use the three locals and call `DamageSlot` last.
2. **`SetAmmo` without the guard.** `R` (and `RefillBases`) would revive a destroyed base.
3. **`StopSpawner` plus `StartSpawner` for game over and `V`.** A pending `Delay` makes two loops. Use the `bGameOver` check in `SpawnRandomEnemy` only.
4. **`RestoreAll` starting the spawner.** It must not: the self-test calls it while the spawner is off. `StartSpawner` stays in its two places.
5. **Test explosions left ticking or alive.** Freeze each one right after the call (`SetActorTickEnabled(false)`) and destroy it, or T19 and the live explosion counts break.
6. **Reading `left` before `Demolish`.** The count must exclude the object just destroyed.
7. **Random numbers read twice** in `SpawnRandomEnemy` and `DestroyRandomCity`. Store the index and the X first.
8. **Taking the `[DD][Game]` lines out of the tests' silence.** All test calls pass `bLog = false`; B8 and B12 depend on that.
9. **Calling `RestartGame` from a test.** It reloads the level and kills the self-test. Test the lockout through `RequestFire` while `IsRestartAllowed` is false only.
10. **Placing the guard in `RequestFire` after the debounce or after `ScreenToPlane`.** It must be the first thing in the function.
11. **Leaving the cyan and green overlay boxes.** A green box over rubble makes a destroyed city look alive.
12. **Using `IsValid` on an actor destroyed in the same frame.** Use `Length(GM.ActiveEnemies)` or `Length(GM.ActiveExplosions)`.
13. **A test that leaves something destroyed.** Every test ends with everything standing (the `RestoreAll` line); otherwise the next test starts in a damaged state.
