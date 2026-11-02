# Phase 7.1 (patch) — Game Over Rule, Title Exit Button, Debug Keys, Call Audit

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Type** | **Patch** to Phase 7. It changes one existing rule (game over), adds one title-screen button and two debug-key changes, and audits earlier work. |
| **Depends on** | Phase 7 (agent run accepted; the human play-tested it: title, start, pause, resume, quit to title, game over, click back to the title, high score saved and reloaded after restarting PIE, pause button visible and clickable). One defect was found in play and **already fixed by hand** (see rule 12). |
| **Patch goal** | **(A)** The game no longer ends the moment the last city falls. It ends at the **end of the wave's tally**, and only if no city stands after the banked bonus cities were rebuilt (the arcade rule). If nothing at all is left standing the wave ends at once. **(B)** An **EXIT GAME** button on the title screen. **(C)** Debug key `B` (bank a bonus city) and `X` also destroying bases once no city stands, so (A) can be tested by hand. **(D)** A read-only audit of the Phase 7 calls with a boolean argument, and of the functions whose writes failed silently in Phase 7. |

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **No bars, no aspect caps.** The whole window is the playfield.
3. **Existing logic must survive** unless a step says to change it. Read each graph (`read_graph_dsl`) before editing it.
4. **Compile after every Blueprint change.** Every Blueprint ends with no errors and no warnings.
5. **Run each step's "Check"** before moving on. **Save All after each step's Check passes.**
6. **You cannot press keys, click, touch, or resize in PIE.** You therefore cannot exercise the new rule, the keys or the button. Interactive checks (group D) are **NEEDS-HUMAN**; device checks are **NEEDS-DEVICE**. Never mark a D or T item PASS.
7. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
8. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after the run. **Save before loading a level.**
9. **This patch adds no asset, no config value and changes no config value.**
10. **NEVER REWRITE AN EXISTING FUNCTION OR EVENT GRAPH** with `write_graph_dsl` (it deletes all nodes and rebuilds; on big graphs this ran the editor out of memory in Phase 5). A **brand-new** function may be written once, in **one small write** (every new function below is under 40 nodes). **Existing graphs are changed only by small node edits** (`create_node`, `connect_pins`, break a link, set a pin default, delete one node). This includes the event graphs: the new key event is made with `create_node` and `connect_pins`.
11. **READ EVERY NEW FUNCTION BACK right after writing it** (`read_graph_dsl`). In Phase 7 three writes reported success and read back empty. If a body reads back empty or incomplete: write it **once more**; if it is still wrong, **stop and report** under *Tool problems*. Do not continue on top of an empty function.
12. **Already fixed by the human; do not change:** in `BP_DDGameMode.PauseGame` the call to `ApplyPause` had its `bPause` pin unticked (default false), so pausing set the time dilation to 1.0 and nothing froze. The human ticked it. Leave it. `ApplyPause` itself is correct (Branch on `bPause`: True `Set Global Time Dilation(0.0001)`, False `Set Global Time Dilation(1.0)`) and is **frozen**.
13. **After any tool error, do not retry the same large write.** Stop that item, try one smaller piece at most, and if it still fails **stop and report**. **If the editor stops responding or restarts, stop and report.** Do not continue with a half-edited graph.
14. **Order is part of the spec:** game mode first (Steps 2 to 4a), then controller, then HUD.
15. **Do not clean up anything this document does not list** (see section 2).

---

## 2. Scope

### In scope
- `BP_DDGameMode`: new functions `NothingStands`, `EndWaveNoTargets`, `DebugBankCity`, `DestroyRandomBase`, `DestroyRandomTarget`, `GetExitButtonRect`; small edits in `CheckWaveEnd`, `StepTally`, `DamageSlot`.
- `BP_DDPlayerController`: new function `HandleTitleTapAt`; new key event `B`; swap one call in key `X`; swap one call in `HandleFlowTap`; delete `HandleTitleTap`.
- `BP_DDHUD`: new function `DrawExitButton`; one inserted call in `DrawFlowScreens`.
- The read-only audit (Step 1), with small fixes only where it finds a wrong value.

