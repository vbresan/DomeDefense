# Phase 9c.2 (patch) — Stretch the Missile Head and Glow Along the Direction of Flight

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Depends on** | Phase 9c.1.2 (built; the repository `master` at commit "Patching the patched patch"). The project is exactly at that state. An earlier attempt to give missiles a bullet shape was reverted: **there are no bullet components of any kind in the project** (no `BulletBody`, `BulletNose`, `SteerBullet`, `ArmBullet`, `SmartBombTurnRate`). |
| **Phase goal** | **Visual only. No gameplay change.** The round head (`Dot`) and the round glow (`Halo`) of every ordinary missile are **stretched by 1.25 along the direction of flight**, so they become slightly oval, with the long axis along the missile's path (and along its tail). This applies to `BP_EnemyMissile` (enemy missiles, MIRV warheads, missiles dropped by flyers) and `BP_ABM`. **The smart bomb stays round** (unchanged). Nothing is added: two parts per Blueprint are moved under `TailPivot` and get new scale values. |

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **No gameplay change.** The game uses one point per missile, the actor's location (the explosion test `IsInsideExplosion` calls `ContainsPoint(GetActorLocation)`; flight, impact, smart-bomb sensing and dodging use the same position). Every visual component has no collision. Positions, speeds, targets, kills, impacts, scoring, waves, flyers, MIRV splits, smart-bomb behaviour and game flow are unchanged.
3. **HANDS OFF.** Do not change, and report as unchanged after the work:
   - `BP_EnemyMissile`: the components `Root`, `DotSmart`, `HaloSmart`, `TailPivot`, `Tail`, `TailSmart`; **`Dot` and `Halo` except their parent, relative location and relative scale** (their mesh, material, rotation and flags stay); all variables (`StartPos`, `Target`, `TrailLen`, `TrailWidth`, `TailHideHold`, `TailHoldLeft`, `bSmart`, `bDodging`, `bActive`, and the others); every function and graph (`AimTrail`, `UpdateTrail`, `UpdateSmartTail`, `UpdateSmartTail2`, `Advance`, `ApplyDodge`, `ComputePush`, `Die`, `Impact`, `Split`, `CheckSplit`, `Init`, `IsInsideExplosion`, `ArmSmart`, `Event Tick`, and the rest).
   - `BP_ABM`: the components `Root`, `TailPivot`, `Tail`; **`Dot` and `Halo` except their parent, relative location and relative scale**; all variables and all graphs.
   - **Every material and material instance.**
4. **No graph is changed in this patch.** Do not open a graph for editing. Reading graphs (Step 0) is allowed.
5. **Compile after every Blueprint change.** Every Blueprint ends with no errors and no warnings.
6. **Run each step's "Check"** before moving on. **Save All after each step's Check passes.**
7. **You cannot press keys, click, touch, move the mouse, or resize in PIE, and you cannot start a game.** No missile exists on the title screen, so **you cannot see a stretched missile in PIE.** You can verify: compiles, static read-backs of every component, and (Step 3) a capture of temporary actors in the **editor** world. Everything seen in play is group D (**NEEDS-HUMAN**); device checks are **NEEDS-DEVICE**. Never mark a D or T item PASS.
8. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
9. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after the run. **Save before loading a level.** **Do not touch Project Settings, input settings or any `Config/*.ini` file.** **Do not use, edit or delete anything in `/Game/DomeDefense/Spike/`.**
10. **VISIBILITY AUDIT.** `Dot` and `Halo` must keep `bVisible` TRUE and `bHiddenInGame` FALSE, `Mobility` Movable and their collision setting. Read all of them back after the changes and list them in the report (A6). (`DotSmart`, `HaloSmart` and `TailSmart` keep `bVisible` false as in 9c.)
11. **After any tool error, do not retry the same operation more than once.** If it still fails, stop that item and report. **If the editor stops responding or restarts, stop and report.**
12. **Order is part of the spec:** read first, then `BP_EnemyMissile`, then `BP_ABM`, then the calibration, then the run.
13. **FROZEN: do not edit** every asset except `BP_EnemyMissile` and `BP_ABM`. In particular frozen: `BP_DDGameMode`, `BP_DDPlayerController`, `BP_DDHUD`, `BP_Flyer`, `BP_Explosion`, `BP_MissileBase`, `PDA_DDConfig`, `DA_DDConfig`, `L_Game` (except the fully reverted proof of Step 3), and every material and material instance.
14. **No new keys, no new log lines, no new config values, no new variables, no new functions, no new components, no new assets.** The only edits are the two re-parents and the scale and location values of `Dot` and `Halo` in the two Blueprints.
15. **Do not clean up anything this document does not list.**
16. **If the project is not in the state described in "Depends on"** (an unexpected component in either Blueprint, a `Dot` or `Halo` that is not a child of `Root`, a `TailPivot` with a non-zero rotation, a scale other than 1, or a location X or Z other than 0), **stop and report** before changing anything.

