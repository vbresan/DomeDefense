# Phase 9c.1.2 (patch) — Smart-Bomb Tail: Hold After Dodging and Re-Aim

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Depends on** | Phase 9c.1.1 (built: `UpdateSmartTail` with the `bActive` guard, called from `Event Tick`). The project is at the 9c state plus 9c.1 and 9c.1.1; no bullet-shaped missile work is in it. |
| **Phase goal** | **Visual only. No gameplay change.** Two defects of the smart-bomb tail, found in play. **(1) Flicker:** a smart bomb that hovers on the edge of an explosion's sensing range starts a dodge every one to three frames (the play log shows up to 11 dodge starts in 230 ms), so its tail is switched on and off at frame rate. Fix: after the last dodge frame the tail stays hidden for a short **hold** (`TailHideHold`, default 0.15 s). **(2) Wrong direction:** the tail is aimed once at spawn along the original line; a smart bomb that has bounced sideways flies on from its new position straight at its target, but its tail still points along the old line. Fix: every tick, the smart bomb's tail is re-aimed along **Target − current position**, which is the bomb's real heading in free flight. One new function, two new variables and one replaced call, all in `BP_EnemyMissile`. |

---

## 0. Evidence

- **Flicker** (play log, one smart bomb): 74 dodge starts in one life; 49 of the 73 gaps between starts are 100 ms or less (35 of them 40 ms or less); eight bursts of 4 to 11 starts; one burst is 11 starts in about 230 ms with the bomb nearly stationary. Each start means `bDodging` went false and then true again, so the tail was shown for a frame and hidden again.
- **Direction** (reported in play): after a sideways bounce the tail does not show the bomb's direction. Cause: `AimTrail` runs only in `Init` (9c 3.3), along `Target − StartPos`; nothing re-aims a smart bomb's `TailPivot` afterwards.

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **No gameplay change.** Positions, speeds, targets, kills, impacts, scoring, waves, flyers, MIRV splits, smart-bomb sensing and dodging, and game flow are unchanged. Only the visibility and the rotation of the tail of a **smart bomb** change.
3. **HANDS OFF.** Do not change, and report as unchanged after the work: every component of `BP_EnemyMissile` (transform, mesh, material, flags); the variables `TrailLen`, `TrailWidth`, `StartPos`, `Target`, `bSmart`, `bDodging`, `bActive`, `SenseRadius`, `DodgeSpeed`; the functions `AimTrail`, `UpdateTrail`, `ArmSmart`, `ApplyDodge`, `ComputePush`, `Advance`, `Die`, `Impact`, `Split`, `CheckSplit`, `Init`, `IsInsideExplosion`, and the **old `UpdateSmartTail`** (it stays in the Blueprint, uncalled, unchanged; it is removed in a later clean-up); every material and material instance.
4. **Existing logic must survive.** Read each graph (`read_graph_dsl`) before editing it.
5. **Compile after every Blueprint change.** `BP_EnemyMissile` ends with no errors and no warnings.
6. **Run each step's "Check"** before moving on. **Save All after each step's Check passes.**
7. **You cannot press keys, click, touch, move the mouse, or resize in PIE, and you cannot start a game.** No missile exists on the title screen, so **you cannot see a smart bomb, its tail or a dodge in PIE.** You can verify: compiles, static read-backs, and a **pin-by-pin trace** of the new function with the cases of 3.4. Everything seen in play is group D (**NEEDS-HUMAN**); device checks are **NEEDS-DEVICE**. Never mark a D or T item PASS.
8. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
9. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after the run. **Save before loading a level.** **Do not touch Project Settings, input settings or any `Config/*.ini` file.** **Do not use, edit or delete anything in `/Game/DomeDefense/Spike/`.**
10. **NEVER REWRITE AN EXISTING FUNCTION OR EVENT GRAPH** with `write_graph_dsl`. The new function of Step 2 is a **brand-new** function, written **once**. `Event Tick` is changed only by small node edits (Step 3).
11. **READ EVERY NEW OR CHANGED GRAPH BACK** right after changing it. If it reads back empty, wrong or incomplete: write or correct it once more; if it is still wrong, **stop and report** under *Tool problems*.
12. **VISIBILITY.** No component is added or changed. `TailSmart` keeps `bVisible` false (its default), **`bHiddenInGame` false**, Movable. The new function uses `SetVisibility` only; it never calls `SetHiddenInGame`.
13. **CHECK EVERY PIN YOU BUILD** (the lesson of the Phase 7 pause defect: a pin left at its default). Read every input pin of every node of the new function back and list them in the report (A8). Most important: the **`Dt` input** must be wired from `DeltaSeconds` in `Event Tick` (a `Dt` of 0 would never let the hold run out and the tail would stay hidden forever after the first dodge); `Atan2` arguments; `Make Rotator` pins; both `SetVisibility` calls; both `Branch` conditions.
14. **After any tool error, do not retry the same large operation.** Stop that item, try one smaller piece at most, and if it still fails **stop and report**. **If the editor stops responding or restarts, stop and report.**
15. **Order is part of the spec:** read first, then the variables, then the function, then the hook, then the run.
16. **FROZEN: do not edit** every asset except `BP_EnemyMissile`. In particular frozen: `BP_ABM`, `BP_DDGameMode`, `BP_DDPlayerController`, `BP_DDHUD`, `BP_Flyer`, `BP_Explosion`, `BP_MissileBase`, `PDA_DDConfig`, `DA_DDConfig`, `L_Game`, and every material and material instance.
17. **No new keys, no new log lines, no new config values, no new components, no new Tick, no new asset.** The only additions are the two variables of 3.2, the function `UpdateSmartTail2` and its call.
18. **Node IDs with parentheses cannot be written in the graph DSL** (`Atan2 (Degrees)`, the float comparison nodes, `Select` variants, and others). Create such nodes with `create_node` and connect them with `connect_pins`. Nodes that are not a "Max" or "Clamp" may be written normally.
19. **A condition that tests `IsValid` or a property of another object must be nested branches.** (Not needed here: every condition below reads a member variable of this Blueprint.)
20. **Do not clean up anything this document does not list.**

