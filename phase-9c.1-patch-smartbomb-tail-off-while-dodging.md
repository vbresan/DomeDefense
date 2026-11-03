# Phase 9c.1 (patch) — Smart-Bomb Tail Off While Dodging

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Depends on** | Phase 9c (built: round heads, halos, fading comet tails). The project is exactly at the 9c state. Any "bullet-shaped missile" work (components named `BulletBody`, `BulletNose`, `SmartBody`, `SmartNose`, functions `SteerBullet`, `ArmBullet`, config value `SmartBombTurnRate`) is **not** part of the project. |
| **Phase goal** | **Visual only. No gameplay change.** While a smart bomb is dodging an explosion, its **tail is switched off** (`TailSmart` invisible). When the bomb no longer senses an explosion, the tail is **switched on again**. The smart bomb's head (`DotSmart`) and halo (`HaloSmart`) stay visible the whole time. Ordinary missiles, MIRV warheads, flyer-dropped missiles and ABMs are not affected. One new function and one added call, both in `BP_EnemyMissile`. |

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **No gameplay change.** Positions, speeds, targets, kills, impacts, scoring, waves, flyers, MIRV splits, smart-bomb sensing and dodging, and game flow are unchanged. A smart bomb dodges exactly as before; only the visibility of one component changes.
3. **HANDS OFF.** Do not change, and report as unchanged after the work: every component of `BP_EnemyMissile` (`Root`, `Dot`, `DotSmart`, `Halo`, `HaloSmart`, `TailPivot`, `Tail`, `TailSmart`: transform, mesh, material, flags); the variables `TrailLen`, `TrailWidth`, `bSmart`, `bDodging`, `SenseRadius`, `DodgeSpeed`; the functions `AimTrail`, `UpdateTrail`, `ArmSmart`, `ApplyDodge`, `ComputePush`, `Advance`, `Die`, `Impact`, `Split`, `CheckSplit`, `Init`, `IsInsideExplosion`; **every material and material instance**.
4. **Existing logic must survive.** Read each graph (`read_graph_dsl`) before editing it.
5. **Compile after every Blueprint change.** `BP_EnemyMissile` ends with no errors and no warnings.
6. **Run each step's "Check"** before moving on. **Save All after each step's Check passes.**
7. **You cannot press keys, click, touch, move the mouse, or resize in PIE, and you cannot start a game.** No missile exists on the title screen, so **you cannot see a smart bomb or its tail in PIE.** You can verify: compiles, static read-backs, and a **pin-by-pin trace** of the new function with the cases of 3.3. Everything seen in play is group D (**NEEDS-HUMAN**); device checks are **NEEDS-DEVICE**. Never mark a D or T item PASS.
8. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
9. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after the run. **Save before loading a level.** **Do not touch Project Settings, input settings or any `Config/*.ini` file.** **Do not use, edit or delete anything in `/Game/DomeDefense/Spike/`.**
10. **NEVER REWRITE AN EXISTING FUNCTION OR EVENT GRAPH** with `write_graph_dsl`. Existing graphs are changed only by small node or pin edits (`create_node`, `connect_pins`, `set_pin_value`). The new function of Step 1 is written **once** as a brand-new function.
11. **READ EVERY NEW OR CHANGED GRAPH BACK** right after changing it. If it reads back wrong or incomplete: correct it once more; if it is still wrong, **stop and report** under *Tool problems*.
12. **VISIBILITY.** No component is added or changed. `TailSmart` keeps `bVisible` false (its default), **`bHiddenInGame` false** and Movable. The new function uses `SetVisibility` only; it never calls `SetHiddenInGame` and never changes `bHiddenInGame`. Read the flags of `TailSmart` back before and after and list them in the report.
13. **CHECK EVERY PIN YOU BUILD** (the lesson of the Phase 7 pause defect: a boolean pin left at its default). After building the `SetVisibility` call, read all its input pins back and list them in the report (A7): **Target** (`TailSmart`), **New Visibility** (fed by the NOT of `bDodging`, **not** a literal), **Propagate To Children** (explicitly false).
14. **After any tool error, do not retry the same large operation.** Stop that item, try one smaller piece at most, and if it still fails **stop and report**. **If the editor stops responding or restarts, stop and report.**
15. **Order is part of the spec:** read first, then the function, then the hook, then the run.
16. **FROZEN: do not edit** every asset except `BP_EnemyMissile`. In particular frozen: `BP_ABM`, `BP_DDGameMode`, `BP_DDPlayerController`, `BP_DDHUD`, `BP_Flyer`, `BP_Explosion`, `BP_MissileBase`, `PDA_DDConfig`, `DA_DDConfig`, `L_Game`, and every material and material instance.
17. **No new keys, no new log lines, no new config values, no new variables, no new components, no new Tick, no new asset.** The only additions are the function `UpdateSmartTail` and one call to it.
18. **Node IDs with parentheses cannot be written in the graph DSL.** If a node you need has one (the boolean NOT node, for example), create it with `create_node` and connect it with `connect_pins`.
19. **Do not clean up anything this document does not list.**

