# Phase 9c.1.1 (patch) — Active Guard for the Smart-Bomb Tail Function

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Depends on** | Phase 9c.1 (built: `UpdateSmartTail` and its call in `Event Tick`; patch "Smart-Bomb Tail Off While Dodging") |
| **Phase goal** | **Fix one defect. No visible or gameplay change.** The play log shows four Blueprint runtime errors, one for every smart bomb that was killed, from `UpdateSmartTail`: the function runs on a smart bomb that has just been destroyed in the same tick and reads its `TailSmart` component, which is no longer valid. Add the guard that `UpdateTrail` already has: do nothing when the missile is not active. One node and its pins in one function of `BP_EnemyMissile`. |

---

## 0. The defect (evidence)

From the play log (4 occurrences, each in the same frame as a `Killed` line of the same actor, and one per smart bomb: 4 armed, 4 killed):

```
[BP_EnemyMissile_C_29] [DD][Enemy] Killed slot=BASE-LEFT X=-898 Z=485
LogScript: Warning: Attempted to access TailSmart via property TailSmart, but TailSmart is not valid (pending kill or garbage)
    Function /Game/DomeDefense/Actors/BP_EnemyMissile.BP_EnemyMissile_C:UpdateSmartTail
PIE: Error: Blueprint Runtime Error: "Attempted to access TailSmart ..." Node: SetVisibility Graph: UpdateSmartTail
```

Cause: in one `Event Tick`, `Advance(DeltaSeconds)` can kill or land the missile (`Die` or `Impact` destroys the actor); `UpdateTrail()` returns at once because its first node is `If NOT bActive → return`; `UpdateSmartTail()` then runs on the destroyed actor and calls `SetVisibility` on `TailSmart`. 9c.1 left that guard out. Nothing is wrong on screen; the error is logged.

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **No gameplay change and no visible change.** A smart bomb's tail behaves exactly as in 9c.1 for every active missile.
3. **HANDS OFF.** Do not change, and report as unchanged after the work: every component of `BP_EnemyMissile`; all its variables; the functions `AimTrail`, `UpdateTrail`, `ArmSmart`, `ApplyDodge`, `ComputePush`, `Advance`, `Die`, `Impact`, `Split`, `CheckSplit`, `Init`, `IsInsideExplosion`; **the call order in `Event Tick`** (`Advance` → `UpdateTrail` → `UpdateSmartTail` → whatever followed); every material and material instance. The only graph that changes is `UpdateSmartTail`.
4. **Existing logic must survive.** Read `UpdateSmartTail` (`read_graph_dsl`) before editing it.
5. **Compile after every Blueprint change.** `BP_EnemyMissile` ends with no errors and no warnings.
6. **Run each step's "Check"** before moving on. **Save All after each step's Check passes.**
7. **You cannot press keys, click, touch, move the mouse, or resize in PIE, and you cannot start a game.** No missile exists on the title screen, so **you cannot reproduce the error or see a smart bomb in PIE.** You can verify: compiles, static read-backs, and a **pin-by-pin trace** of the corrected function with the cases of 3.2. Everything seen in play is group D (**NEEDS-HUMAN**). Never mark a D or T item PASS.
8. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
9. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after the run. **Save before loading a level.** **Do not touch Project Settings, input settings or any `Config/*.ini` file.** **Do not use, edit or delete anything in `/Game/DomeDefense/Spike/`.**
10. **NEVER REWRITE AN EXISTING FUNCTION OR EVENT GRAPH** with `write_graph_dsl`. `UpdateSmartTail` is changed only by small node and pin edits (`create_node`, `connect_pins`). The only exception is the named Fallback of Step 1.
11. **READ THE CHANGED GRAPH BACK** right after changing it. If it reads back wrong or incomplete: correct it once more; if it is still wrong, **stop and report** under *Tool problems*.
12. **VISIBILITY.** No component is added or changed. `TailSmart` keeps `bVisible` false (its default), **`bHiddenInGame` false**, Movable. The function never calls `SetHiddenInGame`.
13. **CHECK EVERY PIN YOU TOUCH.** After the edit, read back all input pins of the new node, of the existing `Branch` and of the `SetVisibility` call, and list them in the report (A5). A pin that shows a connection is changed by changing **what it is connected to**; `set_pin_value` on a wired pin changes nothing.
14. **After any tool error, do not retry the same large operation.** Stop that item, try one smaller piece at most, and if it still fails **stop and report**. **If the editor stops responding or restarts, stop and report.**
15. **Order is part of the spec:** read first, then the edit, then the run.
16. **FROZEN: do not edit** every asset except `BP_EnemyMissile`. In particular frozen: `BP_ABM`, `BP_DDGameMode`, `BP_DDPlayerController`, `BP_DDHUD`, `BP_Flyer`, `BP_Explosion`, `BP_MissileBase`, `PDA_DDConfig`, `DA_DDConfig`, `L_Game`, and every material and material instance.
17. **No new keys, no new log lines, no new config values, no new variables, no new components, no new functions (except the Fallback), no new Tick, no new asset.**
18. **Node IDs with parentheses cannot be written in the graph DSL.** Create such nodes (the boolean AND node, for example) with `create_node` and connect them with `connect_pins`.
19. **Do not clean up anything this document does not list.**