### Out of scope (do NOT do)
- Any config change, any new asset, any new level actor, any UMG.
- Changing `ApplyPause`, `GameOver`, `CheckGameOver`'s body, `FinishGame`, `EndWave`, `RebuildCities`, scoring, damage, targeting (`SpawnRandomEnemy` keeps picking uniformly among standing slots), the nearest-base rule.
- An exit button anywhere except the title screen. No confirmation dialog.
- Any other key. The key set after this patch is exactly: `H`, `R`, `E`, `K`, `X`, `V`, `N`, `P`, `Delete`, **`B`** (new) and the Android Back key.
- Cleaning up the known leftovers: `StopSpawner`, `bDirectorSilent` and the places that test it, the two disconnected spawn chains in `SpawnRandomEnemy`, HUD lines 2 to 4, the overlay `BP_PlayfieldDebug`.

---

## 3. Design (decided; do not change)

### 3.1 When the game ends
Old rule (Phase 5): the game ended **at once** when the last city was destroyed, inside `DamageSlot`.
**New rule:**
- Losing the last city does **nothing special**. The wave plays on; enemies attack the **bases** (targeting only ever picks standing slots, unchanged); the player keeps shooting while a base stands and ammo lasts; kills and the tally can still bank a bonus city.
- At the end of a wave the existing order is unchanged: `EndWave` computes the tally, adds the points (a city banked by them counts), then `RebuildCities` spends the bank on random destroyed cities.
- **The game ends when the tally banner time is over and no city stands** (after that rebuild). Otherwise the next wave starts as before.
- With a banked bonus city and no city standing, the rebuild always restores at least one city, so the game goes on.

Where it is decided: `StepTally`, at the moment `Now >= StateEndTime`, calls `CheckGameOver` (its body is unchanged: `If bGameOver → return ; n = CountAliveCities ; If n == 0 → GameOver`). If `bGameOver` is now true the director does **not** start the next wave. `GameOver` and `FinishGame` run exactly as before (enemies cleared, `GAME OVER` log, score committed, `Flow` log), only later.

Consequences, on purpose:
- You see the tally banner for its full time, then the GAME OVER screen.
- The high score is saved **after** the tally points were added.
- `GAME OVER cities=0 bases=<n>` can now show bases still standing.
- `DamageSlot` no longer calls `CheckGameOver`.

### 3.2 Nothing left standing (safety valve)
If **no city and no base** stands while the wave is Running, nothing can score any more and `SpawnRandomEnemy` returns none (its `n == 0` guard), so the spawn counter would never reach zero and the wave would hang. Rule: the wave **ends at once**: all enemies are removed without explosions or points (`ClearEnemies(false)`), `WaveToSpawn = 0`, then `EndWave(Now)` (normal tally, bank spent). Then 3.1 decides: game over if no city stands, otherwise the next wave.
This is checked in `CheckWaveEnd` (called by `StepRunning` every step) **before** its existing tests, and the path ends there (the old tests are on the False path), so `EndWave` can never run twice.
Enemies that are in flight at that moment are removed (they would land on rubble anyway).

### 3.3 EXIT GAME on the title screen
- A button **EXIT GAME** near the bottom of the title screen: `GetExitButtonRect`: `bw = Max(ViewW * 0.24, 200)`, `bh = Max(ViewH * 0.09, 60)`, `X = ViewW * 0.5 - bw * 0.5`, `Y = ViewH * 0.80`. **No padding** (a near miss starts the game, it does not quit).
- Drawn as a rectangle `(0.35, 0.12, 0.12, 0.9)` with the label `EXIT GAME` white, scale 2.0, centered (`DrawInRect`). Only while `FlowState == 0`.
- A tap or click on the title: inside the 0.5 s lockout it is consumed and does nothing (unchanged). Afterwards: **inside the button** → log `[DD][Flow] Exit`, then `Quit Game` (Quit Preference = Quit, Ignore Platform Restrictions = false; same node setup as `HandleBack`). **Anywhere else** → `BeginGame` (unchanged).
- Nothing needs saving on the title (the high score was saved at game over or Quit to title).
- In the editor `Quit Game` ends the PIE session; on Android it closes the app. Android Back on the title still quits (unchanged). The pause screen is unchanged.

### 3.4 Debug keys (desktop only)
- **`B`** → `GM.DebugBankCity`: `BonusCities + 1`, log `[DD][Debug] Bonus city banked (debug) bank=<n>`. It does not touch `BonusEarned`.
- **`X`** (was `GM.DestroyRandomCity(true)`) → `GM.DestroyRandomTarget(true)`: if a city stands → `DestroyRandomCity` (unchanged); otherwise → `DestroyRandomBase` (a random standing base through the normal `DamageSlot` path: explosion, counters, logs); if no base stands either: logs `No surviving base` and does nothing.
- `DestroyRandomCity` itself is **not edited**.