---

## 2. Scope

### In scope
- `BP_EnemyMissile`: variables `TailHideHold` and `TailHoldLeft`; the new function `UpdateSmartTail2(Dt)`; replacing the call of `UpdateSmartTail` in `Event Tick` by a call of `UpdateSmartTail2(DeltaSeconds)`.

### Out of scope (do NOT do)
- Showing the tail **during** a dodge (the tail stays hidden while the bomb dodges; smoothing a visible swinging tail is a different design).
- Any change to the head, halo, colours, sizes, `TrailLen`, `TrailWidth`, or to the tail of any other missile (ordinary, MIRV warhead, flyer-dropped, ABM).
- Any change to how any missile moves, senses or dodges.
- Deleting the old `UpdateSmartTail`.

---

## 3. Design (decided; do not change)

### 3.1 What the player sees
- **Free flight:** a smart bomb shows its magenta tail, pointing straight back along its **current heading** (the direction from the bomb to its target). For a bomb that has never left its original line this is exactly what 9c showed.
- **Dodging:** the tail is hidden (as in 9c.1).
- **After the last dodge frame:** the tail stays hidden for **`TailHideHold` seconds (default 0.15)** of continuous non-dodging time, then it appears. If the bomb starts dodging again inside that time, the countdown starts over (so a bomb hovering on the edge of an explosion shows no tail, with no flicker).
- **Bounce:** the tail is re-aimed every tick (also while hidden), so when it appears after a sideways bounce it already points back along the bomb's real heading. Nothing visibly snaps.

In free flight the heading is always `Target − current position`: the existing `Advance` recomputes the direction to the target every tick (8c). `Target` and `GetActorLocation` are X/Z positions in the play plane (Y is constant).

### 3.2 Variables (`BP_EnemyMissile`)

| Variable | Type | Default | Notes |
|---|---|---|---|
| `TailHideHold` | float | **0.15** | **Instance Editable**, category `Trail` (next to `TrailLen` and `TrailWidth`). Seconds. 0 = no hold (the tail then hides only on the frames in which `bDodging` is true, as in 9c.1). |
| `TailHoldLeft` | float | 0.0 | Not Instance Editable. Seconds of hold remaining. Written only by `UpdateSmartTail2`. |