---

## 2. Scope

### In scope
- `BP_EnemyMissile.UpdateSmartTail`: make the existing branch condition `bSmart AND bActive`.

### Out of scope (do NOT do)
- Any change to what the tail looks like or when it switches (no hold time, no fade, no smoothing: that would be a separate patch).
- Any other function, including `UpdateTrail`.
- Hiding other components of a dying missile.

---

## 3. Design (decided; do not change)

### 3.1 The corrected function
`UpdateSmartTail` as built in 9c.1:
```
Branch(bSmart)
    True  → TailSmart.SetVisibility(New Visibility = NOT bDodging, Propagate To Children = false)
    False → (nothing)
```
becomes
```
Branch(bSmart AND bActive)
    True  → TailSmart.SetVisibility(New Visibility = NOT bDodging, Propagate To Children = false)
    False → (nothing)
```
Both inputs of the AND are plain member booleans of the same Blueprint (`bSmart`, `bActive`); no `IsValid` or property access of another object is involved, so one AND node is enough. A missile that has been killed or has landed has `bActive` false (this is how `UpdateTrail` already protects itself), so the function then does nothing and never touches `TailSmart`.

### 3.2 Cases for the trace (the whole behaviour)

| # | `bSmart` | `bActive` | `bDodging` | AND | `SetVisibility` call | Result |
|---|---|---|---|---|---|---|
| 1 | false | true | false | false | none | `TailSmart` untouched (ordinary missile) |
| 2 | true | true | false | true | New Visibility = **true** | tail shown (flying freely) |
| 3 | true | true | true | true | New Visibility = **false** | tail hidden (dodging) |
| 4 | true | **false** | any | false | **none** | **the dying smart bomb: no access to `TailSmart`, no error** |

---

## 4. What this phase changes in earlier specs

| Earlier text | Now |
|---|---|
| 9c.1 3.2: `Branch(bSmart)` | `Branch(bSmart AND bActive)`. Nothing else in 9c.1 changes. |
| 9c.1 pitfall list | Add: a function that touches a component of the missile after `Advance` in `Event Tick` must start with the `bActive` check, because `Advance` can destroy the actor in that tick. |

---

## 5. Tooling notes

1. The edit is: create one boolean AND node; wire `bSmart` getter → input A, `bActive` getter → input B, AND output → the existing `Branch`'s **Condition** pin. Connecting the AND output to the `Branch`'s Condition replaces the old link from the `bSmart` getter (a data input takes one link). If the old `bSmart` getter node is left unused, leave it, or reuse it as the AND's input A.
2. `bActive` is the member boolean `UpdateTrail` already tests. Use a getter of the member.
3. Read all pins back with the node-reading tools after the edit (rule 13).

---

## 6. Implementation steps