### 3.5 Audit (read-only unless a wrong value is found)
See Step 1. Reason: the pause defect was a boolean pin left at its default when the call was built through the tool. Every other call with a boolean argument in the Phase 7 graphs is checked for the same defect.

### 3.6 Log lines
| Category | Line |
|---|---|
| `Wave` | `[DD][Wave] No targets left n=<wave> bank=<BonusCities>` (once, from `EndWaveNoTargets`; guarded by `NOT bDirectorSilent` like the other director logs) |
| `Flow` | `[DD][Flow] Exit` |
| `Debug` | `[DD][Debug] Bonus city banked (debug) bank=<n>` |
| `Damage` | `[DD][Damage] No surviving base` (only when `X` finds no base) |

All other lines are unchanged. Order at the end of a lost game: `[DD][Wave] Complete ...` → (tally banner time) → `[DD][Game] GAME OVER cities=0 bases=<n>` → `[DD][Save] Saved high=<n>` (if a record) → `[DD][Flow] Game over score=<n> high=<n> new=<0 or 1>`.

---

## 4. What this patch changes in earlier specs

| Earlier text | Now |
|---|---|
| Phase 5, 3.7: game over **at once** when the last city is destroyed; `DamageSlot` ends with `CheckGameOver` | Game over is decided at the end of the tally (3.1). `DamageSlot` no longer calls `CheckGameOver`; `StepTally` does. |
| Phase 6, 3.4: "A banked bonus city does not prevent game over" | **Replaced:** a banked city is rebuilt at the tally and prevents game over. |
| Phase 6, 3.9: game over clears every enemy, the director halts at its next step | Unchanged, but it now happens at the end of the tally, when no enemy is left. |
| Phase 6 `CheckWaveEnd` | One new first test: nothing standing → end the wave (3.2). |
| Phase 7, 3.1: title tap starts the game after 0.5 s | Unchanged, except a tap inside the EXIT GAME button quits (3.3). |
| Phase 7: `HandleTitleTap` | Replaced by `HandleTitleTapAt(ScreenX, ScreenY, W, H)`; `HandleTitleTap` is deleted. |
| Phase 5, 3.10: `X` destroys a random standing city | Also a random standing base when no city stands. New key `B`. |

---

## 5. Tooling notes

1. **Do not guess node IDs.** Use `find_node_types`. Key event ID pattern: `Input|KeyboardEvents|B`.
2. **A value read by a later node is evaluated when that node runs.** Store a value in a local before changing what it depends on. A **non-Pure function's output** may be wired directly into the very next node of the same exec chain (as used in `CheckWaveEnd` below).
3. **Non-Pure functions need exec wires.** Call each in sequence, store each result. The rectangle helpers have four outputs: store all four right after the call.
4. **No nodes after a Branch or a Return** in the same chain. Where steps follow an `If`, connect **both branch ends to the same next exec pin** (an exec input accepts several wires). A path that ends needs no node: **a void function cannot create a Return node; a path ending on an unconnected Then pin is fine.**
5. **Random reads:** each read rolls again. Store the index in a variable or local first.
6. **Deleting a function:** use the tool that deletes a function (as in Phase 7 Step 9). One item per call. Never "empty" a function with `write_graph_dsl`.
7. **PIE tools:** `StartPIE`, `StopPIE`, `IsPIERunning`, `GetLogEntries`, `CaptureViewport`, `CaptureEditorImage`.
8. Viewport sizes from `Get Viewport Size` are integers: convert to float before the rectangle helpers (already done in `HandleFlowTap`).

---

## 6. Implementation steps

### Step 1 — Audit (read-only; fix only a proven wrong value)

Use `read_graph_dsl` on each graph below. **Do not rewrite anything.** A wrong value is fixed by a small edit (set the pin default or reconnect the pin) and recorded.