---

## 2. Scope

### In scope
- `BP_EnemyMissile` and `BP_ABM`: re-parent `Dot` and `Halo` from `Root` to `TailPivot`, and set their relative location and relative scale as in section 3.

### Out of scope (do NOT do)
- The smart bomb's look (`DotSmart`, `HaloSmart`, `TailSmart` stay as they are).
- Any change to a tail, a colour, a material, a mesh, a rotation, or a config value.
- Any change to how any missile moves, is aimed, senses or dodges.
- Any bullet shape.

---

## 3. Design (decided; do not change)

### 3.1 Why the parts move under `TailPivot`
`Root` never rotates, so a stretched `Dot` under `Root` would be stretched along the screen's horizontal axis whatever way the missile flies. `TailPivot` is the component that is rotated to the flight direction: `AimTrail` (called once in `Init`) sets its Pitch to `Atan2Degrees(d.Z, d.X)` with `d = Target - StartPos`, so **the pivot's local +X axis is the direction of flight**. Its location is `(0, 68.5, 0)` in `BP_EnemyMissile` and `(0, 78.5, 0)` in `BP_ABM`, with no rotation and scale 1 by default, so in X and Z the pivot sits exactly on the actor's location (the missile's point). Children of the pivot turn with it. Stretching a child along its local X therefore stretches it along the flight line.

### 3.2 The values
The stretch factor is **1.25**. Nothing else about the size changes (width across the flight line stays 22.5 for the head and 55 for the glow).

| Component | Mesh | Before (parent `Root`) | After (parent `TailPivot`) |
|---|---|---|---|
| `Dot` (`BP_EnemyMissile`) | Sphere | location (0, 70, 0), rotation 0, scale (0.225, 0.225, 0.225) | **location (0, 1.5, 0)**, rotation (0, 0, 0), **scale (0.28125, 0.225, 0.225)** |
| `Halo` (`BP_EnemyMissile`) | Plane | location (0, 69.3, 0), rotation Roll 90, scale (0.55, 0.55, 0.55) | **location (0, 0.8, 0)**, rotation Roll 90 (unchanged), **scale (0.6875, 0.55, 0.55)** |
| `Dot` (`BP_ABM`) | Sphere | location (0, 80, 0), rotation 0, scale (0.225, 0.225, 0.225) | **location (0, 1.5, 0)**, rotation (0, 0, 0), **scale (0.28125, 0.225, 0.225)** |
| `Halo` (`BP_ABM`) | Plane | location (0, 79.3, 0), rotation Roll 90, scale (0.55, 0.55, 0.55) | **location (0, 0.8, 0)**, rotation Roll 90 (unchanged), **scale (0.6875, 0.55, 0.55)** |

Arithmetic to put in the report: 0.225 x 1.25 = 0.28125 and 0.55 x 1.25 = 0.6875. Visible sizes: head 28.125 x 22.5, glow 68.75 x 55. World layers are unchanged: 68.5 + 1.5 = 70.0 and 68.5 + 0.8 = 69.3 (enemy); 78.5 + 1.5 = 80.0 and 78.5 + 0.8 = 79.3 (ABM). X and Z of both parts stay 0, so the oval is centred on the missile's point; its front edge is 14.0625 ahead of the point instead of 11.25 (2.8125 further), and kills and impacts still use the exact point.

Why the X scale: the sphere has no rotation, so its local X is the pivot's X. The plane's Roll 90 is a rotation about the X axis, so the plane's local X also stays on the pivot's X; the plane's local Y becomes the in-screen axis across the flight line. So the factor goes on **X** for both parts.

### 3.3 The smart bomb
Unchanged. `ArmSmart` hides `Dot`, `Halo` and `Tail` and shows `DotSmart`, `HaloSmart` and `TailSmart`; it refers to the components by name, so it still works after the re-parent. `DotSmart` and `HaloSmart` stay children of `Root`, round, as before.

---

## 4. What this phase changes in earlier specs