### Step 0 — Read (no changes)
1. **`UpdateSmartTail`** exactly as built: `read_graph_dsl` plus every node's input pins; record whether the 9c.1 fallback (`Branch(bDodging)` with two `SetVisibility` calls) was used. **If the function differs from 3.1** (extra nodes, another structure), stop and report what it is.
2. **`Event Tick`:** the order `Advance` → `UpdateTrail` → `UpdateSmartTail`, and what follows.
3. **`UpdateTrail`:** confirm its first node is the `bActive` check (`If NOT bActive → return`).
4. **`Die` and `Impact`:** confirm that each sets `bActive` to false **before** it destroys the actor or ends. Record the node order of each. (Do not change them.) If either path does not set `bActive` false before the actor is destroyed, report it as a risk; the guard would then not cover that path.
5. **`bActive`:** type, default, member scope.

**Check:** the report section for Step 0 is complete.

### Step 1 — The edit
1. Create the boolean AND node (`create_node` if its ID has parentheses). Wire it as in tooling note 1.
2. If the AND node cannot be created: **Fallback** — insert a **new `Branch`** with condition `bActive` at the start of the function: the function's entry execution → the new `Branch`; **True** → the existing `Branch(bSmart)`; **False** → nothing. Connect with `connect_pins`; read back. Record the use of the fallback.
3. Compile. Read `UpdateSmartTail` back and list the pins (rule 13): AND (A = `bSmart`, B = `bActive`, output = `Branch` Condition); `SetVisibility` (Target `TailSmart`, New Visibility = NOT `bDodging`, Propagate To Children false); NOT (input `bDodging`).
4. **Trace** the corrected function with the four cases of 3.2 and write, for each, the value on every pin and whether `SetVisibility` is called. Case 4 must show **no call**.

**Check:** `BP_EnemyMissile` compiles with 0 errors and 0 warnings; `UpdateSmartTail` reads back as 3.1; the trace matches 3.2 in all four cases; `Event Tick` still reads `Advance` → `UpdateTrail` → `UpdateSmartTail` → (old successor); `TailSmart` flags unchanged; every item of rule 3 reads exactly as before. **Save All.**

### Step 2 — Run
1. **Save All.** `StartPIE`; the title screen appears at once; **wait about 20 seconds**; nobody taps, so no game starts and no missile exists.
2. `CaptureEditorImage` at about 5 s and about 15 s; describe each (the title screen, unchanged).
3. `GetLogEntries` (filter `[DD]`), then `StopPIE`.
4. Fill in the report (section 8).

---

## 7. Acceptance tests

### A. Static checks
- [ ] **A1** Changed assets: `BP_EnemyMissile` only. No other asset, no material or instance, nothing under `Spike`, no Project Settings, input settings or `.ini` change. `L_Game` unchanged.
- [ ] **A2** `BP_EnemyMissile` compiles with 0 errors and 0 warnings.
- [ ] **A3** Step 0 findings are in the report: `UpdateSmartTail` as built, `Event Tick` order, `UpdateTrail`'s first node, the order of `bActive = false` and destruction in `Die` and `Impact`.
- [ ] **A4** `UpdateSmartTail` reads back as 3.1 (`Branch(bSmart AND bActive)`, or the recorded fallback); the rest of the function is as in 9c.1.
- [ ] **A5** **Pin audit (rule 13):** a table of every input pin of the AND node, the `Branch`, the `SetVisibility` call and the NOT node (node, pin, wired from / value).
- [ ] **A6** The trace table of 3.2 is in the report with the **actual** values and matches, **including case 4 (no `SetVisibility` call)**.
- [ ] **A7** **Visibility (rule 12):** `TailSmart` before and after: `bVisible` false, `bHiddenInGame` false, Movable, parent `TailPivot`, material `MI_DD_TailSmart`, transform unchanged.
- [ ] **A8** **Hands-off audit (rule 3):** a table that shows, for each item of rule 3, that it reads exactly as before this patch.