### 3.3 The function `UpdateSmartTail2`
New function in `BP_EnemyMissile`: non-Pure, **input `Dt` (float)**, no outputs. Locals: `loc` (vector), `d` (vector), `phi` (float), `raw` (float), `left` (float). Write it **once**:

```
Branch(bSmart)
    False → (end)
    True  → Branch(bActive)
        False → (end)
        True  →
            loc = GetActorLocation                                     // store
            d   = Target - loc                                         // store (vector subtraction)
            phi = Atan2Degrees(d.Z, d.X)                               // create_node; first argument d.Z, second d.X; store
            TailPivot.SetRelativeRotation(Make Rotator(Roll 0, Pitch phi, Yaw 0), bSweep false)
            Branch(bDodging)
                True →
                    TailHoldLeft = TailHideHold
                    TailSmart.SetVisibility(New Visibility = false, Propagate To Children = false)
                False →
                    raw  = TailHoldLeft - Dt                           // store
                    left = Select(raw > 0.0, raw, 0.0)                 // store (the larger of raw and 0)
                    TailHoldLeft = left                                // writes the MEMBER
                    TailSmart.SetVisibility(New Visibility = (left <= 0.0), Propagate To Children = false)
```

- The rotation is built **exactly like the one in `AimTrail`** (same `Atan2` argument order, same `Make Rotator` with Roll 0 and Yaw 0 and the angle on **Pitch**, same `SetRelativeRotation` target `TailPivot` and `bSweep` false); only the vector differs (`Target − current position` instead of `Target − StartPos`). Step 0 records `AimTrail`'s exact wiring; copy it.
- `bSmart` and `bActive` are tested by two nested branches (no AND node). The `bActive` branch is the guard of 9c.1.1: a missile that was just killed or has landed does nothing and never touches its components.
- The function runs **every tick for every smart bomb**, hidden or not, so the rotation is always current.
- `SetVisibility` to the value the component already has is a no-op in the engine; calling it every tick costs nothing.
- `UpdateTrail` keeps updating `TailSmart`'s location and scale every tick (also while hidden), so the tail has its right length when it appears.

### 3.4 Cases for the trace (`TailHideHold` 0.15, `Dt` 0.016)

| # | Situation | Branches | `phi` / `TailHoldLeft` | `SetVisibility` call |
|---|---|---|---|---|
| 1 | ordinary missile, `bSmart` false | first branch False | nothing happens | none |
| 2 | smart bomb, `bActive` false (killed or landed this tick) | second branch False | nothing happens | **none** (no access to `TailSmart`) |
| 3 | smart bomb, free flight from the start, never dodged (`TailHoldLeft` 0.0, `bDodging` false) | True, True, `bDodging` False | `raw` -0.016, `left` 0.0 | New Visibility = (0.0 <= 0.0) = **true** |
| 4 | the first dodging frame (`bDodging` true) | `bDodging` True | `TailHoldLeft` = **0.15** | New Visibility = **false** |
| 5 | the 1st non-dodging frame after case 4 | `bDodging` False | `raw` 0.134, `left` 0.134, `TailHoldLeft` 0.134 | (0.134 <= 0.0) = **false** (still hidden) |
| 6 | the 9th non-dodging frame after case 4 | `bDodging` False | `left` 0.006 | **false** (still hidden) |
| 7 | the 10th non-dodging frame after case 4 | `bDodging` False | `raw` -0.010, `left` **0.0** | **true** (tail appears) |
| 8 | `bDodging` true again 4 frames after case 4 (the edge-hover case) | `bDodging` True | `TailHoldLeft` back to **0.15** | **false** |
| 9 | `TailHideHold` = 0, `bDodging` true | `bDodging` True | `TailHoldLeft` = 0.0 | **false** (a dodging bomb still hides its tail) |
| 10 | the frame after case 9, `bDodging` false | `bDodging` False | `raw` -0.016, `left` 0.0 | **true** |