| Earlier text | Now |
|---|---|
| 9c: `Dot` and `Halo` are children of `Root`, round | Children of `TailPivot`, stretched by 1.25 along the flight direction |
| 9c: `Dot` location Y 70 / 80 and `Halo` Y 69.3 / 79.3 | The same world layers, written as pivot-relative 1.5 and 0.8 |
| 9c frozen list | `BP_EnemyMissile` and `BP_ABM` are unfrozen for the edits of this document only. |

---

## 5. Tooling notes

1. Re-parent with `ActorTools.set_parent_component`. **After each re-parent read back the parent and the relative location, rotation and scale** (the tool may keep either the world transform or the relative transform). Then set location and scale explicitly to the values of 3.2 and read them back again.
2. Edit existing component properties with the object-property tools and read back.
3. `CaptureViewport` needs a camera transform and shows the **editor** world (use `(0, 3000, 900)` as in 9b and 9c). **PIE cannot spawn actors**; nothing flies on the title screen.
4. If a tool reports that a property cannot be read or set, say so; do not guess.

---

## 6. Implementation steps

### Step 0 — Read (no changes)
1. **`BP_EnemyMissile`:** every component with parent, relative transform, mesh, material, `bVisible`, `bHiddenInGame`, `Mobility` and collision flag. Expected: `Root`; under it `Dot`, `DotSmart`, `Halo`, `HaloSmart`, `TailPivot`; under `TailPivot` `Tail`, `TailSmart`. **Anything else: stop and report (rule 16).**
2. `TailPivot` (both Blueprints): relative location, rotation and scale. Expected: `(0, 68.5, 0)` / `(0, 78.5, 0)`, rotation 0, scale 1. **Anything else: stop and report.**
3. `Dot` and `Halo` (both Blueprints): mesh path (expected `/Engine/BasicShapes/Sphere` for `Dot`, `/Engine/BasicShapes/Plane` for `Halo`), parent (expected `Root`), the transforms of the "Before" column of 3.2. **A difference: stop and report.**
4. `AimTrail` and `Init` (`BP_EnemyMissile`): confirm `AimTrail` sets the Pitch of `TailPivot` and is called from `Init` (read only). In `BP_ABM`: the same for its aiming. If a Blueprint does not rotate `TailPivot` at all, stop and report.
5. List **every node in every graph that references `Dot` or `Halo`** (both Blueprints). Expected in `BP_EnemyMissile`: only the `SetVisibility` calls in `ArmSmart`. If anything else uses them (for example a location read), record it; do not change it.
6. `ArmSmart`: list its calls and pins (read only).

**Check:** the report section for Step 0 is complete.

### Step 1 — `BP_EnemyMissile`
Read the Blueprint again first.
1. Re-parent `Dot` under `TailPivot`. Read parent and relative transform back. Set location `(0, 1.5, 0)`, rotation `(0, 0, 0)`, scale `(0.28125, 0.225, 0.225)`. Read back.
2. Re-parent `Halo` under `TailPivot`. Read back. Set location `(0, 0.8, 0)`, keep rotation Roll 90, scale `(0.6875, 0.55, 0.55)`. Read back.
3. Read the whole component tree back: `Root` has `DotSmart`, `HaloSmart`, `TailPivot`; `TailPivot` has `Dot`, `Halo`, `Tail`, `TailSmart`. Mesh, material and flags of every component unchanged.
4. Compile.

**Check:** compiles clean; `Dot` and `Halo` read back exactly as in 3.2; every item of rule 3 reads exactly as before; tick settings unchanged. **Save All.**

### Step 2 — `BP_ABM`
Read the Blueprint again first. Do the same as Step 1 with the `BP_ABM` values of 3.2 (they are the same pivot-relative numbers). The component tree after the change: `Root` has `TailPivot`; `TailPivot` has `Dot`, `Halo`, `Tail`.

**Check:** compiles clean; `Dot` and `Halo` read back exactly as in 3.2; everything else as before. **Save All.**

