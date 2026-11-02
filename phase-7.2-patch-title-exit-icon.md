# Phase 7.2 (patch) — Title Exit Icon

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Type** | **Patch** to Phase 7.1. It replaces one UI element. No rule, flow or score change. |
| **Depends on** | Phase 7.1 (agent run accepted; the human play-test log verified the game-over rule, the debug keys, pause, `Delete` and `N`; the game was tested on a handheld device: touch works). The on-screen joysticks were removed **by the human** (Project Settings → Input → Mobile → Default Touch Interface cleared, app rebuilt, verified on the device). That setting is **not** part of this patch. |
| **Patch goal** | The text button **EXIT GAME** (bottom of the title screen) is replaced by an **icon-only exit button in the top-right corner**, at the same rectangle as the pause button, **on the title screen only**. Tapping it quits the app. |

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **No bars, no aspect caps.** The whole window is the playfield.
3. **Existing logic must survive** unless a step says to change it. Read each graph (`read_graph_dsl`) before editing it.
4. **Compile after every Blueprint change.** Every Blueprint ends with no errors and no warnings.
5. **Run each step's "Check"** before moving on. **Save All after each step's Check passes.**
6. **You cannot press keys, click, touch, or resize in PIE.** Interactive checks (group D) are **NEEDS-HUMAN**; device checks (T) are **NEEDS-DEVICE**. Never mark a D or T item PASS.
7. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
8. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after the run. **Save before loading a level.**
9. **This patch adds no asset and no config value. Do not touch Project Settings, the input settings or any `Config/*.ini` file** (the human changed `DefaultInput.ini` already; leave it).
10. **NEVER REWRITE AN EXISTING FUNCTION OR EVENT GRAPH** with `write_graph_dsl` (it deletes all nodes and rebuilds; on big graphs this ran the editor out of memory in Phase 5). The one new function below may be written once (about 15 nodes). **Existing graphs are changed only by small node edits** (`create_node`, `connect_pins`, break a link, delete one node).
11. **READ THE NEW FUNCTION BACK right after writing it** (`read_graph_dsl`). In Phase 7 three writes reported success and read back empty. If it reads back empty or incomplete: write it **once more**; if it is still wrong, **stop and report** under *Tool problems*.
12. **After any tool error, do not retry the same large write.** Stop that item, try one smaller piece at most, and if it still fails **stop and report**. **If the editor stops responding or restarts, stop and report.**
13. **Order is part of the spec:** HUD first (Step 1), then controller (Step 2), then game mode (Step 3). A function is deleted only after nothing calls it.
14. **Do not clean up anything this document does not list.**

---

## 2. Scope

### In scope
- `BP_DDHUD`: new function `DrawExitIcon`; one call swapped in `DrawFlowScreens`; function `DrawExitButton` deleted.
- `BP_DDPlayerController`: one call swapped in `HandleTitleTapAt`.
- `BP_DDGameMode`: function `GetExitButtonRect` deleted.

### Out of scope (do NOT do)
- Any other change. No change to `ApplyPause`, the game-over rule, scoring, waves, damage, targeting, the pause button, the pause screen, the title texts, the lockout, `BeginGame`.
- Any joystick or touch-interface code (`Activate Touch Interface` and the like). The human's project setting is enough.
- Padding around the exit icon. A confirmation dialog. An exit button anywhere except the title screen. Any new key.
- Cleaning up the known leftovers: `StopSpawner`, `bDirectorSilent` and the places that test it, the two disconnected spawn chains in `SpawnRandomEnemy`, HUD lines 2 to 4, the overlay `BP_PlayfieldDebug`.

---

## 3. Design (decided; do not change)

### 3.1 Rectangle
The exit icon uses **the pause button's rectangle**, from the existing `GM.GetPauseButtonRect(ViewW, ViewH)` (outputs `X`, `Y`, `Bw`, `Bh`): `mx = ViewW * 0.04`, `my = ViewH * 0.04`, `s = Clamp(ViewH * 0.09, 56, 140)`, `X = ViewW - mx - s`, `Y = my`, `Bw = Bh = s`.
Drawing and hit-testing both use it. The pause button is drawn only while `FlowState == 1` and not game over; the exit icon only while `FlowState == 0`. They are never visible together, so sharing the rectangle is safe.

### 3.2 Look (canvas only)
- A square `(0, 0, 0, 0.45)` at the rectangle (the same fill as the pause button).
- An **X** of two white `(1, 1, 1, 0.9)` lines: inset `i = Bw * 0.28`; thickness `th = Max(Bw * 0.09, 3.0)`.
  - Line 1: from `(X + i, Y + i)` to `(X + Bw - i, Y + Bh - i)`.
  - Line 2: from `(X + Bw - i, Y + i)` to `(X + i, Y + Bh - i)`.