---

## 2. Scope

### In scope
- `BP_EnemyMissile`: one new function `UpdateSmartTail`; one call to it in `Event Tick`.

### Out of scope (do NOT do)
- Hiding or changing the smart bomb's head, halo, colours, sizes or the tail's length, width or fade.
- Any change to how any missile moves, senses or dodges; any change to ordinary missiles, MIRV warheads, flyer-dropped missiles or ABMs.
- Fading the tail in or out (it switches on and off instantly).
- Any change to `ArmSmart`, `UpdateTrail`, `ApplyDodge` or any other existing function.

---

## 3. Design (decided; do not change)

### 3.1 What the player sees
A smart bomb is a `BP_EnemyMissile` with `bSmart` true (8c). It shows a magenta head (`DotSmart`), a magenta halo (`HaloSmart`) and a magenta fading tail (`TailSmart`) (9c).

- While the bomb **flies without sensing an explosion**: head, halo and tail are visible (as in 9c).
- While the bomb **dodges** (it senses at least one explosion): the **tail is not drawn**. Head and halo stay.
- When the bomb **stops sensing** any explosion: the tail is drawn again at once.

"Dodging" is exactly the existing variable **`bDodging`**: `ApplyDodge` sets `bDodging = sensed` at its end, once per tick, and `sensed` is true when at least one active explosion is closer than `E.Radius + SenseRadius` (8c). Nothing about how it is computed changes.

### 3.2 The function
New function `UpdateSmartTail` in `BP_EnemyMissile`: non-Pure, no inputs, no outputs, no local variables.

```
Branch(bSmart)
    True  → TailSmart.SetVisibility(New Visibility = NOT bDodging, Propagate To Children = false)
    False → (nothing; the function ends)
```

- It is called **every tick** from `Event Tick`, **immediately after `UpdateTrail()`** (which itself comes after `Advance(DeltaSeconds)`). At that point `ApplyDodge` has already run for this frame, so `bDodging` is current.
- It does nothing for a missile with `bSmart` false: `TailSmart` of an ordinary missile is never touched and stays `bVisible` false. (Without the `bSmart` branch every ordinary missile would show a magenta tail, because its `bDodging` is false and `NOT false` is true.)
- Setting a component's visibility to the value it already has is a no-op in the engine, so calling it every tick costs nothing and needs no extra state variable.
- `UpdateTrail` keeps updating `TailSmart`'s location and scale while it is hidden. When it reappears it already has the correct length.

### 3.3 Cases for the trace (the whole behaviour)

| # | `bSmart` | `bDodging` | Branch | `SetVisibility` call | Result |
|---|---|---|---|---|---|
| 1 | false | false | False | none | `TailSmart` untouched (ordinary missile) |
| 2 | true | false | True | New Visibility = NOT false = **true** | tail shown (flying freely) |
| 3 | true | true | True | New Visibility = NOT true = **false** | tail hidden (dodging) |
| 4 | true | false, the frame after case 3 | True | New Visibility = **true** | tail shown again |