### Step 3 — Calibration of direction (recommended; a few calls)
Goal: see that the oval lies along the pivot's X axis, which is the flight direction.
1. List the actors in `L_Game` (record the count). Place two temporary actors: `TestRed` (`BP_EnemyMissile`) at `(-300, 0, 900)` and `TestAbm` (`BP_ABM`) at `(300, 0, 900)`.
2. **Capture A** (`CaptureViewport` from `(0, 3000, 900)`) with both pivots at pitch 0: the head and the glow must be **wider than tall** (about 28 x 22.5 and 69 x 55). Describe each and estimate the width-to-height ratio (expected about 1.25).
3. **Capture B:** set `TailPivot` on both instances (the instance only) to (Roll 0, Pitch **-90**, Yaw 0): the head and the glow must now be **taller than wide**, the long axis vertical.
4. **Capture C:** set `TailPivot` on `TestRed` to Pitch **150**: the long axis must run along the tilted direction (30 degrees from horizontal, up-left to down-right).
5. **Fix rule:** if the head is stretched correctly but the **glow** is stretched across the flight line in capture A (taller than wide), put the 1.25 on the glow's **Y** scale instead: `Halo` scale `(0.55, 0.6875, 0.55)` in both Blueprints (Blueprint defaults, not the instances), recompile, and repeat the captures. If the head is wrong, stop and report; do not invent another change.
6. **Revert:** delete both temporary actors, save the level, list the actors again: the count and the list **must equal** the one recorded in step 1 and `L_Game` must not be dirty. If anything cannot be reverted, **stop and report**.

**Fallback:** if the actors cannot be placed or the capture is unclear, skip the step, leave the values as built and record why; the human checks the direction in play (D1 and D2).

**Check:** the direction is recorded (and the fix applied if needed); the level is exactly as before. **Save All.**

### Step 4 — Run
1. **Save All.**
2. `StartPIE`. The title screen appears at once. **Wait about 20 seconds**; nobody taps, so no game starts and no missile exists.
3. `CaptureEditorImage` at about 5 s and again at about 15 s. Describe each image (the title screen as in 9c; nothing else changed).
4. `GetLogEntries` (filter `[DD]`), then `StopPIE`.
5. Fill in the report (section 8).

---

## 7. Acceptance tests

### A. Static checks
- [ ] **A1** Changed assets: `BP_EnemyMissile` and `BP_ABM` only. **No new asset of any kind**, no material or instance touched, nothing under `Spike`, no Project Settings, input settings or `.ini` change. `L_Game` unchanged.
- [ ] **A2** `BP_EnemyMissile` and `BP_ABM` compile with 0 errors and 0 warnings.
- [ ] **A3** Step 0 findings are in the report: component lists, `TailPivot` values, mesh paths, all references to `Dot` and `Halo`, `ArmSmart` calls.
- [ ] **A4** In both Blueprints `Dot` and `Halo` are children of `TailPivot` with exactly the values of 3.2 (or the one fix of Step 3.5, recorded); mesh, material and rotation unchanged.
- [ ] **A5** The arithmetic of 3.2 is in the report (scales, visible sizes, world layers, front edge).
- [ ] **A6** **Visibility audit:** a table of `Dot` and `Halo` in both Blueprints with `bVisible`, `bHiddenInGame`, `Mobility` and the collision flag, before and after: `bVisible` true, `bHiddenInGame` false, Movable, collision unchanged.
- [ ] **A7** **Unchanged (rule 3):** a table that shows, for each item of rule 3, that it reads exactly as before this patch, including the component tree of both Blueprints and the six calls of `ArmSmart`.
- [ ] **A8** Calibration (Step 3): the direction in captures A, B and C and the fix if one was applied; or the recorded fallback; if the step ran, `L_Game` has the same actors as before.

### B. Log checks (agent; the one 20 s run, nobody taps)
- [ ] **B1** The startup lines as in the 9c.1.2 run: camera, GameMode, config (`Width=2250.0 Height=1800.0`), bases 3, cities 6, PlayerController, view target, crosshair, HUD, `[DD][Area] Viewport=...`.
- [ ] **B2** Exactly one of `[DD][Save] Loaded high=<n>` or `[DD][Save] No file high=<n>`, followed by exactly one `[DD][Flow] Title high=<n>`.
- [ ] **B3** During the 20 s wait: no `Spawner started`, no `[DD][Wave]`, `[DD][Enemy]`, `[DD][Flyer]`, `[DD][Score]`, `[DD][Damage]`, `[DD][Input] Fire request`, `[DD][Aim]`, `[DD][Game]`, and no `[DD][Flow]` line except the one `Title` line.
- [ ] **B4** No line containing `ERROR`, `Accessed None`, `Divide by zero` or `Blueprint Runtime Error`.

### C. Visual checks (captures; describe in words)
- [ ] **C1** The title screen is unchanged from 9c. No new object appears (no missile exists).
- [ ] **C2** Step 3 captures (if run): the head and the glow are oval with the long axis along the pivot's X axis, as described in A8.