**1a. Calls with a boolean argument.** For each call node below, report the node's function, the boolean pin, and its value (literal true/false, or the wired source):
- `BP_DDGameMode`: `EnterTitle` (`ApplyPause`, `ClearEnemies`, `RestoreAll` calls); `StartNewGame` (same); `PauseGame` (`ApplyPause`); `ResumeGame` (`ApplyPause`); `QuitToTitle` (`EnterTitle`); `ClearField` (`ClearEnemies`); `GameOver` (`FinishGame`); every `DDLog` call in the Phase 7 functions (`bIsError`).
  Expected: `ApplyPause` is **false** in `EnterTitle`, `StartNewGame`, `ResumeGame` and **true** in `PauseGame`; `ClearEnemies` and `RestoreAll` are **false** in `ClearField`, `EnterTitle`, `StartNewGame`; `EnterTitle` in `QuitToTitle` gets `bLog`; `FinishGame` in `GameOver` gets `bLog`; `bIsError` is false except on the `Save failed` and `Load failed` lines.
- `BP_DDPlayerController`: `BeginGame` (`StartNewGame`); `HandlePausedTap` (`ResumeGame`, `QuitToTitle`); `HandlePlayingTap` (`PauseGame`); `TogglePause` (`PauseGame`, `ResumeGame`); `HandleBack` (`QuitToTitle`, `EnterTitle`, `PauseGame`, `Quit Game`); `RequestFire` (`EnterTitle`); `Event BeginPlay` (`EnterTitle`).
  Expected: `bLog` is **true** on all of these; `Quit Game`: Quit Preference = Quit, Ignore Platform Restrictions = false.

**1b. Bodies that were empty in Phase 7 or never exercised.** Read back and confirm each is non-empty and matches the Phase 7 pseudo-code: `GetPauseButtonRect`, `GetMenuButtonRect`, `PointInRect`, `ApplyPause`, `ResetHighScore`, `TogglePause`, `HandleBack`, `SkipWave`. Report the node count of each.

**1c. `SkipWave`.** Confirm that the local `old` is **set** to `Wave` before `ClearEnemies` and before `StartWave`, that `StartWave`'s first input is `old + 1`, and that the log's `n=` reads `old` while `next=` reads `Wave`. (Phase 7 Step 4c; reported done, never run.) If it is not so, fix it as Phase 7 Step 4c says.

**Check:** the report contains the table of 1a (every call, pin, value, OK/fixed), the node counts of 1b, and the result of 1c. Everything compiles clean. **Save All.**

---

### Step 2 — `BP_DDGameMode`: game over at the end of the tally

**2a. New function `NothingStands`** (no inputs; output `bNone` boolean; non-Pure):
```
nc = CountAliveCities() ; nb = CountAliveBases()                   // store both
return bNone = (nc + nb) == 0
```

**2b. New function `EndWaveNoTargets`** (input `Now` float; non-Pure):
```
c = ClearEnemies(false)                                            // store (ignored)
WaveToSpawn = 0
If NOT bDirectorSilent → DDLog("No targets left n=" + Wave + " bank=" + BonusCities, "Wave", false)
EndWave(Now)
```
(Both ends of the `If` lead into the `EndWave` call.)

**Read both back (rule 11).**

**2c. `CheckWaveEnd` (edit in place).** Read it. It is: `If WaveToSpawn > 0 → return ; If Length(ActiveEnemies) > 0 → return ; EndWave(Now)`. Insert **at the very top**, before its first `If`:
```
NothingStands()                                  // non-Pure call; its output goes straight into the next Branch
Branch(bNone):
    True  → EndWaveNoTargets(Now)               // this path ends here, nothing after it
    False → the existing first If (WaveToSpawn > 0), unchanged
```
Break the one wire from the function's entry to the old first `If`; route it through the two new nodes. Change nothing else.

**2d. `StepTally` (edit in place).** Read it. It is: `If Now >= StateEndTime → StartWave(Wave + 1, Now)`. On the **True** pin insert:
```
CheckGameOver(NOT bDirectorSilent)               // the existing function; Phase 5 body, unchanged
Branch(bGameOver):
    True  → nothing (the path ends)
    False → StartWave(Wave + 1, Now)             // the existing node, moved to this pin
```
`bGameOver` is read when the Branch runs (after `CheckGameOver`).

**2e. `DamageSlot` (edit in place).** Read it. Its last nodes are `SpawnExplosion(P, bLog)` → the `Set bPlayerMade = false` on the returned explosion (patch 6.1) → `CheckGameOver(bLog)` → return `true`. **Delete the one node `CheckGameOver(bLog)`** and connect the exec output of the `Set bPlayerMade` node to the Return node (the output `bDamaged` stays `true`). Nothing else changes.
**Fallback:** if the node cannot be deleted cleanly, **stop and report**; do not leave the function half-wired.