Rotation samples (case 3 and later, whenever the function runs): bomb at `loc` = (200, 70, 900), `Target` = (500, 70, 100): `d` = (300, 0, -800), `phi` = **-69.44**. After a sideways bounce, `loc` = (-800, 70, 600), `Target` = (-252, 70, 100): `d` = (548, 0, -500), `phi` = **-42.4** (about); the pivot's Pitch is `phi`, as `AimTrail` would give for that start and target.

### 3.5 What the human should expect
- A smart bomb that is dodging often has no tail. That is intended; with the hold, the tail also comes back 0.15 s after the dodge ends.
- The tail is a straight strip behind the head. It shows the **direction**, not the exact path taken.
- Tuning: `TailHideHold` in the Class Defaults of `BP_EnemyMissile` (category `Trail`). A larger value keeps the tail off longer after a dodge.

---

## 4. What this phase changes in earlier specs

| Earlier text | Now |
|---|---|
| 9c 3.3: the tail is oriented once, in `Init`, for every missile | Still true for every missile **except** a smart bomb, whose `TailPivot` is re-aimed every tick by `UpdateSmartTail2` along `Target − current position`. |
| 9c.1 3.2: `UpdateSmartTail` hides the tail while `bDodging` is true | The tail is hidden while dodging **and** for `TailHideHold` seconds afterwards. |
| 9c.1.1: `Branch(bSmart AND bActive)` | The same guard as two nested branches, inside `UpdateSmartTail2`. |
| 9c.1 `Event Tick`: `UpdateSmartTail()` after `UpdateTrail()` | `UpdateSmartTail2(DeltaSeconds)` after `UpdateTrail()`. The old `UpdateSmartTail` stays uncalled. |
| 9c.1 3.4: "the tail can be a few degrees off the bomb's heading" | No longer true: the tail is re-aimed every tick. |

---

## 5. Tooling notes

1. A new function is written once with the graph-writing tool, then read back. Writes sometimes read back empty: if so, write it **once more**; if it is still empty, stop and report (rules 11 and 14).
2. Replacing the call in `Event Tick` without rewriting it: first read which node `UpdateTrail()` leads to and which node `UpdateSmartTail()` leads to (call that `NEXT`, or "nothing"). Create the call node `UpdateSmartTail2`; wire `DeltaSeconds` (the `Event Tick` output) to its `Dt` pin; connect `UpdateTrail.then → UpdateSmartTail2.exec`, then `UpdateSmartTail2.then → NEXT` (if any). Connecting a connected execution output replaces its old link, so do the second connection right away. Then delete the old `UpdateSmartTail` call node (or, if the tool cannot delete a node, leave it unconnected and record it). Read `Event Tick` back.
3. `Atan2 (Degrees)`, the float comparisons and any `Select` node may have parenthesised IDs: use `create_node`.
4. The vector subtraction `Target - loc` is the engine's vector minus node. Break `d` with a `Break Vector` (or read its `X` and `Z` through the struct member pins) for `Atan2`.
5. `TailHoldLeft` and `TailHideHold` are **member variables**; the function has no local of those names. When the graph reads or writes them, they are the members.
6. `SetVisibility` is the **Set Visibility** node of a scene component (inputs: Target, New Visibility, Propagate To Children). Read all three input pins back.

---

## 6. Implementation steps

### Step 0 — Read (no changes)
1. **Confirm the project state.** `BP_EnemyMissile` has the old function `UpdateSmartTail` (from 9c.1.1) and **no** `BulletBody`, `BulletNose`, `SmartBody`, `SmartNose`, `SteerBullet`, `ArmBullet`. **If any bullet-phase item exists, stop and report.** It has **no** variable `TailHideHold` or `TailHoldLeft`, and no function `UpdateSmartTail2`.
2. **Read the graphs:** the old `UpdateSmartTail` (as built, with its guard form), `AimTrail` with **a pin table for every node** (record the exact `Atan2` argument pins, the `Make Rotator` pins, the `SetRelativeRotation` target and `bSweep`), `UpdateTrail`, `Event Tick`. In `Event Tick` record: the order of `Advance`, `UpdateTrail`, `UpdateSmartTail`, and **which node `UpdateSmartTail()` leads to** (`NEXT`).
3. **Variables:** `Target` (type, member), `StartPos`, `bSmart`, `bDodging`, `bActive`, `TrailLen`, `TrailWidth` (type, default, Instance Editable, category).
4. **`TailSmart` and `TailPivot`:** parent, relative transform, `bVisible`, `bHiddenInGame`, `Mobility`.

