# Phase 7 — Game Flow, Pause, High Score and Self-Test Removal

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Depends on** | Phase 6 and patch 6.1 (accepted: agent checks 41/41; desktop play-tests verified waves, tally, bonus cities, kill origin, `N`, `V`, `K`, the restart click and window resize; touch on a device is still open and is not repeated here) |
| **Phase goal** | The game gets a real flow instead of "reload the level on click". A **title screen** shows the high score and waits for a tap or click. A tap starts a new game (wave 1, score 0) **without reloading the level**. A small **pause button** (top right) pauses the game; the pause screen has **Resume** and **Quit to title**. After **game over** the screen shows the final score and the high score, and a tap (after the existing 2-second lockout) returns to the title. The **high score** is saved on the device (a SaveGame slot) and shown on the title screen, during play (`HI`) and on the game-over screen. Desktop: key `P` toggles pause, key `Delete` resets the saved high score. Android: the Back button pauses, goes to the title, or quits. **The self-test is deleted**, together with its test-only variables and the dead code listed in Step 9. The debug overlay and the debug keys stay. |

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **No bars, no aspect caps.** The whole window is the playfield. Do not add any.
3. **Existing logic must survive** unless a step says to change or delete it. Read each Blueprint's current graphs (`read_graph_dsl`) before editing.
4. **Compile after every Blueprint change.** Every Blueprint ends with no errors and no warnings.
5. **Run each step's "Check"** before moving on. **Save All after each step's Check passes.**
6. **You cannot press keys, click, touch, or resize in PIE.** You therefore **cannot exercise** the flow (tap to start, pause, resume, quit, game over, saving). What you can verify: compiles, static inspection of the graphs, the startup log, and captures of the play view. Interactive checks (group D) are **NEEDS-HUMAN**; device checks (group D-DEVICE) are **NEEDS-DEVICE**. Never mark a D or D-DEVICE item PASS.
7. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
8. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after each run. **Save before loading a level.**
9. **Read sections 4 and 5 before building graphs.**
10. **This phase adds no config values and exactly one asset: `BP_DDSave`.** No other new asset. No GameInstance Blueprint, no UMG widget, no Project Settings change, no change to `DA_DDConfig`.
11. **NEVER REWRITE AN EXISTING FUNCTION OR EVENT GRAPH.** `write_graph_dsl` replaces a graph by deleting all its nodes and rebuilding it. On a big graph this ran the editor out of memory in Phase 5 and the editor crashed. Therefore:
    - **A brand-new function** (it has no nodes yet) may be written with `write_graph_dsl`, in **one small write**. Keep each new function to **about 40 nodes at most**. Functions in this document that look bigger are already split into helpers: build the helpers as separate functions. (The 40 is a guess; if a write is slow or fails, build smaller pieces.)
    - **An existing graph** (anything listed as "edit in place") is changed **only** by small node edits: `create_node`, `connect_pins`, set a pin default value, delete a single node. **Never** `write_graph_dsl` on it. This includes the event graphs of `BP_DDGameMode` and `BP_DDPlayerController`: new events and key handlers there are made with `create_node` and `connect_pins`.
12. **TEST FUNCTIONS: until Step 9, do not open them for writing, do not edit them, do not rewrite them.** All in `BP_DDPlayerController`: `RunSelfTest`, `RunPhase2Tests`, `RunPhase3Tests`, `RunPhase4Tests`, `RunPhase4Rest`, `RunPhase5Tests`, `RunPhase6Tests`, `RunT32` to `RunT41`, `BonusCase`, `FreezeEnemies`, `CapRoomCase`, `WaveSpeedCase`, `OriginCaseA`, `OriginCaseB`, `OriginCaseC`, `SelfTestFlightStart`, `SelfTestFlightMid`, `SelfTestFlightEnd`, `SelfTestExplosionEnd`, `SelfTestEnemyStart`, `SelfTestEnemyEnd`, `ReportTest`. In Step 9 they are **deleted, never edited**. If a test fails in Run 1 (Step 7), **stop and report**; do not "fix" a test.
13. **After any tool error (DSL error, timeout, "not found"), do not retry the same large write.** Stop that item, try one smaller piece at most, and if it still fails, **stop and report** under *Tool problems*. **If the editor stops responding or restarts, stop and report.** Do not continue with a half-built graph.
14. **The old startup (self-test, then spawner) must keep working until Step 8.** Run 1 (Step 7) proves it still passes with all the Phase 7 additions in place. Only Step 8 changes the startup.
15. **Deleting (Step 9) has its own rules** (one item per tool call, callers before callees, compile every five deletions, no graph rewrites). Follow them exactly.

---

## 2. Scope

### In scope
- New asset `BP_DDSave` (a SaveGame Blueprint holding the high score).
- `BP_DDGameMode`: flow state, high score (load, save, commit, reset), pause helper, field cleanup, `EnterTitle`, `StartNewGame`, `PauseGame`, `ResumeGame`, `QuitToTitle`, button rectangle helpers, `FinishGame`; one inserted call in `GameOver`; the `Skipped` log fix in `SkipWave`.
- `BP_DDPlayerController`: tap handling for the new states, `BeginGame`, `TogglePause`, `HandleBack`, keys `P`, `Delete`, Android Back, one inserted call at the top of `RequestFire`, one swapped call inside the existing game-over branch of `RequestFire`, the new startup chain.
- `BP_DDHUD`: title screen, pause button, pause screen, `HI` line, game-over info, new title line.
- Deleting the self-test and the other dead code listed in Step 9.

### Out of scope (do NOT create)
- Any UMG, any widget Blueprint, any new actor in the level.
- Sounds, particles, imported art, an attract mode, an options or settings menu, a leaderboard, initials entry, cloud saves.
- Any change to waves, scoring, damage, explosions, enemies, ABMs, the camera or the visible-area logic.
- Any new config value; any Project Settings change; a GameInstance Blueprint.
- Any key besides `P`, `Delete` and the Android Back key (new). The existing keys `H`, `R`, `E`, `K`, `X`, `V`, `N` stay exactly as they are.
- Removing: the debug overlay (`BP_PlayfieldDebug`, key `H`), the HUD lines 2 to 4, `StopSpawner`, `bDirectorSilent` and the places that test it. (They stay; see section 9.)

---

## 3. Design (decided; do not change)

### 3.1 Flow states
`GM.FlowState` (integer): **0 Title**, **1 Playing**, **2 Paused**.
- State 1 includes the wave intro, the tally and the game-over screen. **Game over is not a flow state**: it is `bGameOver` inside state 1 (Phase 5 fact, unchanged).
- The variable's **default is 1**, so the old startup keeps working until Step 8. Step 8 makes the startup call `EnterTitle`, which sets 0.

| From | Event | To | Done by |
|---|---|---|---|
| startup | `BeginPlay` | 0 | `GM.EnterTitle` |
| 0 Title | tap or click, at least 0.5 s after the title appeared | 1 | controller `BeginGame` (calls `GM.StartNewGame`, then starts wave 1) |
| 1 Playing, not game over | tap on the pause button, key `P`, Android Back | 2 | `GM.PauseGame` |
| 2 Paused | tap on Resume, key `P` | 1 | `GM.ResumeGame` |
| 2 Paused | tap on Quit to title, Android Back | 0 | `GM.QuitToTitle` (commits the score, then `EnterTitle`) |
| 1 Playing, game over | tap or click after the lockout (`IsRestartAllowed`), Android Back | 0 | `GM.EnterTitle` |

**The level is never reloaded.** `RestartGame` and `[DD][Game] Restart requested` disappear (Step 9).

### 3.2 Pause
- `ApplyPause(bPause)` is the **only** place that touches time: `Set Global Time Dilation(0.0001)` when pausing, `Set Global Time Dilation(1.0)` when resuming.
- Why not `Set Game Paused`: a paused game ignores input events unless every input node is flagged "Execute When Paused", and that flag may not be settable through the tools. With a dilation of 0.0001 the world is effectively frozen (enemies, ABMs, explosions, `Delay`, `Get Game Time In Seconds`, the director) while **input and the HUD keep running**, so the pause screen can be tapped.
- `EnterTitle` and `StartNewGame` both call `ApplyPause(false)` first, so a stray dilation can never survive a state change.

### 3.3 Title screen
- `EnterTitle` clears the field (enemies, ABMs in flight, explosions, tap markers), rebuilds every city and base (`RestoreAll`), resets the game state (`ResetGame`), loads the high score, sets `FlowState = 0`, records `TitleEnterTime`.
- The world stays visible behind the text: green cities, bases with ammo `10`, the ground. **No** score block, **no** wave banner, **no** pause button, **no** enemies, **no** spawner activity.
- A tap or click starts the game only if `Now - TitleEnterTime >= 0.5`. A tap inside that 0.5 s is consumed and does nothing. Taps never fire an ABM on the title.