**Check:** `BP_DDGameMode` compiles clean. `CheckGameOver` is called from `StepTally` only; `GameOver` is called from `CheckGameOver` only; `DamageSlot` no longer calls `CheckGameOver`; `CheckWaveEnd` starts with `NothingStands` and a Branch whose True path ends in `EndWaveNoTargets`; `EndWave` is called from `CheckWaveEnd` and `EndWaveNoTargets` only. **Save All.**

---

### Step 3 — `BP_DDGameMode`: debug helpers

New functions, one small write each (non-Pure), each **read back** (rule 11):

**`DebugBankCity`** (no inputs):
```
BonusCities = BonusCities + 1
DDLog("Bonus city banked (debug) bank=" + BonusCities, "Debug", false)
```

**`DestroyRandomBase`** (input `bLog` boolean; output `bDone` boolean):
```
Alive = local array of integers (empty)
For Slot = 0 to 8:
    a = IsSlotAlive(Slot) ; bi = SlotToBaseIndex(Slot)             // store both
    If a AND bi >= 0 → Alive.Add(Slot)
n = Length(Alive)                                                  // store
If n == 0:
    If bLog → DDLog("No surviving base", "Damage", false)
    return false
k = Random Integer In Range(0, n - 1)                              // store in a variable or local FIRST
Slot2 = Alive[k]                                                   // store
r = DamageSlot(Slot2, bLog)                                        // store
return r
```
(`SlotToBaseIndex` returns a value below 0 for city slots; this is the same test `DamageSlot` uses.) **Fallback:** if a local array cannot be created, use a game-mode variable `AliveScratch` (array of integer; if Phase 5 created one, reuse it), cleared at the start. Record it.

**`DestroyRandomTarget`** (input `bLog` boolean; output `bDone` boolean):
```
nc = CountAliveCities()                                            // store
If nc > 0 → r = DestroyRandomCity(bLog) ; return r
r2 = DestroyRandomBase(bLog) ; return r2
```
(Each Return node gets its own value.)

**Check:** all three compile clean; none is called from anywhere yet; `DestroyRandomCity` is unchanged. **Save All.**

---

### Step 4 — `BP_DDGameMode`: the exit button rectangle

**`GetExitButtonRect`** (inputs `ViewW`, `ViewH` floats; outputs `X`, `Y`, `Bw`, `Bh` floats; non-Pure), one small write, read back:
```
bw = Max(ViewW * 0.24, 200.0) ; bh = Max(ViewH * 0.09, 60.0)       // store both
return X = ViewW * 0.5 - bw * 0.5 ; Y = ViewH * 0.80 ; Bw = bw ; Bh = bh
```

**Check:** compiles clean; not called yet. **Save All.**

---

### Step 5 — `BP_DDPlayerController`

Read the existing graphs first. Phase 7's functions are not touched except as listed.

**5a. New function `HandleTitleTapAt`** (inputs `ScreenX`, `ScreenY`, `W`, `H` floats; non-Pure), one small write, read back:
```
Now = Get Game Time In Seconds ; t0 = GM.TitleEnterTime            // store both
If (Now - t0) < 0.5 → return                                       // lockout: consumed, nothing happens
(ex, ey, ew, eh) = GM.GetExitButtonRect(W, H)                      // store all four
inE = GM.PointInRect(ScreenX, ScreenY, ex, ey, ew, eh, 0.0)        // store
If inE:
    DDLog("Exit", "Flow", false)
    Quit Game(Specific Player = Get Player Controller(0), Quit Preference = Quit, Ignore Platform Restrictions = false)
Else:
    BeginGame()
```
(Every path ends; no node after a Branch.)

**5b. `HandleFlowTap` (edit in place).** Read it. In the `st == 0` branch it calls `HandleTitleTap()` and then `return true`. **Delete that one call node** and put **`HandleTitleTapAt(ScreenX, ScreenY, W, H)`** in its place (same incoming and outgoing exec wires). `ScreenX`, `ScreenY` are the function's inputs; `W`, `H` are the float locals it already computed from `Get Viewport Size`. The `st == 2` branch and the playing branch stay.

**5c. Delete the function `HandleTitleTap`** (now unreferenced; confirm nothing calls it first). If no tool can delete it, leave it and list it under *Not deleted*.