**Check:** the report section for Step 0 is complete, including the `AimTrail` pin table and `NEXT`.

### Step 1 — Variables
1. Add `TailHideHold` (float, **0.15**, Instance Editable, category `Trail`) and `TailHoldLeft` (float, 0.0, not Instance Editable). Compile; read both defaults and flags back.

**Check:** both variables read back as in 3.2; `BP_EnemyMissile` compiles clean. **Save All.**

### Step 2 — The function
1. Write **`UpdateSmartTail2`** (3.3) once, with the input `Dt`. Create the parenthesised nodes (`Atan2 (Degrees)`, comparisons, `Select`) with `create_node`.
2. Read it back (`read_graph_dsl`) and read **every input pin of every node** back (rule 13).
3. Compile.
4. **Trace** the function with the ten cases and the two rotation samples of 3.4, and write for each case the value on every pin and what `SetVisibility` receives (the actual wiring, not the intended one). Cases 7 and 10 must show **true**, cases 4, 5, 6, 8 and 9 **false**, cases 1 and 2 **no call**.

**Check:** the function compiles clean; it reads back as 3.3; the trace matches 3.4 in all ten cases and both rotation samples; the `Atan2` arguments are (`d.Z`, `d.X`); the `Make Rotator` has Roll 0, **Pitch from `phi`**, Yaw 0, exactly like `AimTrail`; `TailHoldLeft` is written only in the two places of 3.3; `UpdateSmartTail2` is called nowhere yet. **Save All.**

### Step 3 — The hook
1. Replace the call as in tooling note 2: `Event Tick`: `Advance(DeltaSeconds)` → `UpdateTrail()` → `UpdateSmartTail2(DeltaSeconds)` → `NEXT` (as recorded). **`Dt` is wired from `DeltaSeconds`.**
2. Read `Event Tick` back: the order is as above; the old call is gone (or unconnected and recorded); everything else is as before.
3. Compile.

**Check:** `BP_EnemyMissile` compiles with 0 errors and 0 warnings; `UpdateSmartTail2` is called from `Event Tick` only, with `Dt` wired from `DeltaSeconds`; the old `UpdateSmartTail` is not called from anywhere; both tick flags still true; `TailSmart` flags unchanged; every item of rule 3 reads exactly as before. **Save All.**

### Step 4 — Run
1. **Save All.** `StartPIE`; the title screen appears at once; **wait about 20 seconds**; nobody taps, so no game starts and no missile exists.
2. `CaptureEditorImage` at about 5 s and about 15 s; describe each (the title screen, unchanged).
3. `GetLogEntries` (filter `[DD]`), then `StopPIE`.
4. Fill in the report (section 8).

---

## 7. Acceptance tests