- **Fallback:** if the canvas `Draw Line` node cannot be found or placed, draw the letter instead: `DrawInRect("X", White, X, Y, Bw, Bh, 3.0)`. Record it.

### 3.3 Tap (title screen)
Unchanged from 7.1 except the rectangle: the lockout (`(Now - t0) < 0.5` → consumed, nothing happens); then a tap **inside the exact rectangle** (`PointInRect` with `Pad = 0.0`, on purpose: a near miss starts the game, it must not quit) → log `[DD][Flow] Exit`, then `Quit Game` (Quit Preference = Quit, Ignore Platform Restrictions = false); any other tap → `BeginGame`.

### 3.4 Log lines
No new line. `[DD][Flow] Exit` is unchanged.

---

## 4. What this patch changes in earlier specs

| Earlier text | Now |
|---|---|
| Phase 7.1, 3.3: text button `EXIT GAME` at 80% of the height, width 24% (`GetExitButtonRect`, `DrawExitButton`) | An icon-only exit button at the pause button's rectangle. `GetExitButtonRect` and `DrawExitButton` are deleted. |
| Phase 7.1, 7 (acceptance C1): `EXIT GAME` text button | Replaced by C1 below. |

---

## 5. Tooling notes

1. **Do not guess node IDs.** Use `find_node_types`. The canvas line node is `Draw Line` (inputs: start X and Y, end X and Y, line color, line thickness).
2. **Non-Pure functions need exec wires.** Call `GetPauseButtonRect`, then store all four outputs in locals right after the call.
3. **Reading values:** a value read by a later node is evaluated when that node runs. Store `i` and `th` in locals before the two `Draw Line` calls.
4. **No nodes after a Branch or a Return** in the same chain. A void function cannot create a Return node; a path ending on an unconnected Then pin is fine.
5. **Swapping a call node:** delete the old call node, create the new one, and reconnect the **same** incoming and outgoing exec wires and the **same** data pins. Check every input pin afterwards (an input left at its default is how the pause defect happened).
6. **Deleting a function:** use the tool that deletes a function (as in Phase 7 Step 9). One item per call. Never "empty" a function with `write_graph_dsl`.
7. **PIE tools:** `StartPIE`, `StopPIE`, `IsPIERunning`, `GetLogEntries`, `CaptureViewport`, `CaptureEditorImage`.

---

## 6. Implementation steps

### Step 1 — `BP_DDHUD`

Read the current graphs first. In the text below `GM` means `OwnerPC.GM`.

**1a. New function `DrawExitIcon`** (inputs `SizeX`, `SizeY` floats; non-Pure), one write, then read it back (rule 11). It starts with the guard `IsValid(OwnerPC)` AND `IsValid(OwnerPC.GM)` (otherwise it returns):
```
If GM.FlowState != 0 → return
(x, y, w, h) = GM.GetPauseButtonRect(SizeX, SizeY)                 // store all four
Draw Rect((0, 0, 0, 0.45), x, y, w, h)
i = w * 0.28 ; th = Max(w * 0.09, 3.0)                             // store both
Draw Line(x + i, y + i, x + w - i, y + h - i, (1, 1, 1, 0.9), th)
Draw Line(x + w - i, y + i, x + i, y + h - i, (1, 1, 1, 0.9), th)
```

**1b. `DrawFlowScreens` (edit in place).** Read it. It calls `DrawTitleScreen`, `DrawExitButton`, `DrawPauseButton`, `DrawHighScore`, `DrawGameOverInfo`, `DrawPauseScreen` in that order. **Delete the one call node `DrawExitButton`** and put **`DrawExitIcon(SizeX, SizeY)`** in its place (same incoming and outgoing exec wires; `SizeX` and `SizeY` wired).

**1c. Delete the function `DrawExitButton`** (confirm first that nothing calls it).

**Check:** `BP_DDHUD` compiles clean; `DrawFlowScreens` calls `DrawTitleScreen`, `DrawExitIcon`, `DrawPauseButton`, `DrawHighScore`, `DrawGameOverInfo`, `DrawPauseScreen` in that order; `DrawExitButton` no longer exists; HUD line 1 is unchanged. **Save All.**

---

### Step 2 — `BP_DDPlayerController`