### 3.4 Starting a game
Controller `BeginGame`: `GM.StartNewGame(true)` (stops any pause, clears the field, rebuilds everything, `ResetGame`, `FlowState = 1`, `bNewHighScore = false`), then:
- if `GM.bSpawnerRunning` is true → `GM.StartWave(1, Get Game Time In Seconds)` (the director loop already runs and is never stopped);
- else → `GM.StartSpawner` (first game after launch: it does `ResetGame`, `StartWave(1, now)` and starts `DirectorLoop`; unchanged from Phase 6).

`BeginGame` is the **only** caller of `StartSpawner` once Step 9 is done. If `Config.bEnemyAutoSpawn` is false, `StartSpawner` does nothing and the game starts empty (as in Phase 4): this is accepted.

### 3.5 Game over and the high score
- Game over itself is unchanged (Phase 5: enemies cleared, `bGameOver = true`, lockout `GameOverRestartDelay`).
- `GameOver` now ends with one call, `FinishGame(bLog)`: it commits the score (`CommitScore`) and logs one `Flow` line.
- **`CommitScore`**: does nothing if `bDirectorSilent` is true or `Wave < 1` or `Score <= HighScore`. Otherwise `HighScore = Score`, `bNewHighScore = true`, `SaveHighScore`. (The `bDirectorSilent` and `Wave` guards are what keeps the old self-test from writing test scores into the real save file in Run 1.)
- `CommitScore` is also called by `QuitToTitle` (quitting from the pause screen saves a record, too). It is idempotent: calling it twice with the same score changes nothing.
- `bNewHighScore` is true when the **current game** set a record. It is cleared by `StartNewGame` and `EnterTitle`.
- The game-over screen: the existing **GAME OVER** (large red) and, after the lockout, the second line, now **TAP OR CLICK TO CONTINUE** (was `...TO RESTART`); below them `SCORE <n>` and either `NEW HIGH SCORE!` or `HIGH SCORE <n>` (3.9).
- A tap or click after the lockout calls `GM.EnterTitle(true)` (it replaces the old `RestartGame()` call inside the existing game-over branch of `RequestFire`).
- Debug key `V` after game over still restarts the same wave and keeps the score (Phase 6); this does not change the flow state.

### 3.6 Save slot
- Class `BP_DDSave` (parent `SaveGame`), one variable `HighScore` (integer, 0).
- Slot name: the game-mode variable `SaveSlotName` (string, default `DD_Save`), user index 0.
- `LoadHighScore` takes `Max(HighScore, file value)`, so a failed load can never lower the in-memory value.
- **Fallback if a SaveGame Blueprint cannot be created:** skip `BP_DDSave`; make `LoadHighScore` and `SaveHighScore` log one `Save` error line and do nothing else. The high score then lives only in `GM.HighScore` (it survives new games within one launch, because the level is not reloaded) and is lost when the app closes. Record the deviation.

### 3.7 Input rules (tap or click; one finger only, as before)
`RequestFire` first asks `HandleFlowTap`. If it returns true, the tap is consumed and nothing is fired.

| State | What a tap or click does | Consumed |
|---|---|---|
| 0 Title | after 0.5 s: `BeginGame` | always |
| 1 Playing, not game over | inside the **padded** pause button: `PauseGame`; anywhere else: fires as before | only the pause tap |
| 1 Playing, game over | not handled by `HandleFlowTap`; the existing game-over branch decides: lockout, then `EnterTitle` | no (the branch returns without firing) |
| 2 Paused | inside the Resume rectangle: `ResumeGame`; inside the Quit rectangle: `QuitToTitle`; elsewhere nothing | always |

Touch and mouse already call `RequestFire` (Phase 1.1): **do not change those events.**

### 3.8 Keys and the Android Back button
| Input | Title | Playing | Playing, game over | Paused |
|---|---|---|---|---|
| `P` (desktop) | nothing | pause | nothing | resume |
| Android Back | quit the app | pause | `EnterTitle` | `QuitToTitle` |
| `Delete` (desktop) | reset the saved high score | same | same | same |

- `Delete` → `GM.ResetHighScore` (testing aid; it does not exist on a device). Fallback key if the `Delete` event cannot be created: `U`. Record it.
- The Android Back key is `Android_Back` (a key event; find it with `find_node_types`, query "Android Back"). On the title it calls `Quit Game` (Quit Preference = Quit, Ignore Platform Restrictions = false). **If no such key event exists**, skip it and record the deviation.
- **Auto-pause when the app goes to the background:** look for the Blueprint events "Application Will Deactivate" / "Application Will Enter Background" with `find_node_types` and use one **only if it can be placed in `BP_DDPlayerController`'s event graph as it is**. If it exists only on a GameInstance or needs a new asset or a Project Settings change, **do not do it**; record it under *Deviations* as "auto-pause not available". If found: the event calls `GM.PauseGame(true)` (a no-op unless playing).
- All new key and Back events use the **Pressed** pin only.

### 3.9 HUD
Canvas drawing only (no UMG). `GM` means `OwnerPC.GM`. Touch and mouse positions arrive in viewport pixels and the HUD canvas (`SizeX`, `SizeY`) is in the same pixels: **never convert between them**.

- **Line 1 (title):** `DOME DEFENSE | PHASE 7 | GAME FLOW` (one space on each side of every pipe). Lines 2 to 4 and the GAME OVER block stay.
- **Title screen**, while `GM.FlowState == 0`:
  - `DOME DEFENSE` yellow `(1.0, 0.9, 0.2)`, scale 7, `Y = SizeY * 0.18`.
  - `HIGH SCORE <n>` white, scale 2.5, `Y = SizeY * 0.18 + 150`.
  - `TAP OR CLICK TO START` white, scale 3, `Y = SizeY * 0.58`, **blinking**: let `t = Get Game Time In Seconds`, `ph = FMod(t, 1.2)`; draw only while `ph < 0.8`.
- **Pause button**, while `FlowState == 1` and not `bGameOver`: a dark square `(0, 0, 0, 0.45)` with two white bars (the pause symbol), at the rectangle from `GetPauseButtonRect`.
- **Pause screen**, while `FlowState == 2`: a full-screen dim rectangle `(0, 0, 0, 0.6)`; `PAUSED` yellow, scale 5, `Y = SizeY * 0.22`; two buttons from `GetMenuButtonRect`: **RESUME** (fill `(0.15, 0.35, 0.15, 0.9)`) and **QUIT TO TITLE** (fill `(0.35, 0.12, 0.12, 0.9)`), labels white, scale 2.5, centered in their rectangles.
- **`HI` line**, while `FlowState == 1` and `Wave >= 1`: `HI <Max(HighScore, Score)>` gray `(0.75, 0.75, 0.75)`, scale 1.6, `Y = 85`, centered. The existing `BONUS CITIES` line of the score block moves from `Y = 85` to **`Y = 115`**.
- **Game-over info**, while `bGameOver`: `SCORE <n>` yellow, scale 2.5, `Y = SizeY * 0.40 + 150`; below it (`Y = SizeY * 0.40 + 200`) either `NEW HIGH SCORE!` green `(0.3, 1.0, 0.4)`, scale 2.5, when `bNewHighScore`, or `HIGH SCORE <n>` white, scale 2.0.
- The wave banner, the tally and the score block keep their Phase 6 behavior.

### 3.10 Button rectangles (one source for drawing and hit-testing)
All in viewport pixels. `ViewW`, `ViewH` are the viewport size as floats (the HUD passes `SizeX`, `SizeY`; the controller passes `Get Viewport Size`).

**Pause button** (`GetPauseButtonRect`): `mx = ViewW * 0.04`, `my = ViewH * 0.04`, `s = Clamp(ViewH * 0.09, 56, 140)`; `Bw = s`, `Bh = s`, `X = ViewW - mx - s`, `Y = my`. Its **hit area is the rectangle grown by `Pad = Bw * 0.25` on every side**.

**Menu buttons** (`GetMenuButtonRect`, `Which` 0 = Resume, 1 = Quit): `Bw = Max(ViewW * 0.32, 240)`, `Bh = Max(ViewH * 0.11, 70)`, `X = ViewW * 0.5 - Bw * 0.5`, `Y = ViewH * 0.42` for Resume and `ViewH * 0.60` for Quit. No padding. Any other `Which` returns all zeros.

