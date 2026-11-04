# Phase 9d.2 — Look A in the Real Explosions (Enemy Orange, Yours Cyan-White) and 9d.1 Cleanup

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Depends on** | Phase 9d.1 (built and **committed**; repository `master` at commit "Explosions experiment"). The project contains `BP_ExplosionPreview`, five `M_DD_Exp*` materials with their `MI_DD_Exp*` instances, and the three preview functions and eight keys (`Zero` to `Seven`) in `BP_DDPlayerController`. |
| **Phase goal** | **Visual only. No gameplay change.** The human chose **look A** (a glowing disc with a white-hot core and a soft glow behind it) with its two colour schemes. (1) The real explosion, `BP_Explosion`, is drawn with look A: **cyan-white for explosions that are player made** (the existing `bPlayerMade` flag, which also covers chain reactions started by the player's blasts) and **orange for all others** (enemy impacts and chain reactions started by them). (2) Everything else from 9d.1 is **deleted**: the preview actor, the fire, flash and ember materials, the eight keys and the three preview functions. Radius, timing, the kill logic and everything else in the game stay exactly as they are. |

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **No gameplay change.** The explosion's radius (`Radius`, `MaxRadius`/`PeakRadius`), its timing (`GrowTime`, `HoldTime`, `ShrinkTime`), `RadiusAtAge`, `ContainsPoint`, the registration in `ActiveExplosions`, the log lines (`[DD][Explosion] Start/End ... max=160 ... life≈1300ms`) and the `bPlayerMade` logic are **not changed**. The new layers have no collision.
3. **HANDS OFF.** Do not change, and report as unchanged after the work: `BP_DDGameMode`, `BP_DDHUD`, `BP_EnemyMissile`, `BP_ABM`, `BP_Flyer`, `BP_TapMarker`, `BP_City`, `BP_MissileBase`, `BP_Crosshair`, `PDA_DDConfig`, `DA_DDConfig`, `L_Game`, `M_DD_ExpDisc`, `M_DD_ExpGlow`, `MI_DD_ExpDisc`, `MI_DD_ExpGlow`, `MI_DD_Explosion` and every other existing material and instance; in `BP_Explosion` everything except the additions and the one hidden flag of Step 1; in `BP_DDPlayerController` everything except the removals of Step 2.
4. **NO SCREENSHOTS.** Do not use `CaptureViewport` or `CaptureEditorImage`, and do not place temporary actors in the level. Verify only by compiling and by reading assets back.
5. **Existing logic must survive.** Read each graph (`read_graph_dsl`) before editing it.
6. **Compile after every Blueprint change.** Every Blueprint ends with no errors and no warnings.
7. **Run each step's "Check"** before moving on. **Save All after each step's Check passes.**
8. **You cannot press keys, click, touch, move the mouse, or resize in PIE, and you cannot start a game.** No explosion exists on the title screen. Everything seen in play is group D (**NEEDS-HUMAN**). Never mark a D item PASS.
9. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
10. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after the run. **Save before loading a level.** **Do not touch Project Settings, input settings or any `Config/*.ini` file.** **Do not use, edit or delete anything in `/Game/DomeDefense/Spike/`.**
11. **NEVER REWRITE AN EXISTING FUNCTION OR EVENT GRAPH** with `write_graph_dsl` (`Init`, `SetRadius`, `Advance`, `RadiusAtAge`, `Finish`, `ContainsPoint` of `BP_Explosion`, and every graph of `BP_DDPlayerController`). The two new functions of 3.3 are written once with `write_graph_dsl` and read back. The one call inserted into `SetRadius` is added with `create_node` and `connect_pins`.
12. **READ EVERY NEW OR CHANGED GRAPH BACK** right after writing it. If it reads back wrong or incomplete: correct it once more; if it is still wrong, **stop and report** under *Tool problems*.
13. **CHECK EVERY PIN YOU BUILD** (the lesson of the Phase 7 pause defect: a pin left at its default). After building the new functions and the inserted call, read all input pins of every new node back and list them (A8).
14. **VISIBILITY AUDIT.** Both new components have **`bHiddenInGame` FALSE**, `bVisible` TRUE, collision `NoCollision`, `CastShadow` false, and are **Movable**. The only edit to an existing component is `Disc`: `bVisible` becomes false (its mesh, material, location and scale stay). Read the flags back for the new components and `Disc` and list them (A7).
15. **No cheap shortcuts in materials:** the two materials stay as they are; no material is created or edited in this phase.
16. **After any tool error, do not retry the same large operation.** Stop that item, try one smaller piece at most, and if it still fails **stop and report**. **If the editor stops responding or restarts, stop and report.**
17. **Order is part of the spec:** read first, then `BP_Explosion`, then the controller removals, then the asset deletions, then the run. **Never delete an asset that is still referenced** (check references first, rule 18).
18. **Deletions:** before deleting each asset, list its referencers (`AssetTools`). The only allowed referencers are the other assets deleted in this phase. If anything else refers to it, **stop and report** instead of deleting. After the deletions, list `/Game/DomeDefense/Actors`, `/Game/DomeDefense/Materials` and `/Game/DomeDefense/Core` and compare with the expected final set of 3.5. Delete nothing that is not listed in 3.4.
19. **No new keys, no new log lines, no new config values.** The only new variables are the six palette colours of 3.2; the only new functions are `UpdateLook` and `ApplyPalette`.
20. **If the project is not in the state described in "Depends on"** (a name of 3.2 or 3.3 already exists in `BP_Explosion`, an asset of 3.4 is missing), **stop and report** before changing anything.

---

## 2. Scope

### In scope
- `BP_Explosion`: two new layers, six palette variables, two new functions, one inserted call, `Disc` hidden.
- `BP_DDPlayerController`: removal of the eight preview keys and the three preview functions.
- Deletion of the 9d.1 assets that are not needed.

### Out of scope (do NOT do)
- Any change to the explosion's size, timing, kill logic, score, chain-reaction rules or log lines.
- Any change to `M_DD_ExpDisc`, `M_DD_ExpGlow` and their instances (their tuned values stay).
- Removing `Disc` or `MI_DD_Explosion` (they stay hidden and unused until 9g).
- Anything else in the game.

---

## 3. Design (decided; do not change)

### 3.1 The look
Look A of 9d.1 is exactly two layers, drawn on planes that face the camera and scaled to the explosion's current radius:

| Layer | Material | Depth (Y) |
|---|---|---|
| `LayerGlow` | `MI_DD_ExpGlow` | 89 |
| `LayerBody` | `MI_DD_ExpDisc` | 90 |

Both materials are zero for r >= 1, so nothing is drawn outside the circle of the explosion's radius. The depth of `LayerBody` (90) equals the depth of the old `Disc`, so the layering against missiles, ABMs, cities and the target marker is unchanged.

### 3.2 `BP_Explosion`: additions
**Components** (children of `Root`; mesh `/Engine/BasicShapes/Plane`, which is 100 x 100; rotation **Roll 90**, as the missile halos have it; collision `NoCollision`; `CastShadow` false; Movable; `bVisible` true; `bHiddenInGame` false):

| Component | Material | Location (X, Y, Z) | Initial scale |
|---|---|---|---|
| `LayerGlow` | `MI_DD_ExpGlow` | (0, 89, 0) | (0.01, 0.01, 1) |
| `LayerBody` | `MI_DD_ExpDisc` | (0, 90, 0) | (0.01, 0.01, 1) |

`Disc`: set `bVisible` to **false**. Nothing else on it changes.

**Variables** (linear colour, Instance Editable, category `Colors`; these are the 9d.1 palettes, the values are final):

| Variable | Value |
|---|---|
| `OrangeCore` | (1.0, 0.92, 0.70) |
| `OrangeMid` | (1.0, 0.55, 0.10) |
| `OrangeEdge` | (0.45, 0.08, 0.01) |
| `CyanCore` | (1.0, 1.0, 1.0) |
| `CyanMid` | (0.45, 0.90, 1.0) |
| `CyanEdge` | (0.05, 0.35, 0.75) |

### 3.3 `BP_Explosion`: new functions and the one inserted call
Write both once with `write_graph_dsl`, read back.

**Function `ApplyPalette()`**
```
core = Select(bPlayerMade, CyanCore, OrangeCore)
mid  = Select(bPlayerMade, CyanMid,  OrangeMid)
edge = Select(bPlayerMade, CyanEdge, OrangeEdge)
LayerBody.SetVectorParameterValueOnMaterials("CoreColor", core)
LayerBody.SetVectorParameterValueOnMaterials("MidColor",  mid)
LayerBody.SetVectorParameterValueOnMaterials("EdgeColor", edge)
LayerGlow.SetVectorParameterValueOnMaterials("MidColor",  mid)
```
(`bPlayerMade` is the existing variable; true selects the cyan-white palette.)

**Function `UpdateLook(NewRadius: float)`**
```
S    = Max(NewRadius, 0.5) / 50                 // the same rule SetRadius uses for Disc
LayerGlow.SetRelativeScale3D( MakeVector(S, S, 1) )
LayerBody.SetRelativeScale3D( MakeVector(S, S, 1) )
Fade = Select( Age < GrowTime + HoldTime, 1.0, Clamp(NewRadius / PEAK, 0, 1) )
LayerGlow.SetScalarParameterValueOnMaterials("Fade", Fade)
LayerBody.SetScalarParameterValueOnMaterials("Fade", Fade)
ApplyPalette()
```
`PEAK` is the variable that `RadiusAtAge` uses as the peak radius (`MaxRadius` or `PeakRadius`; Step 0 tells which). `Age`, `GrowTime`, `HoldTime` are the existing variables. In the shrink phase the radius falls linearly from the peak to zero, so `NewRadius / PEAK` fades the look from 1 to 0 over the shrink; in the grow and hold phases `Fade` is 1. (Re-pushing the palette every call is deliberate: the explosion's `bPlayerMade` is assigned by its spawner at a time you cannot rely on, and this way the colour is always right from the first visible frame. The cost is a handful of cheap calls per explosion per frame.)

**Inserted call in `SetRadius(NewRadius)`:** after the last node of its execution chain (the node that sets the `Disc` scale; Step 0 tells which that is), add `UpdateLook(NewRadius)` with its `NewRadius` pin wired from the `NewRadius` input of `SetRadius`. Do nothing else in `SetRadius`.

### 3.4 What is deleted
In this order (rule 18 applies to every asset):
1. In `BP_DDPlayerController`: the eight key events (`Zero`, `One`, `Two`, `Three`, `Four`, `Five`, `Six`, `Seven`, nodes `InpActEvt_<Key>_K2Node_InputKeyEvent`) **with the call nodes wired to them and every node that exists only to feed those calls** (for example the shared integer literals 0, 1, 2 and 3; a node that also feeds something that stays is kept), then the functions `SpawnPreviewGrid`, `SpawnPreviewAtCrosshair`, `SpawnExplosionPreview` (callers first: the grid and the crosshair function call `SpawnExplosionPreview`). Compile.
2. Assets: `BP_ExplosionPreview`, then `MI_DD_ExpFire`, `M_DD_ExpFire`, `MI_DD_ExpFlash`, `M_DD_ExpFlash`, `MI_DD_ExpEmber`, `M_DD_ExpEmber`.

### 3.5 Expected final set
`/Game/DomeDefense/Actors`: no `BP_ExplosionPreview`. `/Game/DomeDefense/Materials`: `M_DD_ExpDisc`, `M_DD_ExpGlow`, `MI_DD_ExpDisc`, `MI_DD_ExpGlow` and `MI_DD_Explosion` remain; `M_DD_ExpFire`, `M_DD_ExpFlash`, `M_DD_ExpEmber` and their three instances are gone. `BP_DDPlayerController`: its key events are those of the 9c.3 state (`H`, `R`, `E`, `X`, `V`, `K`, `N`, `LeftMouseButton`, `P`, `Delete`, `AndroidBack`, `B`, `M`, `F`, `G`, `S`, `InputTouch`) and no `Zero` to `Seven`.

---

## 4. What this phase changes in earlier specs

| Earlier text | Now |
|---|---|
| 9d.1: six preview looks on keys `0` to `7` | Removed. Look A is the real explosion's look. |
| The real explosion is one orange sphere (`Disc`) | Two additive layers (glow and body); `Disc` hidden. Cyan-white when `bPlayerMade`, orange otherwise. |
| 9d.1 frozen list | `BP_Explosion` and `BP_DDPlayerController` are unfrozen for the edits of this document only. |

---

## 5. Tooling notes

1. Components on a Blueprint: `ActorTools.add_component` (engine meshes by path), then `set_parent_component`; read parent and relative transform back after the re-parent. `CastShadow` (not `bCastShadow`) is the shadow property.
2. Component getters in DSL: `Variables|Default|Get<Name>` (as in 9d.1). `SetScalarParameterValueOnMaterials` and `SetVectorParameterValueOnMaterials` are nodes of mesh components and worked in 9d.1.
3. The `UpdateLook` call is added to `SetRadius` with `create_node` (the function-call node of `UpdateLook`) and `connect_pins`; read `SetRadius` back afterwards and show it before and after.
4. Deleting graph nodes and functions: use the Blueprint tools' delete operations for nodes and functions; read the controller's event graph and function list back afterwards.
5. `AssetTools` lists referencers and deletes assets. If an asset cannot be deleted, report it; do not use a workaround.
6. If a tool reports that a property cannot be read or set, say so; do not guess.

---

## 6. Implementation steps

### Step 0 — Read (no changes)
1. `BP_Explosion`: all components with flags; all variables; every function. Read `Init`, `SetRadius`, `Advance`, `RadiusAtAge`, `ReceiveTick`, `ReceiveBeginPlay` completely. Record: (a) which variable `RadiusAtAge` uses as the peak (`MaxRadius` or `PeakRadius`); (b) the exact content of `SetRadius` and which node is the last of its execution chain; (c) where `Age` is increased relative to the call of `SetRadius` (it must already be the new age when `SetRadius` runs; if it is not, record it, the `Fade` still works because the grow and hold test uses `Age < GrowTime + HoldTime`); (d) every place `Disc` is referenced; (e) where `bPlayerMade` is written (in `BP_Explosion` and by whoever spawns it; list the writers if they are visible).
2. Confirm the final values of `MI_DD_ExpDisc` and `MI_DD_ExpGlow` (parameter names and values: read only, they are not edited).
3. `BP_DDPlayerController`: list its key events and the three preview functions; list the nodes wired to each of the eight preview keys, and every node that feeds those calls (the shared integer literals) with everything else each one is wired to.
4. For each asset of 3.4 step 2: list its referencers.
5. Confirm none of the names of 3.2 and 3.3 exist in `BP_Explosion` yet.

**Check:** the report section for Step 0 is complete.

### Step 1 — `BP_Explosion`
Read the Blueprint again first.
1. Add `LayerGlow` and `LayerBody` (3.2). Set `Disc` `bVisible` false. Read back parent, transform, material and flags of the two new components and `Disc`.
2. Add the six palette variables (3.2). Read back names, types, defaults, flags.
3. Write `ApplyPalette` and `UpdateLook` (3.3). Read each back.
4. Insert the `UpdateLook(NewRadius)` call at the end of `SetRadius` (3.3). Read `SetRadius` back.
5. Compile.

**Check:** `BP_Explosion` compiles clean; components, variables and functions read back as specified; `SetRadius` reads exactly as before plus the one call; `Init`, `Advance`, `RadiusAtAge`, `Finish`, `ContainsPoint`, the events and every other variable read exactly as before; tick settings unchanged. **Save All.**

### Step 2 — `BP_DDPlayerController` (removals)
Read the controller again first.
1. Delete the eight key events with their wired call nodes and the nodes that only feed them (the shared integer literals 0, 1, 2, 3 and anything else that exists only for the preview keys; before deleting each, read what it is wired to, and keep it if it also feeds something that stays), then the three preview functions (3.4 step 1). Compile.
2. Read the event graph and the function list back.

**Check:** the controller compiles clean; its key events are exactly those listed in 3.5; no function whose name contains `Preview` exists; the event graph contains no leftover node from 9d.1 (no orphan literal, no unconnected node) and reads as it did in 9c.3; every other graph reads exactly as before. **Save All.**

### Step 3 — Delete the unused 9d.1 assets
Delete in the order of 3.4 step 2, checking the referencers before each deletion (rule 18). After the last deletion: compile `BP_Explosion` and `BP_DDPlayerController`, list the three folders and compare with 3.5.

**Check:** the asset set equals 3.5; both Blueprints compile clean. **Save All.**

### Step 4 — Run
1. **Save All.**
2. `StartPIE`. The title screen appears at once. **Wait about 20 seconds**; nobody taps, so no game starts and no explosion exists. **No screenshots.**
3. `GetLogEntries` (filter `[DD]`), then `StopPIE`.
4. Fill in the report (section 8).

---

## 7. Acceptance tests

### A. Static checks
- [ ] **A1** Changed assets: `BP_Explosion` and `BP_DDPlayerController`. Deleted: exactly the seven assets of 3.4 step 2. Nothing else: no existing material or instance edited, nothing under `Spike`, no Project Settings, input settings or `.ini` change. `L_Game` unchanged, and no temporary actor was placed.
- [ ] **A2** `BP_Explosion` and `BP_DDPlayerController` compile with 0 errors and 0 warnings.
- [ ] **A3** Step 0 findings are in the report (peak variable, `SetRadius` content, where `Age` is increased, references to `Disc`, writers of `bPlayerMade`, referencers of each asset to delete).
- [ ] **A4** `BP_Explosion`: `LayerGlow` and `LayerBody` as in 3.2; `Disc` hidden and otherwise unchanged; six palette variables with the 3.2 values.
- [ ] **A5** `ApplyPalette` and `UpdateLook` read back as in 3.3 with the right peak variable; `SetRadius` shown before and after (one added call).
- [ ] **A6** The controller: eight key events and three functions removed; the key events of 3.5 present and unchanged; no orphan node from 9d.1 remains in the event graph (the report lists each literal or helper node that was deleted and confirms it fed only the deleted calls).
- [ ] **A7** **Visibility audit:** a table of `LayerGlow`, `LayerBody` and `Disc` with `bVisible`, `bHiddenInGame`, `Mobility`, collision and `CastShadow`.
- [ ] **A8** **Pin audit (rule 13):** a table of every input pin of every node in the two new functions and of the inserted call (target, name and value pins, wired from).
- [ ] **A9** **Unchanged (rule 3):** a table that shows, for each item of rule 3, that it reads exactly as before, including `RadiusAtAge`, `ContainsPoint`, `Init`, `Advance`, the `[DD][Explosion]` log nodes and `ActiveExplosions` handling.
- [ ] **A10** The final asset set equals 3.5 (folder listings).

### B. Log checks (agent; the one 20 s run, nobody taps)
- [ ] **B1** The startup lines as in the 9d.1 run: camera, GameMode, config (`Width=2250.0 Height=1800.0`), bases 3, cities 6, PlayerController, view target, crosshair, HUD, `[DD][Area] Viewport=...`.
- [ ] **B2** Exactly one of `[DD][Save] Loaded high=<n>` or `[DD][Save] No file high=<n>`, followed by exactly one `[DD][Flow] Title high=<n>`.
- [ ] **B3** During the 20 s wait: no `Spawner started`, no `[DD][Wave]`, `[DD][Enemy]`, `[DD][Flyer]`, `[DD][Score]`, `[DD][Damage]`, `[DD][Explosion]`, `[DD][Input] Fire request`, `[DD][Aim]`, `[DD][Game]`, and no `[DD][Flow]` line except the one `Title` line.
- [ ] **B4** No line containing `ERROR`, `Accessed None`, `Divide by zero` or `Blueprint Runtime Error`.

### D. Interactive checks (human, PIE) — all NEEDS-HUMAN
List every item as NEEDS-HUMAN in the report; do not run or claim them.
- [ ] **D1** **Your blasts:** an ABM's explosion is look A in cyan-white; a chain reaction started by your blast is cyan-white too (also one started by killing a flyer).
- [ ] **D2** **Enemy blasts:** an enemy missile's impact on a city or base is look A in orange; a chain reaction started by an enemy impact is orange.
- [ ] **D3** **Size and timing:** every explosion grows for 0.4 s, holds 0.4 s and fades over 0.5 s, reaches the same size as before (radius 160), and nothing is drawn outside that circle; the play log still shows `max=160`, `peak=160` and `life` about 1300 ms.
- [ ] **D4** **No gameplay change:** kills, chain reactions, scoring, damage, waves, flyers, pause and game over behave exactly as before; the old orange disc is gone and does not show behind the new look.
- [ ] **D5** The keys `0` to `7` do nothing any more; the other debug keys still work.
- [ ] **D6** A busy late wave with many explosions at once runs without a hitch.
- [ ] **D7** No errors in the play log (`Blueprint Runtime Error`).

### D-DEVICE. Android (NEEDS-DEVICE)
- [ ] **T-1** The frame rate in a busy wave is not noticeably lower than before; the cyan-white and orange blasts are easy to tell apart on the handheld screen.

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped; the editor is on `L_Game`; nothing under `Spike` was touched.

---

## 8. Report format (the agent's final message)

```
## Phase 9d.2 report (look A in the real explosions, cleanup)
**Engine version:** x.y.z
**MCP tools used:** (list)

### Step 0 findings
(peak variable; SetRadius content; where Age is increased; references to Disc; writers of bPlayerMade; referencers of each asset to delete)

### BP_Explosion (Step 1)
(components and flags; variables; the two functions as read back; SetRadius before and after)

### BP_DDPlayerController (Step 2)
(what was removed; the remaining key events and functions)

### Deleted assets (Step 3)
(each asset with its referencers before deletion; the final folder listings)

### Pin audit (rule 13)
(table: function | node | pin | wired from / value)

### Hands-off audit (rule 3)
(table: item | unchanged? | how it was checked)

### Acceptance results
A1 PASS/FAIL ... through E1 PASS/FAIL/NEEDS-HUMAN/NEEDS-DEVICE

### Log excerpt
(all [DD] lines of the run, with timestamps)

### Tool problems (errors, retries, editor restarts, writes that read back empty)
### Deviations
### Open questions / risks
```

---

## 9. What this phase hands to later phases

| Provided | How a later phase uses it |
|---|---|
| Look A with two palettes inside `BP_Explosion` | The colours are the six `Colors` variables; brightness and softness are the parameters of `MI_DD_ExpDisc` and `MI_DD_ExpGlow`. Both can be tuned by hand. |

**Known leftovers that stay (removed in 9g):** the hidden `Disc` of `BP_Explosion` and `MI_DD_Explosion`, the hidden cubes of `BP_Flyer`, the hidden `Dot` of `BP_TapMarker` and of the missiles, the old `UpdateSmartTail` (uncalled), `StopSpawner`, `bDirectorSilent`, the two disconnected spawn chains in `SpawnRandomEnemy`, the disconnected older fire chain in `LaunchAt`, HUD debug lines 2 to 4, `BP_PlayfieldDebug`, the debug keys, the unused `MI_DD_Enemy` and `MI_DD_Abm`, the `Spike` folder.

---

## 10. Common pitfalls

1. **Taking screenshots or placing temporary actors.** Not allowed.
2. **Changing the radius, timing or kill logic.** The new layers only follow `NewRadius` in `SetRadius`.
3. **Using the wrong peak variable** in `UpdateLook` (`MaxRadius` versus `PeakRadius`): the fade would not end at zero. Use the one `RadiusAtAge` uses.
4. **Forgetting to hide `Disc`:** an opaque orange disc would sit behind the new look.
5. **Rewriting `SetRadius`, `Init` or `Advance`** with the graph-writing tool (rule 11). Only the one inserted call.
6. **Deleting an asset that is still referenced** (rule 18). Remove the callers in the controller first, compile, then delete `BP_ExplosionPreview`, then the materials.
7. **Deleting `M_DD_ExpDisc`, `M_DD_ExpGlow`, their instances or `MI_DD_Explosion`.** They stay.
8. **A pin left at its default** in the new nodes (rule 13): `Select` conditions, scalar and vector parameter names, the `NewRadius` wiring of the inserted call.
9. **Marking D items PASS.** You cannot see an explosion in PIE. They are NEEDS-HUMAN and NEEDS-DEVICE.
10. **Touching `Spike` or any other frozen asset**, or cleaning up what this document does not list.