### B. Log checks (agent; the one 20 s run, nobody taps)
- [ ] **B1** The startup lines as in the 9c.1 run (camera, GameMode, config `Width=2250.0 Height=1800.0`, bases 3, cities 6, PlayerController, view target, crosshair, HUD, `[DD][Area] Viewport=...`).
- [ ] **B2** Exactly one of `[DD][Save] Loaded high=<n>` or `[DD][Save] No file high=<n>`, followed by exactly one `[DD][Flow] Title high=<n>`.
- [ ] **B3** During the 20 s wait: no `Spawner started`, no `[DD][Wave]`, `[DD][Enemy]`, `[DD][Flyer]`, `[DD][Score]`, `[DD][Damage]`, `[DD][Input] Fire request`, `[DD][Aim]`, `[DD][Game]`, and no `[DD][Flow]` line except the one `Title` line.
- [ ] **B4** No line containing `ERROR`, `Accessed None`, `Divide by zero` or `Blueprint Runtime Error`.

### C. Visual checks (captures; describe in words)
- [ ] **C1** The title screen is unchanged. No new object appears.

### D. Interactive checks (human, PIE) — all NEEDS-HUMAN
List every item as NEEDS-HUMAN in the report; do not run or claim them.
- [ ] **D1** **The defect is gone:** play, press `S` several times and **kill at least three smart bombs**. The play log contains **no** `Blueprint Runtime Error` and no `Attempted to access TailSmart` (the 9c.1 play log had one per kill).
- [ ] **D2** **Landing:** let one smart bomb reach the ground. No runtime error in the log.
- [ ] **D3** **Tail unchanged:** free flight shows the magenta tail; it disappears while the bomb dodges and returns afterwards (the checks D1 to D4 of 9c.1).
- [ ] **D4** **Other missiles unchanged:** ordinary missiles, MIRV warheads, flyer-dropped missiles and ABMs show their tails exactly as before.
- [ ] **D5** **No gameplay change:** dodge path, kills, the smart-bomb score and the `[DD][Enemy] Dodge` lines behave as before.

### D-DEVICE. Android (NEEDS-DEVICE)
- [ ] **T-1** The tail behaves as in D3 on the phone.

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped; the editor is on `L_Game`; nothing under `Spike` was touched.

---

## 8. Report format (the agent's final message)

```
## Phase 9c.1.1 report (active guard for UpdateSmartTail)
**Engine version:** x.y.z
**MCP tools used:** (list)

### Step 0 findings
(UpdateSmartTail as built; Event Tick order; UpdateTrail first node; Die and Impact: where bActive becomes false relative to the destruction; bActive)

### The edit (Step 1)
(UpdateSmartTail read back as DSL; whether the fallback was used)

### Trace table
(the table of 3.2 with the actual values of the built graph)

### Pin audit (rule 13)
(table: node | pin | wired from / value)

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
| A guarded `UpdateSmartTail` | Any function added to `Event Tick` after `Advance` must start with the `bActive` check. |

**Known leftovers that stay:** `StopSpawner`, `bDirectorSilent`, the two disconnected spawn chains in `SpawnRandomEnemy`, the disconnected older fire chain in `LaunchAt`, HUD debug lines 2 to 4, `BP_PlayfieldDebug`, the debug keys, the unused `MI_DD_Enemy` and `MI_DD_Abm`, the `Spike` folder (removed in 9g).

---

## 10. Common pitfalls

1. **Guarding the wrong function.** The fix goes in `UpdateSmartTail`. `UpdateTrail` already has its guard; do not touch it.
2. **Wiring `bActive` to the `SetVisibility` call instead of the `Branch`.** The check must stop the call from running at all, not feed its New Visibility.
3. **Rewriting the function** with the graph-writing tool (rule 10). One node and its wires.
4. **Wiring the AND output to the wrong pin, or leaving the old `bSmart` link in place.** A data pin takes one link; read the Branch's Condition pin back and confirm its source is the AND.
5. **Using `IsValid(TailSmart)` instead.** It would hide the error but leave the function running on a dead actor; the `bActive` guard is the project's pattern.
6. **Changing `bHiddenInGame` or any component flag** (rule 12).
7. **Touching `Die`, `Impact`, `Advance`, `ArmSmart`, `UpdateTrail`, a material or `Spike`**, or cleaning up what this document does not list.
8. **Marking D or T items PASS.** You cannot reproduce the error in PIE. They are NEEDS-HUMAN and NEEDS-DEVICE.