### 3.4 What the human should expect (so it is not mistaken for a defect)
- A smart bomb that dodges for a long time has no tail for that time. That is intended.
- When the tail returns it is still aimed along the bomb's **original** line (9c orients the tail once, at spawn). If the bomb has been pushed sideways, the tail can be a few degrees off the bomb's actual heading. That is 9c behaviour and is not changed here.

---

## 4. What this phase changes in earlier specs

| Earlier text | Now |
|---|---|
| 9c 3.10 / `ArmSmart`: `TailSmart.SetVisibility(true)` and the tail stays visible for the bomb's life | `ArmSmart` is unchanged and still switches `TailSmart` on. From then on `UpdateSmartTail` switches it off while `bDodging` is true and on again when it is false. |
| 9c `BP_EnemyMissile.Event Tick`: `UpdateTrail()` after `Advance(DeltaSeconds)` | One more call, `UpdateSmartTail()`, directly after `UpdateTrail()`. |
| 9c frozen list | `BP_EnemyMissile` is unfrozen **for this edit only**. |

---

## 5. Tooling notes

1. A new function is written once with the graph-writing tool, then read back. Writes sometimes read back empty: read it back immediately; if it is empty, write it **once more**, and if it is still empty, stop and report (rules 11 and 14).
2. To insert a call into `Event Tick` without rewriting it: first read which node `UpdateTrail()`'s execution output currently leads to (call it `NEXT`, or "nothing"). Then `connect_pins`: `UpdateTrail.then → UpdateSmartTail.exec`, then `UpdateSmartTail.then → NEXT` (if there is a `NEXT`). Connecting a connected execution output replaces its old link, so do the second connection right away. Read `Event Tick` back and confirm nothing downstream was lost.
3. `SetVisibility` is the **Set Visibility** node of a scene component (inputs: Target, New Visibility, Propagate To Children). Read all three input pins back.
4. The boolean NOT node has a parenthesised ID in some tool versions: create it with `create_node`.
5. A member variable read in a function that has no local of the same name is simply the member. This function has no locals.

---

## 6. Implementation steps

### Step 0 — Read (no changes)
1. **Confirm the project state.** `BP_EnemyMissile` has the components `Root`, `Dot`, `DotSmart`, `Halo`, `HaloSmart`, `TailPivot`, `Tail`, `TailSmart` and **no** `BulletBody`, `BulletNose`, `SmartBody`, `SmartNose`; it has **no** function `SteerBullet` or `ArmBullet`; `PDA_DDConfig` has **no** `SmartBombTurnRate`. **If any of these bullet-phase items exists, stop and report**: this patch assumes the project at 9c.
2. **Read the graphs:** `Event Tick` (where `Advance` and `UpdateTrail` sit; **which node `UpdateTrail()` leads to: record it**), `UpdateTrail`, `ArmSmart`, `ApplyDodge`, and the place in `Advance` where `ApplyDodge` is called.
3. **`ApplyDodge`:** record every node that writes `bDodging`. Expected: exactly one Set at the end (`bDodging = sensed`), executed on every tick of a smart bomb whether or not any explosion exists. **If there is any path on which a smart bomb does not reach that Set while no explosion is sensed** (an early return, for example), report it as a risk: the tail could stay hidden. Do not change `ApplyDodge`.
4. **Variables:** `bSmart` and `bDodging` (type, default, member scope).
5. **`TailSmart`:** parent, mesh, material, relative transform, `bVisible`, `bHiddenInGame`, `Mobility`.

**Check:** the report section for Step 0 is complete, including the node `UpdateTrail()` leads to.

### Step 1 — The function and the hook
1. **Write `UpdateSmartTail`** (3.2), once, non-Pure. Build the **NOT** node with `create_node` if needed. Read it back; compile.
   - **Fallback** (only if a boolean NOT node cannot be created): `Branch(bDodging)`: **True** → `TailSmart.SetVisibility(false, false)`; **False** → `TailSmart.SetVisibility(true, false)`, inside the `bSmart` True branch. Set each literal explicitly and read it back (a `false` that equals the default must still be read back as set). Record the use of the fallback.