**`HandleTitleTapAt` (edit in place).** Read it. Its middle part is: `(ex, ey, ew, eh) = GM.GetExitButtonRect(W, H)` → `inE = GM.PointInRect(ScreenX, ScreenY, ex, ey, ew, eh, 0.0)` → branch. **Delete the one call node `GM.GetExitButtonRect`** and put **`GM.GetPauseButtonRect`** in its place:
- same incoming and outgoing exec wires;
- its inputs `ViewW`, `ViewH` wired from the function's `W`, `H` inputs;
- its four outputs `X`, `Y`, `Bw`, `Bh` wired into the same four locals (or directly into the same `PointInRect` pins) that the old call fed.

`PointInRect`'s `Pad` stays **`0.0`**. The lockout, the `Exit` log, `Quit Game` and `BeginGame` stay exactly as they are. Nothing else in the function changes.

**Check:** `BP_DDPlayerController` compiles clean; `HandleTitleTapAt` reads `GetPauseButtonRect(W, H)` and `PointInRect(..., 0.0)`; nothing in the controller calls `GetExitButtonRect`; the keys are exactly `H`, `R`, `E`, `K`, `X`, `V`, `N`, `P`, `Delete`, `B` (+ Android Back); the startup chain is unchanged. **Save All.**

---

### Step 3 — `BP_DDGameMode`

**Delete the function `GetExitButtonRect`.** First confirm that nothing calls it (its two callers, `DrawExitButton` and `HandleTitleTapAt`, are changed above). If no tool can delete it, leave it and list it under *Not deleted*.

**Check:** `BP_DDGameMode` compiles clean. `GetPauseButtonRect` and `GetMenuButtonRect` are unchanged. **Save All.**

---

### Step 4 — Run

1. **Save All.**
2. `StartPIE`. The title screen appears at once. **Wait about 20 seconds**; nobody taps, so nothing else must happen.
3. `CaptureEditorImage` at about 2 s and again about 0.7 s later (the start text blinks), and once at about 15 s.
4. `GetLogEntries` (filter `[DD]`), then `StopPIE`.
5. Fill in the report (section 8). In the report, **state the expected icon rectangle for the run's viewport** (from the `[DD][Area] Viewport=<W>x<H>` line, using 3.1), for example 1332x691: `s = 62.2`, `X = 1216.5`, `Y = 27.6`.

---

## 7. Acceptance tests

### A. Static checks
- [ ] **A1** New assets: **none**. No config value, Project Settings, input settings or `.ini` file changed by the agent.
- [ ] **A2** `BP_DDHUD`, `BP_DDPlayerController`, `BP_DDGameMode` compile with 0 errors and 0 warnings.
- [ ] **A3** `DrawExitIcon` reads back non-empty and matches 1a: guard, `FlowState != 0` return, `GetPauseButtonRect`, the square `(0, 0, 0, 0.45)`, two `Draw Line` calls with the inset and thickness of 3.2 (or the recorded fallback).
- [ ] **A4** `DrawFlowScreens` order is `DrawTitleScreen`, `DrawExitIcon`, `DrawPauseButton`, `DrawHighScore`, `DrawGameOverInfo`, `DrawPauseScreen`.
- [ ] **A5** `HandleTitleTapAt` uses `GetPauseButtonRect(W, H)` with both inputs wired, and `PointInRect(..., Pad = 0.0)`. The lockout `(Now - t0) < 0.5`, the `Exit` log, `Quit Game` (Quit, ignore platform restrictions false) and `BeginGame` are unchanged.
- [ ] **A6** Deleted: `DrawExitButton` and `GetExitButtonRect` (or listed under *Not deleted* with the reason). Nothing references either.
- [ ] **A7** Unchanged: `GetPauseButtonRect`, `GetMenuButtonRect`, `DrawPauseButton`, `DrawPauseScreen`, `ApplyPause`, `GameOver`, the key set (`H`, `R`, `E`, `K`, `X`, `V`, `N`, `P`, `Delete`, `B`, Android Back), the startup chain.
- [ ] **A8** The report states the expected icon rectangle for the run's viewport (Step 4.5).

### B. Log checks (agent; the one 20 s run, nobody taps)
- [ ] **B1** The startup lines as in the 7.1 run: camera, GameMode, config (`Width=2250.0 Height=1800.0`), bases 3, cities 6, PlayerController, view target, crosshair, HUD, `[DD][Area] Viewport=...`.
- [ ] **B2** Exactly one of `[DD][Save] Loaded high=<n>` or `[DD][Save] No file high=<n>`, followed by exactly one `[DD][Flow] Title high=<n>`.
- [ ] **B3** During the 20 s wait: no `Spawner started`, no `[DD][Wave]`, `[DD][Enemy]`, `[DD][Score]`, `[DD][Damage]`, `[DD][Input] Fire request`, `[DD][Game]`, and no `[DD][Flow]` line except the one `Title` line.
- [ ] **B4** No line containing `ERROR`, `Accessed None`, or `Blueprint Runtime Error`.