### 3.11 Log lines
| Category | Line |
|---|---|
| `Save` | `[DD][Save] Loaded high=<n>` (file found and read) |
| `Save` | `[DD][Save] No file high=<n>` (no save file yet; `<n>` is the in-memory value) |
| `Save` | `[DD][Save] Saved high=<n>` |
| `Save` | `[DD][Save] High score reset` |
| `Save` | `[DD][Save] Load failed` / `[DD][Save] Save failed` (error lines, only on failure) |
| `Flow` | `[DD][Flow] Title high=<n>` (from `EnterTitle`) |
| `Flow` | `[DD][Flow] Start` (from `StartNewGame`) |
| `Flow` | `[DD][Flow] Pause` / `[DD][Flow] Resume` |
| `Flow` | `[DD][Flow] Quit to title score=<n> high=<n>` |
| `Flow` | `[DD][Flow] Game over score=<n> high=<n> new=<0 or 1>` (from `FinishGame`) |

Removed: `[DD][Game] Restart requested`, and every `[DD][Test]` line (the tests are deleted). All other Phase 0 to 6 lines stay as they are. Functions called by the old tests pass `bLog = false`, so the new `Flow` and `Save` lines never appear in Run 1.

### 3.12 Layering (unchanged)
Ground 0; bases, ammo numbers and cities +60; enemies +70; ABMs +80; explosions +90; crosshair +100; tap marker +110. The HUD draws on top of everything.

---
## 4. Conventions as built (these replace earlier text where they differ)

| Topic | As built |
|---|---|
| Camera | **Ortho Width is the vertical extent** (visible height). Do not change `ApplyArea`. |
| Visible area | `BP_DDGameMode` holds `VisibleMinX/MaxX/MinZ/MaxZ`. At narrow windows the visible height grows (more sky). |
| Reading values | A value read in a later node is evaluated **when that node runs**. **Store a value in a variable (or a local) before you change what it depends on.** This also applies to **random nodes** (each read rolls again) and to **loop bounds** (a pure expression used as a loop bound is re-evaluated; store it in a local first). |
| Config | Config functions and `GetConfig` are **not Pure**. The game mode uses its cached `Config`; the controller uses its cached `Cfg`. |
| Logging | Only through the static `DDLog(Message, Category, bIsError)`. Coordinates in logs are rounded integers. Phase 7 adds the categories `Save` and `Flow` (just strings). |
| Ticking | A Blueprint actor needs **Can Ever Tick = true** and **Start With Tick Enabled = true** for its Event Tick to run. |
| Controller as built | Variables `GM`, `Cfg`, `Crosshair`, `PlayfieldCamera`, `LastViewportW`, `LastViewportH`, `LastFireTime`, `FireCount`, `LastFirePos`, `LastMarker`, `bDesktopCrosshair`, `bSelfTest`, and the test variables. Functions `ScreenToPlane`, `ClampToPlayfield`, `UpdateArea`, `RequestFire(ScreenX, ScreenY)`, `LaunchAt`, `SelectBase`, `ConsiderBase`. Key events `H`, `R`, `E`, `K`, `X`, `V`, `N`; mouse-left and touch events (both call `RequestFire`; touch only for `Touch1`). |
| `RequestFire` as built | It begins with the Phase 5 branch: `If GM.bGameOver: (ar = GM.IsRestartAllowed ; If ar → RestartGame()) ; return`. The Phase 1.1/2 body (debounce, `ScreenToPlane`, `ClampToPlayfield`, `LaunchAt`) continues from the **False** pin of the first branch. |
| Startup chain as built | `Event BeginPlay`: valid-config branch → … → `Crosshair spawned` log → `Branch(bSelfTest)`. **True**: `Delay(0.5)` → `UpdateArea(Get Viewport Size)` → `Delay(0.5)` → `RunSelfTest()` → `RunPhase3Tests()` → `RunPhase4Tests()` → `RunPhase5Tests()` → `RunPhase6Tests()` → `SelfTestFlightStart()` → `Delay(0.5)` → `SelfTestFlightMid()` → `Delay(2.5)` → `SelfTestFlightEnd()` → `Delay(2.0)` → `SelfTestExplosionEnd()` → `SelfTestEnemyStart()` → `Delay(3.5)` → `SelfTestEnemyEnd()` (prints the summary, calls `GM.StartSpawner` last). **False**: `GM.StartSpawner`. Read the real graph first; it may contain extra nodes. |
| Game mode as built | State: `Cities`, `Bases`, `bGameOver`, `GameOverTime`, `Score`, `BonusCities`, `Wave`, `WaveState` (0 Idle, 1 Intro, 2 Running, 3 Tally, 4 Halted), `bDirectorSilent`, `bSpawnerRunning`. Functions used here: `ClearEnemies(bLog)` (output `Count`), `RestoreAll(bLog)`, `ResetGame`, `StartWave(WaveNumber, Now)`, `IsRestartAllowed`, `GameOver(bLog)`, `SkipWave`. Custom events: `StartSpawner` (guarded by `bSpawnerRunning`, does `ResetGame`, `StartWave(1, now)`, `DirectorLoop`), `DirectorLoop` (never stops), the dead `EnemySpawnLoop`. |
| Actors to clean | `BP_ABM` (flying rockets), `BP_Explosion` (also tracked in `GM.ActiveExplosions`; its `EndPlay` removes it from that list), `BP_TapMarker` (has `Dismiss()`), `BP_EnemyMissile` (cleared by `ClearEnemies`). |
| HUD as built | `Event ReceiveDrawHUD` (outputs `SizeX`, `SizeY`): lines 1 to 4 → `DrawWaveInfo(SizeX, SizeY)` → `If GM.bGameOver` block (**GAME OVER** red, scale 6, at 40% of the height; after the lockout the white line `TAP OR CLICK TO RESTART`, scale 2). Functions `DrawCentered(Text, Color, SizeX, Y, Scale)`, `DrawScoreBlock`, `DrawBanner`, `DrawTally`, `DrawWaveInfo`. `OwnerPC` is the controller; `OwnerPC.GM` the game mode. |
| Time | `Get Game Time In Seconds` follows time dilation. During the pause it advances 10000 times slower; that is intended. |
| Comment boxes / hooks | Comment boxes cannot be created. A "hook" is described by the node **after which** the new call goes. |
| Slow editor | "max tick rate 3" appears when the editor window is not focused. Do not tighten time tolerances. |

---

## 5. Tooling notes

1. **Do not guess node IDs.** Use `find_node_types`. Call `get_graph_dsl_docs` once.
2. **Key events cannot be written as DSL.** Use `create_node` + `connect_pins`, **Pressed** pin only. Known ID pattern: `Input|KeyboardEvents|H`; find `P` and `Delete` the same way. The Android Back key: `find_node_types` with "Android Back".
3. **Save-game nodes** (Gameplay Statics): `Does Save Game Exist`, `Create Save Game Object`, `Load Game From Slot`, `Save Game To Slot`, plus `Cast To BP_DDSave`. Time: `Set Global Time Dilation`. App: `Quit Game`. HUD: `Draw Rect`, `Draw Text`, `Get Text Size`. Find each with `find_node_types`.
4. **Creating `BP_DDSave`:** parent class `SaveGame` (`/Script/Engine.SaveGame`). Use AssetTools; find the right call with the tool list.
5. **Non-Pure functions need exec wires.** Call each function in sequence and store each result in its own local or variable, then combine. This applies to functions with several outputs (the rectangle helpers): store all four outputs in locals right after the call.
6. **Sequence pins** are `then_0`, `then_1`. **No nodes after a Branch or a Return** in the same chain. Where the pseudo-code shows an `If` followed by more steps, either put the `If` in a small helper function or connect **both branch ends to the same next exec pin** (an exec input accepts several wires).
7. **Latent nodes (`Delay`) only work in event graphs**, not in functions. This phase adds no new `Delay`.
8. **A function with output pins** sets them on its Return node(s). With several early returns, each Return node gets its own values.
9. **Loops over arrays that can change:** `Get All Actors Of Class` returns a fresh array, so a plain `For Each` over it is fine even while the actors are destroyed.
10. **Float and integer:** `Get Viewport Size` returns integers; convert to float before using the rectangle helpers. `FMod` is the float remainder node.
11. **Variable defaults:** set the default on the variable **and** read it back. New variables are **not** Instance Editable.
12. **Properties not settable through ObjectTools:** `bIsPureFunc`, `BlueprintType`, `ActorLabel`. The variable-rename tool does not update references: do not rename anything.
13. **PIE tools:** `StartPIE`, `StopPIE`, `IsPIERunning`, `GetLogEntries`, `CaptureViewport`, `CaptureEditorImage`. Use `CaptureEditorImage` for the play view.
14. **Deleting.** Look through the tool list for the tools that delete a function, a variable and a single graph node (search for "delete" and "remove"). **Never "delete" a function by emptying it with `write_graph_dsl`.** If no tool can delete a function or a variable, leave the item where it is, list it under *Not deleted* in the report and carry on.
15. **Do not clean up anything this document does not list.** Leave unreferenced nodes, `StopSpawner`, `bDirectorSilent`, and the disconnected spawn chains in `SpawnRandomEnemy` where they are.