**5d. New key event `B`** (`create_node` + `connect_pins`, **Pressed** pin only): `B` → `GM.DebugBankCity()`.

**5e. Key `X` (edit in place).** Read it. It calls `GM.DestroyRandomCity(true)`. **Delete that one call node** and put **`GM.DestroyRandomTarget(true)`** in its place (same wires). Nothing else on the `X` event changes.

**Check:** `BP_DDPlayerController` compiles clean. Keys: exactly `H`, `R`, `E`, `K`, `X`, `V`, `N`, `P`, `Delete`, `B` (+ Android Back); mouse and touch events unchanged; `HandleFlowTap` calls `HandleTitleTapAt` in the `st == 0` branch; `HandleTitleTap` no longer exists (or is listed under *Not deleted*); the startup chain is unchanged. **Save All.**

---

### Step 6 — `BP_DDHUD`

Read the current graphs first. In the text below `GM` means `OwnerPC.GM`; the new function starts with the guard `IsValid(OwnerPC)` AND `IsValid(OwnerPC.GM)` (otherwise it returns).

**6a. New function `DrawExitButton`** (inputs `SizeX`, `SizeY` floats; non-Pure), one small write, read back:
```
If GM.FlowState != 0 → return
(x, y, w, h) = GM.GetExitButtonRect(SizeX, SizeY)                  // store all four
Draw Rect((0.35, 0.12, 0.12, 0.9), x, y, w, h)
DrawInRect("EXIT GAME", White, x, y, w, h, 2.0)
```

**6b. `DrawFlowScreens` (edit in place).** Read it: it calls `DrawTitleScreen`, `DrawPauseButton`, `DrawHighScore`, `DrawGameOverInfo`, `DrawPauseScreen` in that order. **Insert `DrawExitButton(SizeX, SizeY)` directly after `DrawTitleScreen` and before `DrawPauseButton`** (break that one wire; route it through the new call).

**Check:** compiles clean; `DrawFlowScreens` calls the six functions in the order `DrawTitleScreen`, `DrawExitButton`, `DrawPauseButton`, `DrawHighScore`, `DrawGameOverInfo`, `DrawPauseScreen`; the HUD line 1 text is unchanged. **Save All.**

---

### Step 7 — Run

1. **Save All.**
2. `StartPIE`. The title screen appears at once. **Wait about 20 seconds**; nobody taps, so nothing else must happen.
3. `CaptureEditorImage` at about 2 s and again about 0.7 s later (the start text blinks), and once at about 15 s.
4. `GetLogEntries` (filter `[DD]`), then `StopPIE`.
5. Fill in the report (section 8).

---

## 7. Acceptance tests

### A. Static checks
- [ ] **A1** New assets: **none**. `DA_DDConfig` unchanged. No UMG, no level actor, no Project Settings change.
- [ ] **A2** `BP_DDGameMode`, `BP_DDPlayerController`, `BP_DDHUD` compile with 0 errors and 0 warnings.
- [ ] **A3** `BP_DDGameMode` has the new functions `NothingStands`, `EndWaveNoTargets`, `DebugBankCity`, `DestroyRandomBase`, `DestroyRandomTarget`, `GetExitButtonRect`, each read back non-empty. `BP_DDPlayerController` has `HandleTitleTapAt`. `BP_DDHUD` has `DrawExitButton`.
- [ ] **A4** `DamageSlot` no longer calls `CheckGameOver` and still returns `true`. `CheckGameOver` is called from `StepTally` only (`NOT bDirectorSilent` as its `bLog`). `StepTally`: on `Now >= StateEndTime` → `CheckGameOver` → `Branch(bGameOver)`: True ends, False → `StartWave(Wave + 1, Now)`. `GameOver` is called from `CheckGameOver` only. The bodies of `CheckGameOver` and `GameOver` are unchanged.
- [ ] **A5** `CheckWaveEnd` begins with `NothingStands` → `Branch(bNone)`; True → `EndWaveNoTargets(Now)` and the path ends; False → the old first `If`. `EndWave` is called from `CheckWaveEnd` and `EndWaveNoTargets` only (and nowhere twice in one path).
- [ ] **A6** Keys: exactly `H`, `R`, `E`, `K`, `X`, `V`, `N`, `P`, `Delete`, `B`, plus Android Back. `B` calls `GM.DebugBankCity`; `X` calls `GM.DestroyRandomTarget(true)`; `DestroyRandomCity` is unchanged.
- [ ] **A7** `HandleFlowTap` calls `HandleTitleTapAt(ScreenX, ScreenY, W, H)` in the `st == 0` branch. `HandleTitleTapAt`: lockout `< 0.5` returns; button hit → `DDLog("Exit", "Flow", false)` then `Quit Game`; otherwise `BeginGame`. `HandleTitleTap` is deleted (or listed under *Not deleted*).
- [ ] **A8** `DrawFlowScreens` order as in Step 6. `DrawExitButton` draws only while `FlowState == 0`.
- [ ] **A9** Audit: the table of Step 1a is in the report; every boolean value matches its expectation or was fixed and listed; `PauseGame → ApplyPause(true)`, `ResumeGame → ApplyPause(false)` confirmed; Step 1b bodies non-empty (node counts reported); Step 1c result reported.
- [ ] **A10** Unchanged: `ApplyPause`, `GameOver`, `FinishGame`, `EndWave`, `RebuildCities`, `SpawnRandomEnemy`, `StopSpawner`, `bDirectorSilent`, HUD lines 2 to 4, `BP_PlayfieldDebug` and key `H`.