### C. Visual checks (editor image of the play view; NEEDS-HUMAN if you cannot capture it)
- [ ] **C1** Title screen: a dark square with a white **X** in the **top-right corner**, about 4% of the window size away from the right and top edges, the size of the pause button. **No** `EXIT GAME` text or button anywhere (not at the bottom).
- [ ] **C2** The icon overlaps no HUD text. The rest of the title is as before (yellow `DOME DEFENSE`, `HIGH SCORE <n>`, blinking `TAP OR CLICK TO START`, six cities, three bases with `10`). No pause button, score block, wave banner or enemies.
- [ ] **C3** HUD line 1 is unchanged (`DOME DEFENSE | PHASE 7 | GAME FLOW`).

### D. Interactive checks (human, PIE) — all NEEDS-HUMAN
List every item as NEEDS-HUMAN in the report; do not run or claim them.
- [ ] **D1** A click on the icon logs `[DD][Flow] Exit` and ends the PIE session.
- [ ] **D2** A click anywhere else starts the game; a click within 0.5 s of the title appearing does nothing.
- [ ] **D3** A click a few pixels **outside** the icon starts the game (it does not quit).
- [ ] **D4** Resize the window on the title: the icon stays in the top-right corner and a click inside it works.
- [ ] **D5** In play, the pause button is at the same place and there is no exit icon.

### D-DEVICE. Android (NEEDS-DEVICE)
- [ ] **T-1** A tap on the icon closes the app; a tap anywhere else starts the game; Android Back on the title still leaves the app.
- [ ] **T-2** No on-screen joysticks (the human's project setting; check again after the new build).

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped.

---

## 8. Report format (the agent's final message)

```
## Phase 7.2 report
**Engine version:** x.y.z
**MCP tools used:** (list)

### Assets created or changed
(table: path | type | change | compiled clean)

### Deleted
(DrawExitButton, GetExitButtonRect; "Not deleted" with the reason if a tool could not delete)

### Acceptance results
A1 PASS/FAIL ... through E1 PASS/FAIL/NEEDS-HUMAN/NEEDS-DEVICE

### Expected icon rectangle
(viewport from the log; s, X, Y)

### Log excerpt
(all [DD] lines of the run, with timestamps)

### Screenshot notes
(what the title screen looked like in each capture; when each capture was taken)

### Tool problems (errors, retries, editor restarts, writes that read back empty)
(every tool error or timeout, what was tried, whether the editor stopped responding or restarted; "none" if none)

### Deviations
(Draw Line fallback, anything else)

### Open questions / risks
```

---

## 9. What this hands to later phases

| Provided | How a later phase uses it |
|---|---|
| The corner rectangle `GetPauseButtonRect`, shared by the pause button and the exit icon | The pattern for any later corner button: one rectangle function for drawing and hit-testing. The art phase replaces the drawn X and the pause bars with icons by editing only `DrawExitIcon` and `DrawPauseButton`. |

**Known leftovers that stay:** `StopSpawner` (stops nothing), `bDirectorSilent` and every `If NOT bDirectorSilent` (always false now), the two disconnected spawn chains in `SpawnRandomEnemy` that read `EnemyBaseSpeed`, the HUD debug lines 2 to 4, the overlay `BP_PlayfieldDebug`.

---

## 10. Common pitfalls

1. **Rewriting an existing function or event graph with `write_graph_dsl`.** New function only; edits by small node swaps. Never "empty" a function to delete it.
2. **A write that reports success and reads back empty.** Read `DrawExitIcon` back (rule 11).
3. **A call swapped without reconnecting its inputs.** `GetPauseButtonRect` needs `ViewW` and `ViewH` wired from `W` and `H`; an unwired pair gives a rectangle at the wrong place or size.
4. **Padding on the exit hit area.** `PointInRect`'s `Pad` stays `0.0`.
5. **Deleting `GetExitButtonRect` or `DrawExitButton` before their callers are changed.** Order: Step 1, Step 2, Step 3.
6. **Drawing the icon outside the title.** `DrawExitIcon` returns unless `FlowState == 0`.
7. **Touching Project Settings, input settings or `.ini` files, or adding touch-interface code.** The human already removed the joysticks.
8. **Marking D or T items PASS.** You cannot click or tap. They are NEEDS-HUMAN and NEEDS-DEVICE.
9. **Cleaning up what this document does not list.**