---

## 6. Implementation steps

### Step 1 — `BP_DDSave` and the game-mode variables

**1a.** Create the Blueprint class **`BP_DDSave`** in `/Game/DomeDefense/Core/`, parent `SaveGame`. Add one variable `HighScore` (integer, default 0). Compile. (Fallback: section 3.6.)

**1b.** Add to `BP_DDGameMode` (not Instance Editable). Read the existing graphs first; keep everything.

| Variable | Type | Default |
|---|---|---|
| `FlowState` | integer | **1** |
| `HighScore` | integer | 0 |
| `bNewHighScore` | boolean | false |
| `TitleEnterTime` | float | 0 |
| `SaveSlotName` | string | `DD_Save` |

Compile; read every default back.

**Check:** `BP_DDSave` exists with `HighScore = 0` and compiles clean; the five variables exist with these defaults (`FlowState` is **1**). **Save All.**

---

### Step 2 — `BP_DDGameMode`: high score functions

New functions, one small write each, in this order (all non-Pure):

**`LoadHighScore`** (no inputs):
```
ex = Does Save Game Exist(SaveSlotName, 0)                         // store
If NOT ex → DDLog("No file high=" + HighScore, "Save", false) ; return
S = Load Game From Slot(SaveSlotName, 0)                           // store
D = Cast To BP_DDSave(S)                                           // cast failed → DDLog("Load failed", "Save", true) ; return
h = D.HighScore                                                    // store
HighScore = Max(HighScore, h)
DDLog("Loaded high=" + HighScore, "Save", false)
```

**`SaveHighScore`** (no inputs):
```
S = Create Save Game Object(BP_DDSave)                             // store
D = Cast To BP_DDSave(S)                                           // cast failed → DDLog("Save failed", "Save", true) ; return
D.HighScore = HighScore
ok = Save Game To Slot(D, SaveSlotName, 0)                         // store
If ok → DDLog("Saved high=" + HighScore, "Save", false)
Else   → DDLog("Save failed", "Save", true)
```

**`CommitScore`** (no inputs):
```
If bDirectorSilent → return
If Wave < 1 → return
If Score <= HighScore → return
HighScore = Score
bNewHighScore = true
SaveHighScore()
```
(Three early `return`s. Do not add a log here; `SaveHighScore` logs.)

**`ResetHighScore`** (no inputs):
```
HighScore = 0
bNewHighScore = false
SaveHighScore()
DDLog("High score reset", "Save", false)
```

**Check:** all four compile clean. **Save All.**

---

### Step 3 — `BP_DDGameMode`: flow functions

New functions, one small write each, in this order (all non-Pure):

**`ApplyPause`** (input `bPause` boolean):
```
If bPause → Set Global Time Dilation(0.0001)
Else      → Set Global Time Dilation(1.0)
```