### A. Static checks
- [ ] **A1** Changed assets: `BP_EnemyMissile` only. No other asset, no material or instance, nothing under `Spike`, no Project Settings, input settings or `.ini` change. `L_Game` unchanged.
- [ ] **A2** `BP_EnemyMissile` compiles with 0 errors and 0 warnings.
- [ ] **A3** Step 0 findings are in the report: project state, the old `UpdateSmartTail`, the `AimTrail` pin table, `Event Tick` order and `NEXT`, the variables, the flags of `TailSmart` and `TailPivot`.
- [ ] **A4** `TailHideHold` (float, 0.15, Instance Editable, category `Trail`) and `TailHoldLeft` (float, 0.0, not editable) read back.
- [ ] **A5** `UpdateSmartTail2` reads back as 3.3: input `Dt`; two nested branches (`bSmart`, `bActive`); `loc`, `d`, `phi`; `Atan2` (`d.Z`, `d.X`); `SetRelativeRotation` on `TailPivot` with `Make Rotator` (Roll 0, Pitch `phi`, Yaw 0) and `bSweep` false, wired as in `AimTrail`; `Branch(bDodging)`; `TailHoldLeft` written as in 3.3; both `SetVisibility` calls on `TailSmart`.
- [ ] **A6** The trace table of 3.4 is in the report with the **actual** values of the built graph for all ten cases and both rotation samples, and matches.
- [ ] **A7** `Event Tick` reads back: `Advance(DeltaSeconds)` → `UpdateTrail()` → `UpdateSmartTail2(DeltaSeconds)` → `NEXT`; the old call is gone or unconnected (recorded); `UpdateSmartTail2` is called nowhere else.
- [ ] **A8** **Pin audit (rule 13):** a table of every input pin of every node of `UpdateSmartTail2` (node, pin, wired from / value), and of the `UpdateSmartTail2` call in `Event Tick` (`Dt` wired from `DeltaSeconds`).
- [ ] **A9** **Visibility (rule 12):** `TailSmart` before and after: `bVisible` false, `bHiddenInGame` false, Movable, parent `TailPivot`, material `MI_DD_TailSmart`, transform unchanged. `TailPivot` transform and parent unchanged.
- [ ] **A10** **Hands-off audit (rule 3):** a table that shows, for each item of rule 3, that it reads exactly as before this patch (including the old `UpdateSmartTail`).

### B. Log checks (agent; the one 20 s run, nobody taps)
- [ ] **B1** The startup lines as in the 9c.1.1 run (camera, GameMode, config `Width=2250.0 Height=1800.0`, bases 3, cities 6, PlayerController, view target, crosshair, HUD, `[DD][Area] Viewport=...`).
- [ ] **B2** Exactly one of `[DD][Save] Loaded high=<n>` or `[DD][Save] No file high=<n>`, followed by exactly one `[DD][Flow] Title high=<n>`.
- [ ] **B3** During the 20 s wait: no `Spawner started`, no `[DD][Wave]`, `[DD][Enemy]`, `[DD][Flyer]`, `[DD][Score]`, `[DD][Damage]`, `[DD][Input] Fire request`, `[DD][Aim]`, `[DD][Game]`, and no `[DD][Flow]` line except the one `Title` line.
- [ ] **B4** No line containing `ERROR`, `Accessed None`, `Divide by zero` or `Blueprint Runtime Error`.

### C. Visual checks (captures; describe in words)
- [ ] **C1** The title screen is unchanged. No new object appears.

### D. Interactive checks (human, PIE) — all NEEDS-HUMAN
List every item as NEEDS-HUMAN in the report; do not run or claim them.
- [ ] **D1** **Free flight:** press `S` (smart bomb). Before any explosion is near it, the magenta tail is visible and points straight back along the bomb's flight, as in 9c.
- [ ] **D2** **Direction after a bounce:** make the bomb bounce sideways (an explosion beside its path). After the dodge, when the tail appears, it points back along the bomb's **current** direction of flight (the bomb moves down-left: the tail extends up-right), not along the line it spawned on.
- [ ] **D3** **Hold, no flicker:** while the bomb hovers on the edge of an explosion it shows **no tail** (no strobing). About 0.15 s after the dodge really ends, the tail appears.
- [ ] **D4** **Tuning:** `TailHideHold` appears under `Trail` in the Class Defaults of `BP_EnemyMissile`; a larger value (for example 0.4) keeps the tail off longer after a dodge; 0 brings back the frame-rate flicker of 9c.1.1.
- [ ] **D5** **Other missiles unchanged:** ordinary missiles, MIRV warheads, flyer-dropped missiles and ABMs show their tails exactly as before. No magenta tail on any ordinary missile.
- [ ] **D6** **No gameplay change and no errors:** the dodge path, kills, the smart-bomb score and the `[DD][Enemy] Dodge` lines behave as before; kill at least three smart bombs and let one land; the play log has no `Blueprint Runtime Error` and no `Attempted to access`.
- [ ] **D7** A busy late wave runs without a hitch.

### D-DEVICE. Android (NEEDS-DEVICE)
- [ ] **T-1** The tail behaves as in D1 to D3 on the phone.

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped; the editor is on `L_Game`; nothing under `Spike` was touched.