2. **Pin audit (rule 13):** read back all input pins of the `Branch`, the `SetVisibility` call and the NOT node: Branch condition = `bSmart` (member getter); `SetVisibility.Target` = `TailSmart`; `New Visibility` = output of NOT; NOT input = `bDodging` (member getter); `Propagate To Children` = false.
3. **Hook** (tooling note 2): `Event Tick`: `UpdateTrail()` → `UpdateSmartTail()` → (the node recorded in Step 0.2). Read `Event Tick` back: `Advance(DeltaSeconds)`, `UpdateTrail()`, `UpdateSmartTail()` in that order and everything else as before.
4. Compile.
5. **Trace** the function with the four cases of 3.3 and write, for each case, the value on every pin and what `SetVisibility` receives (the actual wiring, not the intended one).

**Check:** `BP_EnemyMissile` compiles with 0 errors and 0 warnings; `UpdateSmartTail` reads back non-empty and matches 3.2; the trace matches 3.3 in all four cases; `UpdateSmartTail` is called from `Event Tick` only; `TailSmart` flags unchanged (`bVisible` false, `bHiddenInGame` false, Movable); `ArmSmart`, `UpdateTrail`, `AimTrail`, `ApplyDodge`, `ComputePush`, `Advance`, `Die`, `Impact`, `Split`, `CheckSplit`, `Init` read exactly as before; both tick flags still true. **Save All.**

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
- [ ] **A3** Step 0 findings are in the report: the project is at 9c (no bullet-phase items), the node `UpdateTrail()` led to, and the list of nodes that write `bDodging` (with the risk statement of Step 0.3, or "none").
- [ ] **A4** `UpdateSmartTail` reads back as 3.2: `Branch(bSmart)`; True → one `SetVisibility` call on `TailSmart`; False → nothing; no locals; no inputs or outputs; or the recorded fallback.
- [ ] **A5** The trace table of 3.3 is in the report with the **actual** values of the built graph for all four cases and matches the expected results.
- [ ] **A6** `Event Tick` reads back: `Advance(DeltaSeconds)` → `UpdateTrail()` → `UpdateSmartTail()` → (old successor, if any); `UpdateSmartTail` is called nowhere else.
- [ ] **A7** **Pin audit (rule 13):** a table of every input pin of the `Branch`, the `SetVisibility` call and the NOT node (node, pin, wired from / value). `New Visibility` is fed by the NOT of `bDodging`, not by a literal.
- [ ] **A8** **Visibility (rule 12):** `TailSmart` before and after: `bVisible` false, `bHiddenInGame` false, `Mobility` Movable, parent `TailPivot`, material `MI_DD_TailSmart`, transform unchanged.
- [ ] **A9** **Hands-off audit (rule 3):** a table that shows, for each item of rule 3, that it reads exactly as before this patch.

### B. Log checks (agent; the one 20 s run, nobody taps)
- [ ] **B1** The startup lines as in the 9c run (camera, GameMode, config `Width=2250.0 Height=1800.0`, bases 3, cities 6, PlayerController, view target, crosshair, HUD, `[DD][Area] Viewport=...`).
- [ ] **B2** Exactly one of `[DD][Save] Loaded high=<n>` or `[DD][Save] No file high=<n>`, followed by exactly one `[DD][Flow] Title high=<n>`.
- [ ] **B3** During the 20 s wait: no `Spawner started`, no `[DD][Wave]`, `[DD][Enemy]`, `[DD][Flyer]`, `[DD][Score]`, `[DD][Damage]`, `[DD][Input] Fire request`, `[DD][Aim]`, `[DD][Game]`, and no `[DD][Flow]` line except the one `Title` line.
- [ ] **B4** No line containing `ERROR`, `Accessed None`, `Divide by zero` or `Blueprint Runtime Error`.

### C. Visual checks (captures; describe in words)
- [ ] **C1** The title screen is unchanged. No new object appears.