**`ClearField`** (no inputs):
```
n = ClearEnemies(false)                                            // store (ignored)
A = Get All Actors Of Class(BP_ABM)       → For Each: Destroy Actor
X = Get All Actors Of Class(BP_Explosion) → For Each: Destroy Actor
M = Get All Actors Of Class(BP_TapMarker) → For Each: Dismiss()
```
(Three loops in sequence: each loop's **Completed** pin leads to the next `Get All Actors Of Class`.)

**`EnterTitle`** (input `bLog` boolean):
```
ApplyPause(false)
ClearField()
RestoreAll(false)
ResetGame()
LoadHighScore()
FlowState = 0
bNewHighScore = false
TitleEnterTime = Get Game Time In Seconds
If bLog → DDLog("Title high=" + HighScore, "Flow", false)
```

**`StartNewGame`** (input `bLog` boolean):
```
ApplyPause(false)
ClearField()
RestoreAll(false)
ResetGame()
FlowState = 1
bNewHighScore = false
If bLog → DDLog("Start", "Flow", false)
```
(It does **not** start a wave and does **not** call `StartSpawner`; the controller does that, 3.4.)

**`PauseGame`** (input `bLog`):
```
If FlowState != 1 → return
If bGameOver → return
FlowState = 2
ApplyPause(true)
If bLog → DDLog("Pause", "Flow", false)
```

**`ResumeGame`** (input `bLog`):
```
If FlowState != 2 → return
FlowState = 1
ApplyPause(false)
If bLog → DDLog("Resume", "Flow", false)
```

**`QuitToTitle`** (input `bLog`):
```
If FlowState != 2 → return
CommitScore()
If bLog → DDLog("Quit to title score=" + Score + " high=" + HighScore, "Flow", false)
EnterTitle(bLog)
```
(`Score` and `HighScore` are read by the log **after** `CommitScore`; that is intended. `EnterTitle` resets the score afterwards.)

**`GetPauseButtonRect`** (inputs `ViewW`, `ViewH` floats; outputs `X`, `Y`, `Bw`, `Bh` floats):
```
mx = ViewW * 0.04 ; my = ViewH * 0.04                              // store both
s  = Clamp(ViewH * 0.09, 56.0, 140.0)                              // store
return X = ViewW - mx - s ; Y = my ; Bw = s ; Bh = s
```

**`GetMenuButtonRect`** (inputs `Which` integer, `ViewW`, `ViewH` floats; outputs `X`, `Y`, `Bw`, `Bh`):
```
bw = Max(ViewW * 0.32, 240.0) ; bh = Max(ViewH * 0.11, 70.0)       // store both
x  = ViewW * 0.5 - bw * 0.5                                        // store
If Which == 0 → return X = x ; Y = ViewH * 0.42 ; Bw = bw ; Bh = bh
If Which == 1 → return X = x ; Y = ViewH * 0.60 ; Bw = bw ; Bh = bh
return X = 0 ; Y = 0 ; Bw = 0 ; Bh = 0
```

**`PointInRect`** (inputs `Px`, `Py`, `X`, `Y`, `Bw`, `Bh`, `Pad` floats; output `bInside` boolean):
```
bInside = (Px >= X - Pad) AND (Px <= X + Bw + Pad) AND (Py >= Y - Pad) AND (Py <= Y + Bh + Pad)
```

**Check:** all ten compile clean; none is called from anywhere yet. `StartSpawner` is not called by any of them. **Save All.**

---

### Step 4 — `BP_DDGameMode`: `FinishGame`, the `GameOver` hook, the `Skipped` log fix

**4a. New function `FinishGame`** (input `bLog`; non-Pure):
```
CommitScore()
nh = Select Integer(bNewHighScore, 1, 0)                           // store
If bLog → DDLog("Game over score=" + Score + " high=" + HighScore + " new=" + nh, "Flow", false)
```

**4b. `GameOver` (edit in place).** Read it. It ends with `If bLog → nb = CountAliveBases() ; DDLog("GAME OVER cities=0 bases=" + nb, "Game", false)`. Add **one** call node `FinishGame(bLog)` (the `bLog` input of `GameOver`) and wire **both** the exec output of that `DDLog` **and** the `False` pin of that `If` into the new node's exec input. Change nothing else.

**4c. `SkipWave` (edit in place).** The Phase 6 log prints `Skipped n=2 next=2` instead of `n=1 next=2`: the log reads `Wave` after `StartWave` has already changed it. Read the function and make it so that:
- a local integer `old` is **set** (a Set node, not a pure link) to `Wave` **before** `ClearEnemies` and before `StartWave` runs;
- `StartWave`'s first input is `old + 1`;
- the log's `n=` part reads **`old`** (never `Wave`); the `next=` part keeps reading `Wave` (after `StartWave`, so it is the new wave).
Change nothing else.

**Check:** `FinishGame` compiles; `GameOver` calls `FinishGame` exactly once, reached from both ends of the `If bLog` branch; `SkipWave` sets `old` before the `StartWave` call; `BP_DDGameMode` compiles clean. **Save All.**

---

### Step 5 — `BP_DDPlayerController`: input for the new states

Read the existing graphs first. **Test functions are not touched (rule 12).**

#### 5a. New functions (one small write each, in this order, non-Pure)

**`BeginGame`** (no inputs):
```
GM.StartNewGame(true)
running = GM.bSpawnerRunning                                       // store
If running → GM.StartWave(1, Get Game Time In Seconds)
Else       → GM.StartSpawner
```
(`StartSpawner` is a custom event on the game mode; it is called from here exactly as the old code called it.)

**`HandleTitleTap`** (no inputs):
```
Now = Get Game Time In Seconds                                     // store
t0 = GM.TitleEnterTime                                             // store
If (Now - t0) >= 0.5 → BeginGame()
```

**`HandlePausedTap`** (inputs `ScreenX`, `ScreenY`, `W`, `H` floats):
```
(rx, ry, rw, rh) = GM.GetMenuButtonRect(0, W, H)                   // store all four in locals
inR = GM.PointInRect(ScreenX, ScreenY, rx, ry, rw, rh, 0.0)        // store
If inR → GM.ResumeGame(true) ; return
(qx, qy, qw, qh) = GM.GetMenuButtonRect(1, W, H)                   // store all four
inQ = GM.PointInRect(ScreenX, ScreenY, qx, qy, qw, qh, 0.0)        // store
If inQ → GM.QuitToTitle(true)
```

**`HandlePlayingTap`** (inputs `ScreenX`, `ScreenY`, `W`, `H` floats; output `bConsumed` boolean):
```
If GM.bGameOver → return false
(px, py, pw, ph) = GM.GetPauseButtonRect(W, H)                     // store all four
pad = pw * 0.25                                                    // store
inP = GM.PointInRect(ScreenX, ScreenY, px, py, pw, ph, pad)        // store
If inP → GM.PauseGame(true) ; return true
return false
```

**`HandleFlowTap`** (inputs `ScreenX`, `ScreenY` floats; output `bConsumed` boolean):
```
st = GM.FlowState                                                  // store
(vw, vh) = Get Viewport Size                                       // integers; store
W = float(vw) ; H = float(vh)                                      // store
If st == 0 → HandleTitleTap() ; return true
If st == 2 → HandlePausedTap(ScreenX, ScreenY, W, H) ; return true
r = HandlePlayingTap(ScreenX, ScreenY, W, H)                       // store
return r
```

**`TogglePause`** (no inputs):
```
st = GM.FlowState                                                  // store
If st == 1 → GM.PauseGame(true)
If st == 2 → GM.ResumeGame(true)
```
(The two `If`s are chained: the **False** pin of the first leads to the second.)

**`HandleBack`** (no inputs):
```
st = GM.FlowState                                                  // store
If st == 0 → Quit Game(Specific Player = Get Player Controller(0), Quit Preference = Quit, Ignore Platform Restrictions = false)
If st == 2 → GM.QuitToTitle(true)
If st == 1:
    go = GM.bGameOver                                              // store
    If go → GM.EnterTitle(true)
    Else  → GM.PauseGame(true)
```
(Chain the `If`s through their **False** pins; the `st == 1` branch ends after its nested `If`.)

**Check:** all seven compile clean. **Save All.**

#### 5b. Key and Back events (`create_node` + `connect_pins`, Pressed pin only)
- **`P`** → `TogglePause()`.
- **`Delete`** → `GM.ResetHighScore()` (fallback key `U`, record it).
- **Android Back** (`find_node_types`, "Android Back") → `HandleBack()`. If the node does not exist: skip, record it.
- `H`, `R`, `E`, `K`, `X`, `V`, `N`, the mouse event and the touch event stay **exactly** as they are.

#### 5c. `RequestFire` — insert at the very top (edit in place)
Read it. Its first node leads into the Phase 5 branch `If GM.bGameOver`. Insert, **before that branch**:
```
c = HandleFlowTap(ScreenX, ScreenY)                                // store
If c → return                                                      // a new Return node; nothing after it
```
The **False** pin of this new branch goes to the existing Phase 5 branch. Change nothing else.

#### 5d. `RequestFire` — swap one call inside the game-over branch (edit in place)
Inside the existing Phase 5 branch, the node `RestartGame()` (reached when `IsRestartAllowed` is true): **delete that one node** and put **`GM.EnterTitle(true)`** in its place (same incoming and outgoing exec wires). The `IsRestartAllowed` test, the lockout and the `return` stay.

#### 5e. Auto-pause (only if possible; see 3.8)
Try the events named in 3.8. If one can be created in `BP_DDPlayerController`'s event graph as it stands: add it and wire it to `GM.PauseGame(true)`. If not: change nothing and record "auto-pause not available" under *Deviations*.

**Check:** compiles clean; `RequestFire` starts with `HandleFlowTap` then the Phase 5 branch; the branch calls `GM.EnterTitle(true)` and no longer calls `RestartGame` (`RestartGame` still exists, unreferenced); the keys `H`, `R`, `E`, `K`, `X`, `V`, `N`, `P`, `Delete` exist (and Android Back, if it could be created) and no others; the startup chain is **unchanged**; no test function was edited. **Save All.**

---

### Step 6 — `BP_DDHUD`

Read the current graphs first. **Edit in place with small node edits; the existing draw code stays.** In the text below `GM` means `OwnerPC.GM`; every new function starts with the guard `IsValid(OwnerPC)` AND `IsValid(OwnerPC.GM)` (otherwise it returns).

**6a. New functions** (one small write each, in this order, non-Pure):

- **`DrawInRect`** (inputs `Text` string, `Color` linear color, `X`, `Y`, `Bw`, `Bh`, `Scale` floats):
  ```
  (w, h) = Get Text Size(Text, Font = None, Scale = Scale)
  Draw Text(Text, Color, X = X + Bw * 0.5 - w * 0.5, Y = Y + Bh * 0.5 - h * 0.5, Font = None, Scale = Scale, Scale Position = false)
  ```
- **`DrawTitleScreen`** (inputs `SizeX`, `SizeY`):
  ```
  If GM.FlowState != 0 → return
  DrawCentered("DOME DEFENSE", (1.0, 0.9, 0.2), SizeX, SizeY * 0.18, 7.0)
  DrawCentered("HIGH SCORE " + GM.HighScore, White, SizeX, SizeY * 0.18 + 150, 2.5)
  t = Get Game Time In Seconds                                     // store
  ph = FMod(t, 1.2)                                                // store
  If ph < 0.8 → DrawCentered("TAP OR CLICK TO START", White, SizeX, SizeY * 0.58, 3.0)
  ```
- **`DrawPauseButton`** (inputs `SizeX`, `SizeY`):
  ```
  If GM.FlowState != 1 OR GM.bGameOver → return
  (x, y, w, h) = GM.GetPauseButtonRect(SizeX, SizeY)               // store all four
  Draw Rect((0, 0, 0, 0.45), x, y, w, h)
  bw = w * 0.16 ; bh = h * 0.50 ; by = y + (h - bh) * 0.5          // store
  Draw Rect((1, 1, 1, 0.9), x + w * 0.5 - bw * 1.5, by, bw, bh)
  Draw Rect((1, 1, 1, 0.9), x + w * 0.5 + bw * 0.5, by, bw, bh)
  ```
- **`DrawPauseScreen`** (inputs `SizeX`, `SizeY`):
  ```
  If GM.FlowState != 2 → return
  Draw Rect((0, 0, 0, 0.6), 0, 0, SizeX, SizeY)
  DrawCentered("PAUSED", (1.0, 0.9, 0.2), SizeX, SizeY * 0.22, 5.0)
  (rx, ry, rw, rh) = GM.GetMenuButtonRect(0, SizeX, SizeY)         // store all four
  Draw Rect((0.15, 0.35, 0.15, 0.9), rx, ry, rw, rh)
  DrawInRect("RESUME", White, rx, ry, rw, rh, 2.5)
  (qx, qy, qw, qh) = GM.GetMenuButtonRect(1, SizeX, SizeY)         // store all four
  Draw Rect((0.35, 0.12, 0.12, 0.9), qx, qy, qw, qh)
  DrawInRect("QUIT TO TITLE", White, qx, qy, qw, qh, 2.5)
  ```
- **`DrawHighScore`** (input `SizeX`):
  ```
  If GM.FlowState != 1 OR GM.Wave < 1 → return
  hs = Max(GM.HighScore, GM.Score)                                 // store
  DrawCentered("HI " + hs, (0.75, 0.75, 0.75), SizeX, 85, 1.6)
  ```
- **`DrawGameOverInfo`** (inputs `SizeX`, `SizeY`):
  ```
  If NOT GM.bGameOver → return
  DrawCentered("SCORE " + GM.Score, (1.0, 0.9, 0.2), SizeX, SizeY * 0.40 + 150, 2.5)
  If GM.bNewHighScore → DrawCentered("NEW HIGH SCORE!", (0.3, 1.0, 0.4), SizeX, SizeY * 0.40 + 200, 2.5)
  Else                → DrawCentered("HIGH SCORE " + GM.HighScore, White, SizeX, SizeY * 0.40 + 200, 2.0)
  ```
- **`DrawFlowScreens`** (inputs `SizeX`, `SizeY`): `DrawTitleScreen(SizeX, SizeY)` ; `DrawPauseButton(SizeX, SizeY)` ; `DrawHighScore(SizeX)` ; `DrawGameOverInfo(SizeX, SizeY)` ; `DrawPauseScreen(SizeX, SizeY)` (the pause screen last, so it dims everything drawn before it).

**6b. Edits in place** (small node edits):
1. Line 1 text → `"DOME DEFENSE | PHASE 7 | GAME FLOW"`. Lines 2 to 4 and the GAME OVER block stay.
2. In `Event ReceiveDrawHUD`, insert `DrawFlowScreens(SizeX, SizeY)` **between the `DrawWaveInfo` call and the `If GM.bGameOver` branch** (break that one wire; route it through the new call).
3. In `DrawScoreBlock`, the `DrawCentered` call for `BONUS CITIES` has `Y = 85`: set that pin's default to **`115`**. Nothing else in that function changes.
4. In the `If GM.bGameOver` block, the string literal `TAP OR CLICK TO RESTART` → **`TAP OR CLICK TO CONTINUE`** (change the pin default). Nothing else in the block changes.

**Fallback:** if `Get Text Size` is not available, use `w = 8.5 * Scale * Length(Text)` and `h = 14 * Scale`. Record it.

**Check:** compiles clean; the HUD still draws lines 1 to 4, the score block, banner, tally and the GAME OVER block. **Save All.**

---

### Step 7 — Run 1: regression with the old startup

Everything new is in place, `FlowState` is still 1 and the old startup is untouched, so the game must behave exactly like Phase 6 plus a pause button and an `HI` line.

1. **Save All** (save the level before any level load).
2. `StartPIE`. The self-test runs for about **10 seconds**, then the spawner starts and wave 1 begins. Wait about **28 seconds** in total.
3. `CaptureEditorImage` at about 6 s and at about **14 s and 22 s** (enemies on screen, the pause button, the `HI` line). The times depend on when `SelfTest done` was printed; read them from the log afterwards.
4. `GetLogEntries` (filter `[DD]`), then `StopPIE`.
5. If any test line says FAIL, if any line contains `ERROR` or `Accessed None`, or if **any** `[DD][Save]` or `[DD][Flow]` line appears (a test committed a score or called a flow function): **stop and report** (rule 12). Do not continue to Step 8.

---

### Step 8 — Startup: enter the title, unhook the self-test

`Event BeginPlay` in `BP_DDPlayerController`; small node edits only. Read the real graph first (see section 4, "Startup chain as built").
1. Create a call node **`GM.EnterTitle(true)`**.
2. Wire the exec output that currently feeds `Branch(bSelfTest)` into the new `EnterTitle` node; wire `EnterTitle`'s exec output into the **first `Delay(0.5)`** of the True chain (the one before `UpdateArea`).
3. Delete the `Branch(bSelfTest)` node.
4. Delete the **second `Delay(0.5)`** (the one after `UpdateArea`) **and every node after it** in the old True chain (`RunSelfTest` … `SelfTestEnemyEnd`, including the other `Delay` nodes). **One node per tool call, starting from the end of the chain.**
5. Delete the old False-branch call node `GM.StartSpawner`.

The chain is now: `… Crosshair spawned log → GM.EnterTitle(true) → Delay(0.5) → UpdateArea(Get Viewport Size)`, and nothing after `UpdateArea`.

**Check:** `BP_DDPlayerController` compiles clean; `BeginPlay` contains no call to any test function and no `Branch(bSelfTest)`; `StartSpawner` is called from `BeginGame` and, until Step 9, from inside the now unreferenced `SelfTestEnemyEnd`. **Save All.**

---

### Step 9 — Delete the tests and the dead code

#### 9a. Inventory (read-only, first)
List, by name, every function, custom event and variable of `BP_DDPlayerController` and of `BP_DDGameMode` (counts go in the report). Read every **kept** graph once (listed below) so that you know what they reference.

**KEEP in `BP_DDPlayerController`:** `ScreenToPlane`, `ClampToPlayfield`, `UpdateArea`, `RequestFire`, `LaunchAt`, `SelectBase`, `ConsiderBase`; the Phase 7 functions (`BeginGame`, `HandleTitleTap`, `HandlePausedTap`, `HandlePlayingTap`, `HandleFlowTap`, `TogglePause`, `HandleBack`); every event (BeginPlay, Tick, keys, mouse, touch); the variables `GM`, `Cfg`, `Crosshair`, `CrosshairWorldPos`, `PlayfieldCamera`, `LastViewportW`, `LastViewportH`, `LastFireTime`, `FireCount`, `LastFirePos`, `LastMarker`, `bDesktopCrosshair`, and **any other variable that a kept graph references**.
**KEEP in `BP_DDGameMode`:** everything except the three items listed below. In particular keep `StopSpawner`, `bDirectorSilent`, `ClearEnemies`, `RestoreAll`, `RefillBases` and all counters (`EnemiesKilled`, `ExplosionCount`, `LastExplosion`, …).

#### 9b. DELETE list (in this order: callers before callees)
1. Controller: `RunPhase6Tests`; `RunT41`; `RunT40`; `RunT39` down to `RunT32`; `OriginCaseC`, `OriginCaseB`, `OriginCaseA`, `WaveSpeedCase`, `CapRoomCase`, `BonusCase`.
2. Controller: `RunSelfTest`; `RunPhase2Tests`; `RunPhase3Tests`; `RunPhase4Tests`; `RunPhase4Rest`; `RunPhase5Tests`.
3. Controller: `SelfTestEnemyEnd`, `SelfTestEnemyStart`, `SelfTestExplosionEnd`, `SelfTestFlightEnd`, `SelfTestFlightMid`, `SelfTestFlightStart`.
4. Controller: `FreezeEnemies`, then `ReportTest`, then any **other** function whose name starts with `Run`, `SelfTest` or `Test`, or that is called **only** from functions already deleted (the inventory shows them).
5. Controller variables: `bSelfTest`, `TestPassCount`, `Test6Pass`, `TestEnemyCityHitsBefore`, `TestEnemyA`, and every other variable whose name starts with `Test` or that **no kept graph references**.
6. Controller: `RestartGame`, then the variable `bRestarting`.
7. Game mode: `DestroyLastExplosion`, `CacheLastExplosion` (both used only by deleted tests), then the **custom event `EnemySpawnLoop`**: first confirm that nothing calls it (the only call allowed is the one inside its own chain), then delete its nodes one at a time from the end.

#### 9c. How to delete (all rules apply)
- **One item per tool call.** Order as above.
- Compile `BP_DDPlayerController` (and `BP_DDGameMode` after item 7) **after every five deletions** and after the last one; **Save All** after each compile.
- If a compile error names a **missing function or variable**, **stop and report** which one. Do not re-create it, do not guess, do not edit a kept graph to work around it.
- Never use `write_graph_dsl` to "empty" a function (rule 11).
- If no tool can delete an item (note 14 of section 5), leave it and list it under *Not deleted*.

**Check:** both Blueprints compile clean; `StartSpawner` is called from exactly one place (`BeginGame`); the controller contains no test function, no `RestartGame`, no `bSelfTest`; `EnemySpawnLoop` no longer exists; the Phase 7 functions, the keys and the HUD are untouched. **Save All.**

---

### Step 10 — Run 2: final run

1. **Save All** (save the level before any level load).
2. `StartPIE`. The title screen appears at once. **Wait about 20 seconds**; nobody taps, so nothing else must happen.
3. `CaptureEditorImage` at about 2 s and again about 0.7 s later (the start text blinks: one of the two should show it) and once at about 15 s.
4. `GetLogEntries` (filter `[DD]`), then `StopPIE`.
5. Fill in the report (section 8).

---
## 7. Acceptance tests

### A. Static checks
- [ ] **A1** New assets: **exactly one**, `BP_DDSave` (parent `SaveGame`, variable `HighScore` integer 0). No widget, no GameInstance Blueprint, no new level actor. `DA_DDConfig` is unchanged. (If the fallback of 3.6 was used, say so.)
- [ ] **A2** `BP_DDSave`, `BP_DDGameMode`, `BP_DDPlayerController` and `BP_DDHUD` compile with 0 errors and 0 warnings.
- [ ] **A3** `BP_DDGameMode` has the variables `FlowState` (default **1**), `HighScore`, `bNewHighScore`, `TitleEnterTime`, `SaveSlotName` (`DD_Save`), and the functions `LoadHighScore`, `SaveHighScore`, `CommitScore`, `ResetHighScore`, `ApplyPause`, `ClearField`, `EnterTitle`, `StartNewGame`, `PauseGame`, `ResumeGame`, `QuitToTitle`, `GetPauseButtonRect`, `GetMenuButtonRect`, `PointInRect`, `FinishGame`.
- [ ] **A4** `GameOver` calls `FinishGame(bLog)` exactly once, reached from both ends of its `If bLog` branch. `SkipWave` sets the local `old` before `StartWave` and the log's `n=` reads `old`.
- [ ] **A5** `BP_DDPlayerController` has the functions `BeginGame`, `HandleTitleTap`, `HandlePausedTap`, `HandlePlayingTap`, `HandleFlowTap`, `TogglePause`, `HandleBack`, and the key events `H`, `R`, `E`, `K`, `X`, `V`, `N`, `P`, `Delete` (plus the Android Back event, or a recorded deviation) and **no other keys**; the mouse and touch events are unchanged.
- [ ] **A6** `RequestFire` begins with `HandleFlowTap` and a `return` branch, then the Phase 5 branch; that branch calls `GM.EnterTitle(true)` where it used to call `RestartGame`. **`RestartGame` is called from nowhere and no longer exists.**
- [ ] **A7** `Event BeginPlay` ends with `… Crosshair spawned log → GM.EnterTitle(true) → Delay(0.5) → UpdateArea(Get Viewport Size)` and nothing after `UpdateArea`; there is no `Branch(bSelfTest)`.
- [ ] **A8** `StartSpawner` is called from exactly one place (`BeginGame`). `DirectorLoop` is called from `StartSpawner` and from itself only.
- [ ] **A9** `BP_DDHUD` has `DrawInRect`, `DrawTitleScreen`, `DrawPauseButton`, `DrawPauseScreen`, `DrawHighScore`, `DrawGameOverInfo`, `DrawFlowScreens`; `ReceiveDrawHUD` calls `DrawFlowScreens` once, after `DrawWaveInfo` and before the GAME OVER branch; line 1 reads `DOME DEFENSE | PHASE 7 | GAME FLOW`; the `BONUS CITIES` line has `Y = 115`; the game-over line reads `TAP OR CLICK TO CONTINUE`.
- [ ] **A10** Deleted (or listed under *Not deleted* with the reason): every function, variable and event named in Step 9b. The controller has **no** `RunSelfTest`, `RunPhase*`, `RunT*`, `SelfTest*`, `ReportTest`, `BonusCase`, `FreezeEnemies`, `CapRoomCase`, `WaveSpeedCase`, `OriginCase*`, `RestartGame`, `bSelfTest`, `bRestarting`. The game mode has **no** `EnemySpawnLoop`, `CacheLastExplosion`, `DestroyLastExplosion`.
- [ ] **A11** Still present: `StopSpawner`, `bDirectorSilent`, `BP_PlayfieldDebug` and key `H`, HUD lines 2 to 4.
- [ ] **A12** No change in `PDA_DDConfig`; no UMG; no Project Settings change; no level actor added.

### B. Log checks (agent)
Order of lines is not guaranteed. "Before done" means before the line `SelfTest done`.

**Run 1 (Step 7, about 28 s):**
- [ ] **B1** The Phase 0 to 6 startup lines still appear (GameMode, `Config OK Width=2250.0 Height=1800.0 ...`, Camera, PlayerController, View target, Crosshair spawned, HUD, PlayfieldDebug, `[DD][Area] Viewport=...`, `Bases spawned n=3`, `Cities spawned n=6`).
- [ ] **B2** `T1` to `T31` are PASS exactly as in Phase 6; `[DD][Test] SelfTest done: 31/31 PASS` **exactly once**; `T32` to `T39` and the two lines of patch 6.1 (`T40`, `T41`) are PASS and `[DD][Test] Phase6 tests: 10/10 PASS` appears once; all before done.
- [ ] **B3** `[DD][Enemy] Spawner started interval=2500ms` exactly once, **after** done; then `[DD][Wave] Start n=1 count=12 mult=1 speed=130` exactly once and `[DD][Wave] Running n=1` exactly once, 1.5 to 3.5 s after the `Start` line; then at least four `[DD][Enemy] Spawn slot=...` lines. No `[DD][Score]` line and no `[DD][Wave] Complete` line (nobody fires).
- [ ] **B4** **No** line with `[DD][Save]` and **no** line with `[DD][Flow]` anywhere. If one appears, a test committed a score or called a flow function: **stop and report** (Step 7.5).
- [ ] **B5** No line containing `ERROR`, `Accessed None`, or `Blueprint Runtime Error`.

**Run 2 (Step 10, about 20 s):**
- [ ] **B6** The startup lines of B1 still appear (the Crosshair line only where the desktop crosshair is on), including `[DD][Area] Viewport=...`.
- [ ] **B7** Exactly one of `[DD][Save] Loaded high=<n>` or `[DD][Save] No file high=<n>`, immediately followed by exactly one `[DD][Flow] Title high=<n>`, both within the first seconds of the run.
- [ ] **B8** **No** `[DD][Test]` line anywhere. During the 20 s wait there is **no** `Spawner started`, no `[DD][Wave]`, no `[DD][Enemy]`, no `[DD][Score]`, no `[DD][Damage]`, no `[DD][Input] Fire request`, no `[DD][Game]` line (so no `Restart requested`, no `GAME OVER`).
- [ ] **B9** No line containing `ERROR`, `Accessed None`, or `Blueprint Runtime Error`.

### C. Visual checks (editor image of the play view; NEEDS-HUMAN if you cannot capture it)
**Run 1:**
- [ ] **C1** A **pause button** is in the top-right corner: a dark square with two white vertical bars, about 4% of the window size away from the right and top edges.
- [ ] **C2** After wave 1 has started: a gray **`HI 0`** line under the `WAVE 1   x1` line. Everything else as in Phase 6 (banner, score block, enemies).
- [ ] **C3** HUD line 1 reads `DOME DEFENSE | PHASE 7 | GAME FLOW`.

**Run 2:**
- [ ] **C4** **Title screen:** black background, brown ground edge to edge, six green cities, three bases with `10`; a large yellow **`DOME DEFENSE`** in the upper part; `HIGH SCORE <n>` under it; a white **`TAP OR CLICK TO START`** in the lower middle (it blinks: it must be visible in at least one of the two close captures; if in neither, mark **NEEDS-HUMAN**).
- [ ] **C5** On the title: **no** pause button, **no** score block, **no** wave banner, **no** red enemy dots (also in the capture at about 15 s).
- [ ] **C6** HUD line 1 reads `DOME DEFENSE | PHASE 7 | GAME FLOW`.

### D. Interactive checks (human, PIE; click inside the viewport first; `Shift+F1` frees the mouse, `Esc` stops PIE) — all NEEDS-HUMAN
List every item as NEEDS-HUMAN in the report; do not run or claim them.
- [ ] **D1** **Start:** after Play, the title screen waits. A click starts the game: log `[DD][Flow] Start`, then (first game only) `Spawner started`, `[DD][Wave] Start n=1 ...`. A click within 0.5 s of the title appearing does nothing.
- [ ] **D2** **Pause:** clicking the pause button, or `P`, freezes everything (enemies, flying ABMs, explosions, the wave timers); the dimmed pause screen shows **PAUSED**, **RESUME** and **QUIT TO TITLE**; log `[DD][Flow] Pause`. Clicking elsewhere does nothing and fires nothing. A click on the pause button never launches an ABM.
- [ ] **D3** **Resume:** the Resume button or `P` continues from exactly where it stopped (nothing skipped, the wave does not restart): log `[DD][Flow] Resume`.
- [ ] **D4** **Quit to title:** from the pause screen, the field is cleared, bases and cities are rebuilt, the title appears: logs `[DD][Flow] Quit to title score=<n> high=<n>`, then `[DD][Save] Loaded high=<n>` and `[DD][Flow] Title high=<n>`. If the score beat the record, `[DD][Save] Saved high=<n>` appears before them. The next click starts wave 1 with score 0.
- [ ] **D5** **Game over:** the log `[DD][Game] GAME OVER ...` is followed by `[DD][Flow] Game over score=<n> high=<n> new=<0 or 1>`; the screen shows GAME OVER, `SCORE <n>`, and `NEW HIGH SCORE!` or `HIGH SCORE <n>`; after about 2 s the line reads `TAP OR CLICK TO CONTINUE`; a click goes to the title (`[DD][Flow] Title ...`). **No level reload:** no `Restart requested`, no new `[DD][Area]` viewport line, no self-test.
- [ ] **D6** **Persistence:** stop PIE and start it again: the title shows the saved high score. `Delete` resets it to 0 (log `[DD][Save] High score reset`).
- [ ] **D7** **`HI` during play** shows the larger of the saved high score and the current score, and follows the score upward once it is passed.
- [ ] **D8** **Debug keys in play unchanged:** `H`, `R`, `E`, `K`, `X`, `N` work as in Phase 6; `V` after game over still restarts the same wave and keeps the score. (On the title, `E`, `K` and the other debug keys may do odd things; that is not a bug.) `N` logs `[DD][Wave] Skipped n=<old> next=<new>` with **`n` the old wave** (for example `n=1 next=2`).
- [ ] **D9** **Layout:** resize the window (wide, 5:4, nearly square) on the title and on the pause screen: texts stay centered, the buttons stay where they are drawn and a click inside a drawn button works, a click just outside does not.
- [ ] **D10** Phase 2 to 6 behavior unchanged in play: nearest-base firing, ammo numbers, explosions, chain reactions, damage, waves, tally, bonus cities.

### D-DEVICE. Android (NEEDS-DEVICE; touch is still open from Phase 1.1)
- [ ] **T-1** A tap on the title starts the game; one tap starts exactly once.
- [ ] **T-2** A tap on the pause button (including a little outside the drawn square) pauses; Resume and Quit to title respond to taps; a tap on the pause button launches no ABM; a second finger is ignored.
- [ ] **T-3** Android Back: while playing it pauses; on the pause screen it quits to the title; on the game-over screen it goes to the title; on the title it leaves the app.
- [ ] **T-4** Sending the app to the background and returning pauses the game (only if auto-pause could be built; otherwise not applicable).
- [ ] **T-5** The saved high score survives closing the app completely and starting it again.
- [ ] **T-6** The pause button and the texts are fully visible and readable at landscape size (nothing hidden under a camera cutout or rounded corner); the pause button is where it is drawn.
- [ ] **T-7** The touch items still open from Phase 6: a tap launches from the nearest base; one tap one launch; after game over a tap in the first 2 s does nothing.

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped.

---

## 8. Report format (the agent's final message)

```
## Phase 7 report
**Engine version:** x.y.z
**MCP tools used:** (list)

### Assets created or changed
(table: path | type | change | compiled clean)

### Inventory (Step 9a)
(counts of functions, events, variables before and after, for BP_DDPlayerController and BP_DDGameMode)

### Deleted
(every function, variable and event actually deleted, in order)

### Not deleted
(anything on the Step 9b list that was left, with the reason; "none" if none)

### Acceptance results
A1 PASS/FAIL ... through E1 PASS/FAIL/NEEDS-HUMAN/NEEDS-DEVICE

### Log excerpt
(Run 1: all [DD] lines from the regression run, with timestamps. Run 2: all [DD] lines from the final run, with timestamps)

### Screenshot notes
(what the play view looked like in each capture; when each capture was taken)

### Tool problems (errors, retries, editor restarts)
(every tool error or timeout, what was tried, whether the editor stopped responding or restarted; "none" if none)

### Deviations
(fallbacks used: SaveGame fallback, Delete-key fallback, Get Text Size fallback, Android Back or auto-pause not available, anything else)

### Open questions / risks
```

---

## 9. What this hands to Phase 8, and what stays

| Provided | How a later phase uses it |
|---|---|
| `GM.FlowState` (0 Title, 1 Playing, 2 Paused), `EnterTitle`, `StartNewGame`, `PauseGame`, `ResumeGame`, `QuitToTitle`, controller `BeginGame` | The only ways a game starts, pauses and ends. A later phase that adds a weapon or enemy type does not touch them. |
| `ApplyPause` (time dilation) | Everything that runs on game time pauses with it. New code must use game time (`Get Game Time In Seconds`, `Delay`, Tick deltas), never real time. |
| `ClearField` | Must also clear any new actor class a later phase adds (bombers, smart bombs, UFOs): add its `Get All Actors Of Class` loop there. |
| `HighScore`, `CommitScore`, `bNewHighScore`, `BP_DDSave` | A later phase that wants more saved data adds variables to `BP_DDSave` and to the load and save functions. |
| `GetPauseButtonRect`, `GetMenuButtonRect`, `PointInRect`, the HUD `Draw...` functions | The pattern for any later on-screen button: one rectangle function used by both the drawing and the tap test. |
| `SpawnDue`, `OnEnemyKilled(bLog)`, `GetWaveCount`, `GetMultiplier`, `GetEnemySpeed` | As in Phase 6: where Phase 8 adds MIRVs, bombers, smart bombs and UFOs. |

**Known leftovers that stay** (a later cleanup step may remove them): `StopSpawner` (stops nothing), `bDirectorSilent` and every `If NOT bDirectorSilent` (always false now that the tests are gone), the two disconnected spawn chains in `SpawnRandomEnemy` that read `EnemyBaseSpeed`, the HUD debug lines 2 to 4, the overlay `BP_PlayfieldDebug`.

---

## 10. What this phase changes in earlier specs

| Earlier text | Now |
|---|---|
| Phase 5: a click or tap after the game-over lockout reloads the level (`RestartGame`, `[DD][Game] Restart requested`) | It returns to the title screen (`GM.EnterTitle`). `RestartGame` and `bRestarting` are deleted. The lockout (`GameOverRestartDelay`, `IsRestartAllowed`) is unchanged. |
| Phase 5 HUD: `TAP OR CLICK TO RESTART` | `TAP OR CLICK TO CONTINUE`. |
| Phase 4/6: the self-test (T1 to T41) runs for about 10 s at every Play and then starts the spawner; `StartSpawner` is called from `SelfTestEnemyEnd` and the `bSelfTest = false` branch | The self-test and all its variables are deleted. The game starts on the title screen; `StartSpawner` is called from `BeginGame` only. `bSelfTest` no longer exists. |
| Phase 6: `[DD][Wave] Skipped n=<old> next=<new>` printed the new wave in `n=` | Fixed: `n` is the old wave. |
| Phase 6 HUD: the `BONUS CITIES` line at `Y = 85` | At `Y = 115`; the new `HI` line is at `Y = 85`. |
| Phase 6: score block top centre; title line `PHASE 6 \| WAVES AND SCORE` | Title line `PHASE 7 \| GAME FLOW`. |
| Phase 6 spec 3.5: after the first `StartSpawner` the director runs forever | Unchanged. New games reuse the running director through `StartWave(1, now)`. |

---

## 11. Common pitfalls

1. **Rewriting an existing function or event graph with `write_graph_dsl`.** This crashed the editor in Phase 5. New functions only; existing graphs by small node edits. Never "empty" a function to delete it.
2. **Touching the startup chain before Step 8**, or touching a test function before Step 9. Run 1 must show the old self-test passing with every Phase 7 addition in place.
3. **`FlowState` default.** The variable's default is **1**. If it is 0, the old synthetic `RequestFire` call in the self-test starts a game on the title path and the tests fail.
4. **Using `Set Game Paused`.** It freezes input too (unless every input node is flagged); the pause screen could never be tapped. Pause is `Set Global Time Dilation(0.0001)` through `ApplyPause`, nowhere else.
5. **Forgetting `ApplyPause(false)`** in `EnterTitle` and `StartNewGame`. A dilation that survives into the title makes the whole game crawl.
6. **A second `StartSpawner` call site or a second `DirectorLoop` start.** `BeginGame` is the only caller; `StartSpawner` is guarded by `bSpawnerRunning`. New games after the first use `StartWave(1, now)`.
7. **Calling `StartSpawner` from a function in the game mode.** It is a custom event; the controller's `BeginGame` calls it, as the old code did.
8. **Letting `CommitScore` write during the old self-test.** Its `bDirectorSilent` and `Wave < 1` guards must stay exactly as written, or Run 1 can write a test score into the real save file.
9. **Drawing and hit-testing with different rectangles.** Both use `GetPauseButtonRect` and `GetMenuButtonRect`; the pause button's hit area is padded by `Bw * 0.25`, the menu buttons are not.
10. **Converting touch coordinates.** Touch, mouse and the HUD canvas are in the same viewport pixels. Do not scale them.
11. **Passing integers to the rectangle helpers.** `Get Viewport Size` returns integers; convert to float.
12. **Reading `Wave` in the `Skipped` log after `StartWave`.** The `old` local must be set before the call and the log must read `old`.
13. **Reading `Score` or `HighScore` before `CommitScore` runs** in `QuitToTitle` and `FinishGame`; the log lines show the values after the commit.
14. **Deleting out of order.** Callers before callees; one item per call; compile after every five. A compile error naming a missing function or variable means stop and report.
15. **Deleting something a kept graph still uses** (a variable such as `LastFireTime` or `LastMarker`). Read the kept graphs first (9a); when in doubt, keep it.
16. **Cleaning up what this document does not list** (`StopSpawner`, `bDirectorSilent`, the disconnected spawn chains, HUD lines 2 to 4, the overlay).
17. **Marking D or D-DEVICE items PASS.** You cannot tap, click or press keys. They are NEEDS-HUMAN and NEEDS-DEVICE.