### B. Log checks (agent; the one 20 s run, nobody taps)
- [ ] **B1** The startup lines as in the Phase 7 final run: camera, GameMode, config (`Width=2250.0 Height=1800.0`), bases 3, cities 6, PlayerController, view target, crosshair (where the desktop crosshair is on), HUD, `[DD][Area] Viewport=...`.
- [ ] **B2** Exactly one of `[DD][Save] Loaded high=<n>` or `[DD][Save] No file high=<n>`, followed by exactly one `[DD][Flow] Title high=<n>`.
- [ ] **B3** During the 20 s wait: no `Spawner started`, no `[DD][Wave]`, no `[DD][Enemy]`, no `[DD][Score]`, no `[DD][Damage]`, no `[DD][Input] Fire request`, no `[DD][Game]`, no `[DD][Debug]`, and no `[DD][Flow]` line except the one `Title` line.
- [ ] **B4** No line containing `ERROR`, `Accessed None`, or `Blueprint Runtime Error`.

### C. Visual checks (editor image of the play view; NEEDS-HUMAN if you cannot capture it)
- [ ] **C1** Title screen as in Phase 7 (yellow `DOME DEFENSE`, `HIGH SCORE <n>`, blinking `TAP OR CLICK TO START`, six cities, three bases with `10`) **plus** a dark-red rectangle with the white text `EXIT GAME` at the bottom centre: top edge at about 80% of the height, width about 24% of the window.
- [ ] **C2** The button and `TAP OR CLICK TO START` do not overlap; no pause button, no score block, no wave banner, no enemies on the title.
- [ ] **C3** HUD line 1 is unchanged (`DOME DEFENSE | PHASE 7 | GAME FLOW`).

### D. Interactive checks (human, PIE) — all NEEDS-HUMAN
List every item as NEEDS-HUMAN in the report; do not run or claim them.
- [ ] **D1** **Title exit:** a click on EXIT GAME logs `[DD][Flow] Exit` and ends the PIE session; a click anywhere else starts the game; a click within 0.5 s of the title appearing does nothing.
- [ ] **D2** **Banked city keeps the game alive:** start a game; `B` once; `X` six times (all cities): no GAME OVER; the wave plays on, enemies attack the bases; at the wave's end the tally shows `BONUS CITY REBUILT x1`, one city stands and the next wave starts.
- [ ] **D3** **Nothing left, with a bank:** `B`, then `X` nine times (six cities, three bases): `[DD][Wave] No targets left n=<w> bank=1`, enemies vanish, the tally runs, a city is rebuilt, the next wave starts with bases restored.
- [ ] **D4** **Nothing left, empty bank:** `X` nine times: `No targets left ... bank=0`, `[DD][Wave] Complete ...`, the tally banner stays for its full time, then `[DD][Game] GAME OVER cities=0 bases=0`, `[DD][Save] Saved high=<n>` (if a record) and `[DD][Flow] Game over ...`; the click after the lockout returns to the title.
- [ ] **D5** **Cities gone, bases standing, empty bank:** `X` six times; the wave plays on and **no** GAME OVER appears during it; GAME OVER appears only after the wave's tally banner.
- [ ] **D6** **`X` on bases:** once no city stands, each `X` destroys one base (explosion, `[DD][Damage] Base destroyed slot=... left=<n>`); with none left it logs `No surviving base`.
- [ ] **D7** Phase 7 items never run: `Delete` resets the saved high score (`[DD][Save] High score reset`); `N` logs `[DD][Wave] Skipped n=1 next=2`; resizing the window on the title (button stays where drawn, a click inside works, just outside starts the game), on the pause screen, and on the game-over screen.
- [ ] **D8** Pause still freezes everything and Resume continues from where it stopped (the hand fix of rule 12).