---

## 8. Report format (the agent's final message)

```
## Phase 9c.1.2 report (smart-bomb tail hold and re-aim)
**Engine version:** x.y.z
**MCP tools used:** (list)

### Step 0 findings
(project state; old UpdateSmartTail; AimTrail pin table; Event Tick order and NEXT; variables; TailSmart and TailPivot flags)

### Variables (Step 1)
(TailHideHold and TailHoldLeft read back)

### Function (Step 2)
(UpdateSmartTail2 read back as DSL)

### Trace table
(the table of 3.4 and the two rotation samples, with the actual values of the built graph)

### Hook (Step 3)
(Event Tick read back; what happened to the old call node)

### Pin audit (rule 13)
(table: function | node | pin | wired from / value)

### Visibility audit (rule 12)
(TailSmart before and after: bVisible | bHiddenInGame | Mobility | parent | material)

### Hands-off audit (rule 3)
(table: item | unchanged? | how it was checked)

### Acceptance results
A1 PASS/FAIL ... through E1 PASS/FAIL/NEEDS-HUMAN/NEEDS-DEVICE

### Log excerpt
(all [DD] lines of the run, with timestamps)

### Screenshot notes
(paths of the captures and a description of each)

### Tool problems (errors, retries, editor restarts, writes that read back empty)
### Deviations
### Open questions / risks
```

---

## 9. What this phase hands to later phases

| Provided | How a later phase uses it |
|---|---|
| `UpdateSmartTail2`, `TailHideHold` | The hold time is tuned in the Blueprint's Class Defaults. A visible, swinging tail during a dodge would extend this function (a stored heading and a turn rate). |

**Known leftovers that stay:** the old `UpdateSmartTail` (uncalled), `StopSpawner`, `bDirectorSilent`, the two disconnected spawn chains in `SpawnRandomEnemy`, the disconnected older fire chain in `LaunchAt`, HUD debug lines 2 to 4, `BP_PlayfieldDebug`, the debug keys, the unused `MI_DD_Enemy` and `MI_DD_Abm`, the `Spike` folder (removed in 9g).

---

## 10. Common pitfalls

1. **`Dt` left unwired (0).** The hold never runs out: after the first dodge the tail never comes back. Wire `Dt` from `DeltaSeconds` in `Event Tick` and read the pin back (rule 13).
2. **`TailHoldLeft` written to a local, or read before it is set.** Use the member variable; `raw` and `left` are the locals.
3. **Showing the tail on the dodging branch.** The `bDodging` True branch hides it (`false`) and restarts the hold; only the False branch can show it.
4. **`left <= 0.0` replaced by `TailHoldLeft` read before the Set.** Compute `left` first, store it, set the member from it, and test `left`.
5. **No `bSmart` branch.** Ordinary missiles would get a visible `TailSmart` and a re-aimed pivot. Both nested branches are required.
6. **Dropping the `bActive` branch.** The function would run on a destroyed missile and log the runtime error of 9c.1.1.
7. **Aiming from `StartPos`.** The new vector is `Target − GetActorLocation`; with `StartPos` the tail would stay on the original line.
8. **`Atan2` arguments swapped, or the angle on Roll or Yaw.** First argument `d.Z`, second `d.X`; the angle goes on **Pitch**, wired like `AimTrail`.
9. **Rotating with `SetWorldRotation` or on the wrong component.** The target is `TailPivot`, relative rotation, as in `AimTrail`.
10. **Losing the rest of `Event Tick`** when replacing the call. Reconnect `UpdateSmartTail2.then` to `NEXT` and read the graph back.
11. **Deleting or editing the old `UpdateSmartTail`, or rewriting any existing graph** with the graph-writing tool (rule 10).
12. **Changing `bHiddenInGame`** (for example with `SetHiddenInGame`). Only `bVisible` is toggled.
13. **Touching `AimTrail`, `UpdateTrail`, `ArmSmart`, `ApplyDodge`, a material, a component or `Spike`**, or cleaning up what this document does not list.
14. **Marking D or T items PASS.** You cannot see a smart bomb in PIE. They are NEEDS-HUMAN and NEEDS-DEVICE.