### D. Interactive checks (human, PIE) — all NEEDS-HUMAN
List every item as NEEDS-HUMAN in the report; do not run or claim them.
- [ ] **D1** **Free flight:** start a game, press `S` (smart bomb). Before any explosion is near it, the magenta head, halo **and tail** are visible, as in 9c.
- [ ] **D2** **Dodge:** tap so that an explosion grows near the bomb's path. When the bomb starts to dodge (it swerves away), its **tail disappears**; head and halo stay.
- [ ] **D3** **Return:** when the explosion is gone or the bomb has left its range, the **tail reappears**. After the last explosion the tail is **never stuck hidden**.
- [ ] **D4** **Repeat:** several dodges in a row each switch the tail off and on again.
- [ ] **D5** **Other missiles unchanged:** ordinary red missiles, MIRV warheads, flyer-dropped missiles and ABMs show their tails exactly as in 9c. No magenta tail on any ordinary missile.
- [ ] **D6** **No gameplay change:** the dodge path, kills, the smart-bomb score and `[DD][Enemy] Dodge` lines (rising edge only) behave as before. A smart bomb killed while dodging dies normally.
- [ ] **D7** A busy late wave runs without a hitch.

### D-DEVICE. Android (NEEDS-DEVICE)
- [ ] **T-1** The same behaviour as D1 to D3 on the phone.

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped; the editor is on `L_Game`; nothing under `Spike` was touched.

---

## 8. Report format (the agent's final message)

```
## Phase 9c.1 report (smart-bomb tail off while dodging)
**Engine version:** x.y.z
**MCP tools used:** (list)

### Step 0 findings
(project state; Event Tick order and the node UpdateTrail() led to; every node that writes bDodging and the risk statement; bSmart / bDodging; TailSmart flags)

### Function and hook (Step 1)
(UpdateSmartTail read back as DSL; whether the fallback was used; Event Tick read back)

### Trace table
(the table of 3.3 with the actual values of the built graph)

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
| `UpdateSmartTail` | If the halo or head should also react to a dodge, add the same call pattern to this function. |

**Known leftovers that stay:** `StopSpawner`, `bDirectorSilent`, the two disconnected spawn chains in `SpawnRandomEnemy`, the disconnected older fire chain in `LaunchAt`, HUD debug lines 2 to 4, `BP_PlayfieldDebug`, the debug keys, the unused `MI_DD_Enemy` and `MI_DD_Abm`, the `Spike` folder (removed in 9g).

---

## 10. Common pitfalls

1. **No `bSmart` branch.** Ordinary missiles have `bDodging` false, so `NOT bDodging` is true and every ordinary missile would get a visible magenta `TailSmart`. The `bSmart` branch is required.
2. **`New Visibility` left at its default** (false) or fed by a literal instead of `NOT bDodging`. The tail would never show, or never hide. Read the pin back (rule 13).
3. **`bDodging` not negated.** The tail would show only while dodging.
4. **`Propagate To Children` left to a default that is true.** `TailSmart` has no children, so it is harmless, but the pin is set explicitly false like every other `SetVisibility` call in the project.
5. **Changing `bHiddenInGame`** (for example with `SetHiddenInGame`). `bHiddenInGame` stays false everywhere; only `bVisible` is toggled (the invisible smart bomb of 8c).
6. **Calling the function before `Advance`.** `bDodging` would be one frame stale. It goes after `UpdateTrail()`.
7. **Losing the rest of `Event Tick`.** Connecting `UpdateTrail.then` to the new node replaces its old link. Reconnect `UpdateSmartTail.then` to the node recorded in Step 0 and read the graph back.
8. **Rewriting `Event Tick` or any existing function** with the graph-writing tool (rule 10).
9. **Touching `ArmSmart`, `UpdateTrail`, `ApplyDodge`, a material, a component or `Spike`**, or cleaning up what this document does not list.
10. **Marking D or T items PASS.** You cannot see a smart bomb in PIE. They are NEEDS-HUMAN and NEEDS-DEVICE.