### D-DEVICE. Android (NEEDS-DEVICE)
- [ ] **T-1** A tap on EXIT GAME on the title closes the app; a tap anywhere else starts the game; Android Back on the title still leaves the app.
- [ ] **T-2** The touch items still open from Phase 1.1, Phase 6 and Phase 7 (T-1 to T-7 of the Phase 7 spec).

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped.

---

## 8. Report format (the agent's final message)

```
## Phase 7.1 report
**Engine version:** x.y.z
**MCP tools used:** (list)

### Assets created or changed
(table: path | type | change | compiled clean)

### Audit (Step 1)
1a table: Blueprint | function | call | boolean pin | value found | expected | OK / fixed
1b: function | node count | matches the Phase 7 pseudo-code (yes/no)
1c: SkipWave result

### Deleted
(HandleTitleTap; anything else; "Not deleted" with the reason if a tool could not delete)

### Acceptance results
A1 PASS/FAIL ... through E1 PASS/FAIL/NEEDS-HUMAN/NEEDS-DEVICE

### Log excerpt
(all [DD] lines of the run, with timestamps)

### Screenshot notes
(what the title screen looked like in each capture; when each capture was taken)

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
| Game over decided in `StepTally` after the rebuild | Phase 8: new enemy types need no change. Anything that adds a city-damaging weapon goes through `DamageSlot`, which no longer ends the game. |
| `NothingStands`, `EndWaveNoTargets` | The wave can always end: a later spawner must also return "none" when nothing stands (as `SpawnRandomEnemy` does), or the valve in `CheckWaveEnd` must cover it. |
| `GetExitButtonRect`, `DrawExitButton`, `HandleTitleTapAt` | The pattern for any later title-screen button: one rectangle function for drawing and hit-testing, the lockout first. |
| `B` and `X` on bases | Debug aids for testing the game-over rule by hand. |

**Known leftovers that stay:** `StopSpawner` (stops nothing), `bDirectorSilent` and every `If NOT bDirectorSilent` (always false now), the two disconnected spawn chains in `SpawnRandomEnemy` that read `EnemyBaseSpeed`, the HUD debug lines 2 to 4, the overlay `BP_PlayfieldDebug`.

---

## 10. Common pitfalls

1. **Rewriting an existing function or event graph with `write_graph_dsl`.** New functions only; existing graphs by small node edits. Never "empty" a function to delete it.
2. **A write that reports success and reads back empty.** Read every new function back (rule 11).
3. **Leaving a boolean pin at its default** when building a call through the tool (the cause of the pause defect). Check each call you build: `CheckGameOver(NOT bDirectorSilent)`, `ClearEnemies(false)`, `DestroyRandomTarget(true)`, `DebugBankCity` (none), `Quit Game`.
4. **Calling `EndWave` twice.** The valve path in `CheckWaveEnd` **ends** after `EndWaveNoTargets`; only the False path continues to the old tests.
5. **Starting the next wave after game over.** In `StepTally`, `StartWave` is on the **False** pin of `Branch(bGameOver)`, after `CheckGameOver`.
6. **Leaving `CheckGameOver` in `DamageSlot`.** The game would still end at once. Delete exactly that node and keep the return `true`.
7. **Adding padding to the exit button.** Its hit area is exactly the drawn rectangle; near misses start the game.
8. **Skipping the lockout in `HandleTitleTapAt`.** The first test is `(Now - t0) < 0.5 → return`, before the button test.
9. **Reading a random number twice** in `DestroyRandomBase`. Store the index first.
10. **Editing `DestroyRandomCity`, `ApplyPause`, `GameOver` or `EndWave`.** They are unchanged by this patch.
11. **Marking D or T items PASS.** You cannot press keys, click or tap. They are NEEDS-HUMAN and NEEDS-DEVICE.
12. **Cleaning up what this document does not list.**