### D. Interactive checks (human, PIE) — all NEEDS-HUMAN
List every item as NEEDS-HUMAN in the report; do not run or claim them.
- [ ] **D1** **Straight down:** an enemy missile falling straight down has its head and glow stretched **vertically**, along its path and along its tail.
- [ ] **D2** **Diagonals:** missiles coming from the left and from the right are stretched along their paths; the oval always lines up with the tail.
- [ ] **D3** **ABMs:** from launch to the explosion the ABM's head and glow are stretched along its path toward the target.
- [ ] **D4** **Warheads and flyer drops:** after a MIRV split each warhead's oval lines up with its own tail; missiles dropped by bombers and satellites are stretched the same way.
- [ ] **D5** **Smart bomb:** it is the round magenta ball with its round glow and tail at every moment, including while it dodges.
- [ ] **D6** **Look and layering:** the head is slightly oval (about 28 x 22.5), the glow about 69 x 55; the head is drawn in front of its own glow and tail; enemy missiles are behind ABMs; explosions draw over missiles.
- [ ] **D7** **No gameplay change and no errors:** kills, impacts, scoring, waves, flyers, pause, quit and smart-bomb dodging all behave as before; the play log has no `Blueprint Runtime Error`.
- [ ] **D8** A busy late wave runs without a hitch.

### D-DEVICE. Android (NEEDS-DEVICE)
- [ ] **T-1** On a 1080-pixel-high screen the head is about 17 x 13.5 pixels and the glow about 41 x 33; the direction of flight is readable.
- [ ] **T-2** The frame rate in a busy wave is not noticeably lower than before.

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped; the editor is on `L_Game`; nothing under `Spike` was touched; if Step 3 ran, the level is as before.

---

## 8. Report format (the agent's final message)

```
## Phase 9c.2 report (stretch heads along flight)
**Engine version:** x.y.z
**MCP tools used:** (list)

### Step 0 findings
(components with flags in both Blueprints; TailPivot values; mesh paths of Dot and Halo; all references to Dot and Halo; ArmSmart calls; how AimTrail rotates TailPivot)

### Component changes (Steps 1 and 2)
(table: Blueprint | component | parent | location | rotation | scale | bVisible | bHiddenInGame | Mobility | collision flag; Dot and Halo before and after)

### Arithmetic (3.2)
(scales, visible sizes, world layers, front edge)

### Hands-off audit (rule 3)
(table: item | unchanged? | how it was checked)

### Calibration (Step 3)
(what captures A, B and C showed; the fix if any; level actor count before and after; or why it was skipped)

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
| `Dot` and `Halo` under `TailPivot` with the stretch on the X scale | The stretch is a single number per part: X scale = base x factor (head base 0.225, glow base 0.55; factor now 1.25). A different factor is an edit of the two X scales per Blueprint (for 1.5: head 0.3375, glow 0.825). |

**Known leftovers that stay:** the old `UpdateSmartTail` (uncalled), `StopSpawner`, `bDirectorSilent`, the two disconnected spawn chains in `SpawnRandomEnemy`, the disconnected older fire chain in `LaunchAt`, HUD debug lines 2 to 4, `BP_PlayfieldDebug`, the debug keys, the unused `MI_DD_Enemy` and `MI_DD_Abm`, the `Spike` folder (removed in 9g).

---

## 10. Common pitfalls

1. **Scaling `Dot` or `Halo` while they are still children of `Root`.** They would stretch along the screen's horizontal axis, not along the flight line. The re-parent is the point of the patch.
2. **`set_parent_component` changing the relative transform.** Read parent and transform back after each re-parent and set the values explicitly.
3. **Putting the factor on the wrong axis.** It goes on the local **X** of both parts. The sphere has no rotation; the plane's Roll 90 keeps its X on the pivot's X. If capture A shows the glow taller than wide, apply the fix of Step 3.5.
4. **Wrong layers.** Y 1.5 and 0.8 are pivot-relative. Leaving the old 70 / 80 and 69.3 / 79.3 on a child of `TailPivot` would push the parts to about world Y 138 (enemy) and 158 (ABM), in front of the explosions.
5. **Touching the smart bomb** (`DotSmart`, `HaloSmart`, `TailSmart`), a tail, a material, or any graph.
6. **Changing `bHiddenInGame` or `bVisible`** of `Dot` or `Halo`.
7. **Leaving the Step 3 actors in the level.** The actor count must match the count before.
8. **Marking D or T items PASS.** You cannot see a missile in PIE. They are NEEDS-HUMAN and NEEDS-DEVICE.
9. **Touching `Spike`, `BP_DDGameMode` or any other frozen asset**, or cleaning up what this document does not list.
