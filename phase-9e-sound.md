# Phase 9e — Sound

| | |
|---|---|
| **Project** | Dome Defense clone (classic 1980 rules) |
| **Engine** | Unreal Engine 5.8.4 (do not upgrade) |
| **Language** | Blueprint only. No C++. |
| **Executor** | AI agent working through MCP tools |
| **Depends on** | Phase 9d.2 (built; it may or may not be committed, the live project is what counts). `BP_Explosion` contains `UpdateLook` and `ApplyPalette`. The human has imported the sound files into `/Game/DomeDefense/Audio/`. |
| **Phase goal** | **Sound only. No gameplay change and no visual change.** Seven sounds are played by the game: ABM launch, explosion, enemy destroyed, flyer hum, wave-start siren, zero-ammo alarm and game over. Everything is 2D (no panning, no distance). Each sound has its own volume number in `BP_DDGameMode` that the human can edit. Busy moments are limited by voice caps set on the sound assets. Every sound event writes one `[DD][Sfx]` log line so the human and the reviewer can count them against the existing log lines. |

---

## 1. Rules of engagement (read first)

1. **Do exactly what this document says. Nothing more.** Names are exact and case-sensitive.
2. **No gameplay change.** Kill tests, radii, timing, scoring, damage, waves, flyer behaviour, pause, game over and every existing log line stay exactly as they are. The only additions are the nodes listed in this document.
3. **HANDS OFF.** Do not change, and report as unchanged after the work: `BP_DDHUD`, `BP_ABM`, `BP_TapMarker`, `BP_City`, `BP_Crosshair`, `BP_Backdrop`, `BP_PlayfieldCamera`, `BP_PlayfieldDebug`, `BFL_DDDebug`, `PDA_DDConfig`, `DA_DDConfig`, `L_Game`, every material and material instance, and the sound asset `dd_tally_tick` (it stays unused). In the six Blueprints that are edited (section 2), change nothing except the additions listed in sections 3 and 6.
4. **NO SCREENSHOTS.** Do not use `CaptureViewport` or `CaptureEditorImage`, and do not place temporary actors in the level. Verify only by compiling and by reading assets back.
5. **Existing logic must survive.** Read each graph (`read_graph_dsl`) before editing it.
6. **Compile after every Blueprint change.** Every Blueprint ends with no errors and no warnings.
7. **Run each step's "Check"** before moving on. **Save All after each step's Check passes.**
8. **You cannot press keys, click, touch, hear, or start a game in PIE.** No sound is played on the title screen, so nothing in this phase can be heard or triggered by you. Everything heard or seen in play is group D (**NEEDS-HUMAN**). Never mark a D item PASS.
9. **If something cannot be done**, use the named Fallback if there is one; otherwise stop that item and record it under *Deviations*. Do not invent alternatives.
10. **Only touch** `/Game/DomeDefense/`. **Do not commit.** **Stop PIE** after the run. **Save before loading a level.** **Do not touch Project Settings, input settings, audio settings or any `Config/*.ini` file.** **Do not use, edit or delete anything in `/Game/DomeDefense/Spike/`.**
11. **NEVER REWRITE AN EXISTING FUNCTION OR EVENT GRAPH** with `write_graph_dsl`. New functions (sections 3.2 to 3.4) are written once with `write_graph_dsl` and read back. Every hook into an existing graph (3.5) is added with `create_node` and `connect_pins` at the place stated; show the graph before and after.
12. **READ EVERY NEW OR CHANGED GRAPH BACK** right after writing it. If it reads back wrong or incomplete: correct it once more; if it is still wrong, **stop and report** under *Tool problems*.
13. **CHECK EVERY PIN YOU BUILD** (the lesson of the Phase 7 pause defect: a pin left at its default). After building the new functions and the hooks, read all input pins of every new node back and list them (A8). A `PlaySound2D` whose `Is UISound` pin is left at its default **true** is a defect (3.1).
14. **After any tool error, do not retry the same large operation.** Stop that item, try one smaller piece at most, and if it still fails **stop and report**. **If the editor stops responding or restarts, stop and report.**
15. **Order is part of the spec:** read first, then the sound assets, then `BP_DDGameMode`, then the hooks, then the run.
16. **No new keys, no new config assets, no new input actions, no new materials, no new actors.** The only new assets are the optional Sound Concurrency assets of 3.6 Fallback B.
17. **If the project is not in the state described in "Depends on"** (an asset of 3.7 is missing, `UpdateLook` is missing in `BP_Explosion`, or a name of 3.2 to 3.4 already exists), **stop and report** before changing anything.

---

## 2. Scope

### In scope
- Sound assets: looping flag on the hum, voice caps on three sounds (3.6).
- `BP_DDGameMode`: sound variables, helper functions, three hooks (wave start, game over, new game).
- `BP_DDPlayerController`: one call in `LaunchAt`.
- `BP_Explosion`: one variable, one function, one call at the end of `UpdateLook`.
- `BP_EnemyMissile` and `BP_Flyer`: one call in each `Die` (rule 3.5 row 3).
- `BP_Flyer`: one variable, the hum start in `Init`, the hum fade in `ReceiveEndPlay`.
- `BP_MissileBase`: one call in `ConsumeAmmo`.

### Out of scope (do NOT do)
- The wave-complete tally (`dd_tally_tick` is not used in this phase).
- Music, sound for the title screen, menus, pause, the HUD, the target marker, enemy missile launches, MIRV splits, city rebuild, bonus city. Only the seven events of this document make sound.
- Positional audio, attenuation assets, sound classes, sound mixes, a mute switch, a settings screen.
- Any change to the sound files themselves.
- Anything else in the game.

---

## 3. Design (decided; do not change)

### 3.1 How sounds are played
- **One-shots** use `PlaySound2D` (GameplayStatics), always with **`Is UISound` = FALSE** so that they freeze when the game is paused. `Concurrency Settings` and `Owning Actor` stay empty; `Start Time` 0. All one-shots go through the single function `PlaySfx` of 3.2.
- **The hum** is a looping component made with `Create Sound 2D` (3.3 `CreateHum`), also with `UI Sound` set to false, so it pauses with the game.
- Every sound reaches the game through variables of `BP_DDGameMode` (3.2), so the packaged game cooks the files because they are referenced.

### 3.2 `BP_DDGameMode`: variables and helper functions

**Variables** (Instance Editable; Category as stated):

| Variable | Type | Default | Category |
|---|---|---|---|
| `SfxLaunchSound` | Sound Base (object reference) | `/Game/DomeDefense/Audio/dd_launch` | `Sound Assets` |
| `SfxExplosionSound` | Sound Base | `/Game/DomeDefense/Audio/dd_explosion` | `Sound Assets` |
| `SfxHitSound` | Sound Base | `/Game/DomeDefense/Audio/dd_enemy_hit` | `Sound Assets` |
| `SfxHumSound` | Sound Base | `/Game/DomeDefense/Audio/dd_flyer_hum_loop` | `Sound Assets` |
| `SfxSirenSound` | Sound Base | `/Game/DomeDefense/Audio/dd_siren` | `Sound Assets` |
| `SfxAlarmSound` | Sound Base | `/Game/DomeDefense/Audio/dd_alarm` | `Sound Assets` |
| `SfxGameOverSound` | Sound Base | `/Game/DomeDefense/Audio/dd_game_over` | `Sound Assets` |
| `VolLaunch` | float | 0.9 | `Sound` |
| `VolExplosion` | float | 1.0 | `Sound` |
| `VolEnemyHit` | float | 0.9 | `Sound` |
| `VolHum` | float | 0.35 | `Sound` |
| `VolSiren` | float | 0.5 | `Sound` |
| `VolAlarm` | float | 0.4 | `Sound` |
| `VolGameOver` | float | 0.6 | `Sound` |
| `SfxPitchJitter` | float | 0.06 | `Sound` |
| `EnemyBlastPitchScale` | float | 0.8 | `Sound` |
| `HumPitchBomber` | float | 0.8 | `Sound` |
| `HumPitchSatellite` | float | 1.3 | `Sound` |
| `AlarmSpacing` | float | 0.9 | `Sound` |
| `AlarmQueue` | integer | 0 | `Sound` (not Instance Editable) |
| `bAlarmBusy` | boolean | false | `Sound` (not Instance Editable) |

**Functions** (all in `BP_DDGameMode`; write each once, read back):

```
fn JitterPitch() -> float            // pure
  return RandomFloatInRange(1.0 - SfxPitchJitter, 1.0 + SfxPitchJitter)

fn PlaySfx(Sound: SoundBase, Volume: float, Pitch: float)
  if IsValid(Sound):
     PlaySound2D(Sound, Volume, Pitch, StartTime=0.0, ConcurrencySettings=none, OwningActor=none, bIsUISound=FALSE)

fn SfxLaunch()
  Log "[DD][Sfx] Launch"
  PlaySfx(SfxLaunchSound, VolLaunch, JitterPitch())

fn SfxBlast(bPlayerBlast: bool)
  Log "[DD][Sfx] Blast player=" + (bPlayerBlast ? "1" : "0")
  PlaySfx(SfxExplosionSound, VolExplosion, Select(bPlayerBlast, 1.0, EnemyBlastPitchScale) * JitterPitch())

fn SfxHit()
  Log "[DD][Sfx] Hit"
  PlaySfx(SfxHitSound, VolEnemyHit, JitterPitch())

fn SfxSiren()
  Log "[DD][Sfx] Siren"
  PlaySfx(SfxSirenSound, VolSiren, 1.0)

fn SfxGameOver()
  Log "[DD][Sfx] GameOver"
  PlaySfx(SfxGameOverSound, VolGameOver, 1.0)
```
Logging uses the same function and the same way as the existing `[DD]` lines (Step 0 tells which: the `DDLog` of `BFL_DDDebug`). The text of every message is exactly as written above, after the category the existing lines already use for `[DD][...]`.

### 3.3 The hum

**Function `CreateHum(bSatellite: bool) -> AudioComponent`** in `BP_DDGameMode`:
```
Pitch = Select(bSatellite, HumPitchSatellite, HumPitchBomber)
Comp  = CreateSound2D(SfxHumSound, VolHum, Pitch, StartTime=0.0, ConcurrencySettings=none,
                      bPersistAcrossLevelTransition=false, bAutoDestroy=true)
Comp.SetUISound(false)                      // Fallback in 3.6 if this node does not exist
Comp.FadeIn(FadeInDuration=0.15, FadeVolumeLevel=1.0, StartTime=0.0, FadeCurve=Linear)
Log "[DD][Sfx] Hum on sat=" + (bSatellite ? "1" : "0") + " pitch=" + Pitch
return Comp
```
`CreateSound2D` makes a component that does not play by itself and that does **not** belong to the flyer; that is why the flyer must stop it (3.5 row 4).

### 3.4 The zero-ammo alarm queue

Alarms never overlap. Each base that fires its last ABM adds one alarm to a queue; the queue plays them one after another with `AlarmSpacing` seconds between starts.

```
fn QueueAlarm()
  AlarmQueue = AlarmQueue + 1
  Log "[DD][Sfx] Alarm queued q=" + AlarmQueue
  if NOT bAlarmBusy:  PlayNextAlarm()

fn PlayNextAlarm()
  if AlarmQueue > 0:
     AlarmQueue = AlarmQueue - 1
     bAlarmBusy = true
     Log "[DD][Sfx] Alarm play q=" + AlarmQueue
     PlaySfx(SfxAlarmSound, VolAlarm, 1.0)
     SetTimerByFunctionName(self, "PlayNextAlarm", AlarmSpacing, bLooping=false)
  else:
     bAlarmBusy = false

fn ResetAlarms()
  AlarmQueue = 0            // bAlarmBusy is left alone: the pending timer ends it by itself
```
`SetTimerByFunctionName` is used (not a custom event) because it works inside a function. The name `"PlayNextAlarm"` must match exactly.

### 3.5 Hooks (the places where the sounds are triggered)

| # | Blueprint / function | Hook | Sound |
|---|---|---|---|
| 1 | `BP_DDPlayerController.LaunchAt` | On the **live success chain** (the chain that is connected and runs when an ABM is launched; the file has an older disconnected fire chain that must not be used), immediately after the node that finishes spawning the ABM, call `SfxLaunch()` on the game mode. The failure path (no ammo, `bLaunched` false) gets nothing. | Launch |
| 2 | `BP_Explosion` | New variable `bSfxDone` (boolean, default false, not Instance Editable). New function `TryBlastSfx()`: `if (NOT bSfxDone) AND (Age > 0.0): bSfxDone = true; game mode.SfxBlast(bPlayerMade)`. Add a call to `TryBlastSfx()` **after the last node of `UpdateLook`'s execution chain** (after `ApplyPalette()`). (Why: the spawner assigns `bPlayerMade` right after the spawn and `Age` is greater than 0 only from the first `Advance` on, so the colour/pitch decision is final when the sound starts. Each explosion therefore makes exactly one sound.) | Explosion |
| 3 | `BP_EnemyMissile.Die` and `BP_Flyer.Die` | Call `SfxHit()` on the game mode once per destroyed enemy, **on the path where an explosion destroys it** (the same path that writes `[DD][Enemy] Killed ...` or `[DD][Flyer] Killed ...`), after any guard that stops `Die` from running twice. If `Die` is also called from anywhere else than the explosion kill test in `Advance` (cleanup, `ClearEnemies`, `ClearFlyers`, a debug key), put the `SfxHit()` call instead directly after the call to `Die` inside the kill test of `Advance`, so cleanup stays silent. An enemy that reaches its target (`Impact`, `DamageSlot`) makes **no** hit sound; its blast makes the explosion sound. It does not matter whether the kill was made by the player's or by an enemy's blast. | Enemy hit |
| 4 | `BP_Flyer` | New variable `HumComp` (Audio Component object reference, not Instance Editable). In `Init`, **after the flyer type is known and stored**, at the end of its execution chain: `if NOT IsValid(HumComp): HumComp = game mode.CreateHum(bSatellite)`, where `bSatellite` is the same test `Init` already uses to show `GroupSatellite` rather than `GroupBomber`. In `ReceiveEndPlay`, at the end of its existing execution chain: `if IsValid(HumComp): HumComp.FadeOut(FadeOutDuration=0.15, FadeVolumeLevel=0.0, FadeCurve=Linear); Log "[DD][Sfx] Hum off"`. `ReceiveEndPlay` runs for every way a flyer disappears (killed, exits the screen, cleanup, game end), so the hum never outlives its flyer. | Hum |
| 5 | `BP_MissileBase.ConsumeAmmo` | After the node that decreases `Ammo`: `if Ammo == 0` (the value after the decrease) **and** the decrease really happened in this call, call `QueueAlarm()` on the game mode. A base that is destroyed with ammo left, a refill and a rebuild make no alarm. | Alarm |
| 6 | `BP_DDGameMode` wave start (the function that writes `[DD][Wave] Start n=...`, expected `StartWave`) | After that log node, at the end of the chain that runs at every wave start including wave 1: `SfxSiren()`. | Siren |
| 7 | `BP_DDGameMode` game over (the function that writes `[DD][Game] GAME OVER ...`, expected `GameOver`) | After that log node: `SfxGameOver()`. It must run exactly once per game over (if the function can be entered twice, put the call behind the same guard that stops the second entry; if there is none, stop and report). | Game over |
| 8 | `BP_DDGameMode` new game (the function that writes `[DD][Flow] Start`, expected `StartNewGame`) | After that log node: `ResetAlarms()`. | none |

"Game mode" in the rows above is obtained in each Blueprint the way the existing code of that Blueprint already does it (`GetGameMode` and a cast to `BP_DDGameMode`); if the cast fails, nothing happens.

### 3.6 Sound assets (all in `/Game/DomeDefense/Audio/`)

**Looping.** `dd_flyer_hum_loop`: set `bLooping` = true. No other sound loops (all others `bLooping` = false).

**Voice caps** (the human's busy waves have dozens of explosions; a phone cannot mix them all):

| Asset | Max simultaneous (`MaxCount`) | Resolution rule |
|---|---|---|
| `dd_explosion` | 6 | Stop Quietest |
| `dd_enemy_hit` | 4 | Stop Quietest |
| `dd_launch` | 4 | Stop Quietest |

- **Primary:** on each Sound Wave set `bOverrideConcurrency` = true, `ConcurrencyOverrides.MaxCount` and `ConcurrencyOverrides.ResolutionRule` (the enumerator named Stop Quietest; if the tool needs another spelling, list the enum entries and use the one that means Stop Quietest). Read back.
- **Fallback B** (only if the primary cannot be set): create a Sound Concurrency asset per row (`SC_DD_Blast`, `SC_DD_Hit`, `SC_DD_Launch`, in `/Game/DomeDefense/Audio/`) with the same values and assign it to the wave's concurrency set. Read back.
- **If neither works:** apply no caps, continue, and record it under *Deviations* with the exact error. Do not build a Blueprint workaround.
- **Fallback for `SetUISound`:** if the node does not exist on the Audio Component, leave the hum as created and record under *Deviations* ("the hum is not paused with the game").

### 3.7 Required assets
`/Game/DomeDefense/Audio/` must contain: `dd_launch`, `dd_explosion`, `dd_enemy_hit`, `dd_flyer_hum_loop`, `dd_alarm`, `dd_siren`, `dd_game_over` (all Sound Waves). `dd_tally_tick` may also be there; ignore it. If any of the seven is missing, rule 17 applies.

---

## 4. What this phase changes in earlier specs

| Earlier text | Now |
|---|---|
| The game is silent | Seven events make sound. The `[DD][Sfx]` log category is new. |
| 9d.2: `UpdateLook` ends with `ApplyPalette()` | It now ends with a call to `TryBlastSfx()`. |
| Frozen list | The six Blueprints of section 2 are unfrozen for the edits of this document only. |

---

## 5. Tooling notes

1. Variables of type Sound Base: add the variable, set the type to object reference of `SoundBase`, then set the default to the asset path (`/Game/DomeDefense/Audio/<name>`; if the tool needs the form `<path>.<name>`, use that). Read the default back as a path.
2. `PlaySound2D` is `UGameplayStatics::PlaySound2D`; its last pin is `Is UISound` (an advanced pin: it may be hidden; make it visible and set it to false, rule 13). `CreateSound2D` is the node "Create Sound 2D". `FadeIn`, `FadeOut` and `SetUISound` are nodes of the Audio Component.
3. New functions: `write_graph_dsl` once. Hooks into existing graphs: `create_node` + `connect_pins`; to insert after a node that already has an outgoing exec wire, connect the new node between them and read the graph back to show the chain stayed whole.
4. `ReceiveEndPlay` already exists in `BP_Flyer`; add to its chain, do not create a second end-play event.
5. If a tool reports that a property cannot be read or set, say so; do not guess.

---

## 6. Implementation steps

### Step 0 — Read (no changes)
1. List `/Game/DomeDefense/Audio/` (assets and classes). For each of the seven sounds record `Duration`, sample rate, channels, `bLooping`, the concurrency settings, and the compression setting (read only).
2. `BP_DDGameMode`: list functions and variables; confirm none of the names of 3.2 to 3.4 exist. Read the functions that write `[DD][Wave] Start`, `[DD][Game] GAME OVER` and `[DD][Flow] Start` completely, and name them (expected `StartWave`, `GameOver`, `StartNewGame`). For each: the last node of its main execution chain, and whether it can be entered twice. Read `ApplyPause`, `PauseGame` and `ResumeGame` and say how the game is paused (for example `SetGamePaused`).
3. `DDLog` in `BFL_DDDebug`: its inputs, and how an existing node uses it (for `[DD][Explosion] Start`).
4. `BP_DDPlayerController.LaunchAt`: read completely; name the live chain, the node that finishes spawning the ABM, and where `bLaunched` is set. Say which chain is the disconnected older one.
5. `BP_Explosion`: read `UpdateLook` (the last node of its chain), `SetRadius`, `Advance`, `Init`. Confirm that `Age` is greater than 0 when `UpdateLook` runs during `Advance`, and say whether `UpdateLook` is also called while `Age` is 0.
6. `BP_EnemyMissile` and `BP_Flyer`: read `Die`, `Advance`, `ReceiveEndPlay` completely. List **every caller of `Die`** in the whole project (including `BP_DDGameMode.ClearEnemies`, `ClearFlyers`, the debug keys). For each of the two `Die`: where the `[DD][Enemy] Killed` / `[DD][Flyer] Killed` log node sits, whether `Die` has a guard against running twice, and which of the two placements of 3.5 row 3 applies.
7. `BP_Flyer`: read `Init` completely. Name the variable that holds the flyer type, the value that means satellite, and the node whose output decides `GroupSatellite` versus `GroupBomber`. Say what `ReceiveEndPlay` does today and what its last node is.
8. `BP_MissileBase`: read `ConsumeAmmo`, `SetAmmo`, `Demolish`, `Rebuild`. State exactly when `Ammo` is decreased and whether the function can run with `Ammo` already 0.
9. For each Blueprint of section 2, state how it obtains the game mode today (node and cast).
10. Tool check (no changes): confirm that the node types `PlaySound2D` (with the `Is UISound` pin), `CreateSound2D`, `SetUISound`, `FadeIn`, `FadeOut`, `SetTimerByFunctionName`, `RandomFloatInRange` can be created, and that the properties `bLooping`, `bOverrideConcurrency` and `ConcurrencyOverrides` of a Sound Wave can be read.

**Check:** the report section for Step 0 is complete.

### Step 1 — Sound assets
1. `dd_flyer_hum_loop`: `bLooping` = true. Read back. Confirm the other six are not looping.
2. Voice caps (3.6) on `dd_explosion`, `dd_enemy_hit`, `dd_launch`. Read back.

**Check:** the properties read back as specified (or the Fallback/Deviation is recorded). **Save All.**

### Step 2 — `BP_DDGameMode`
Read the Blueprint again first.
1. Add the variables of 3.2; read back names, types, defaults, categories, flags.
2. Write the functions of 3.2, 3.3 and 3.4; read each back.
3. Add the three hooks (3.5 rows 6, 7, 8); show each function before and after.
4. Compile.

**Check:** `BP_DDGameMode` compiles clean; every function reads as specified; the three existing functions read exactly as before plus one call each; every other graph and variable reads exactly as before. **Save All.**

### Step 3 — The other Blueprints
Order: `BP_Explosion`, `BP_MissileBase`, `BP_EnemyMissile`, `BP_Flyer`, `BP_DDPlayerController`. For each: read it again first, add what 3.5 says, show the graph before and after, compile. Details: `BP_Explosion` row 2; `BP_MissileBase` row 5; `BP_EnemyMissile` and `BP_Flyer` row 3 (and `BP_Flyer` row 4); `BP_DDPlayerController` row 1.

**Check:** each compiles clean; each changed graph reads exactly as before plus the stated nodes; no other graph changed. **Save All.**

### Step 4 — Run
1. **Save All.**
2. `StartPIE`. The title screen appears at once. **Wait about 20 seconds**; nobody taps, so no game starts and no sound event happens. **No screenshots.**
3. `GetLogEntries` (filter `[DD]`), then `StopPIE`.
4. Fill in the report (section 8).

---

## 7. Acceptance tests

### A. Static checks
- [ ] **A1** Changed assets: `BP_DDGameMode`, `BP_DDPlayerController`, `BP_Explosion`, `BP_EnemyMissile`, `BP_Flyer`, `BP_MissileBase`, and the Sound Waves `dd_flyer_hum_loop`, `dd_explosion`, `dd_enemy_hit`, `dd_launch`. Created: nothing, except the Sound Concurrency assets if Fallback B was used. Nothing else: no material, no `dd_tally_tick`, nothing under `Spike`, no Project Settings, input, audio or `.ini` change. `L_Game` unchanged, no temporary actor placed.
- [ ] **A2** All six Blueprints compile with 0 errors and 0 warnings.
- [ ] **A3** Step 0 findings are in the report (all ten items).
- [ ] **A4** The variables of 3.2 exist with the stated types, defaults, categories and flags; the seven sound defaults read back as the seven asset paths.
- [ ] **A5** The functions of 3.2, 3.3 and 3.4 read back as written; the three game-mode hooks are shown before and after.
- [ ] **A6** Hooks 1 to 5 are shown before and after in each Blueprint; each is on the stated chain; hook 3 states which placement was used and why (callers of `Die`).
- [ ] **A7** The sound assets read back: the hum loops, the others do not; the three caps are set (or the Fallback or Deviation is stated).
- [ ] **A8** **Pin audit (rule 13):** a table of every input pin of every node in the new functions and of every hook node (target, name and value pins, wired from). It shows `Is UISound` = false on the `PlaySound2D` in `PlaySfx`.
- [ ] **A9** **Unchanged (rule 3):** a table that shows, for each item of rule 3 and for every function not named in this document, that it reads exactly as before, including `RadiusAtAge`, `ContainsPoint`, `Advance`, `Die` (apart from the one call), `DamageSlot`, `SpawnExplosion`, and the existing `[DD]` log nodes.
- [ ] **A10** The seven sound assets still exist at their paths; `dd_tally_tick` is untouched.

### B. Log checks (agent; the one 20 s run, nobody taps)
- [ ] **B1** The startup lines as in the 9d.2 run: camera, GameMode, config (`Width=2250.0 Height=1800.0`), bases 3, cities 6, PlayerController, view target, crosshair, HUD, `[DD][Area] Viewport=...`.
- [ ] **B2** Exactly one of `[DD][Save] Loaded high=<n>` or `[DD][Save] No file high=<n>`, followed by exactly one `[DD][Flow] Title high=<n>`.
- [ ] **B3** During the 20 s wait: no `[DD][Sfx]` line, no `[DD][Wave]`, `[DD][Enemy]`, `[DD][Flyer]`, `[DD][Score]`, `[DD][Damage]`, `[DD][Explosion]`, `[DD][Input] Fire request`, `[DD][Aim]`, `[DD][Game]`, and no `[DD][Flow]` line except the one `Title` line.
- [ ] **B4** No line containing `ERROR`, `Accessed None`, `Divide by zero` or `Blueprint Runtime Error`.

### D. Interactive checks (human, PIE) — all NEEDS-HUMAN
List every item as NEEDS-HUMAN in the report; do not run or claim them. The human compares the sounds heard with the log counts.
- [ ] **D1** **Launch:** a launch sound on every ABM fired; none on a tap with no ammo; log: the number of `[DD][Sfx] Launch` lines equals the number of `[DD][Input] Fire request` lines.
- [ ] **D2** **Explosion:** one explosion sound per blast, chain reactions included; blasts at a city or base sound lower and heavier than your own; log: `[DD][Sfx] Blast` lines equal `[DD][Explosion] Start` lines, and `player=0` lines are the enemy-made blasts.
- [ ] **D3** **Enemy hit:** a hit sound for every enemy missile or flyer destroyed by a blast; none when a missile hits a city or base; log: `[DD][Sfx] Hit` lines equal the `[DD][Enemy] Killed` lines plus the `[DD][Flyer] Killed` lines.
- [ ] **D4** **Hum:** a bomber hums low and a satellite hums higher; it starts when the flyer appears and fades when it is killed or leaves; no hum after the flyer is gone; the hum is silent while the game is paused; log: each `Hum on` is followed by a `Hum off`.
- [ ] **D5** **Siren:** at the start of every wave including wave 1, never on the title screen; log: `[DD][Sfx] Siren` lines equal `[DD][Wave] Start` lines.
- [ ] **D6** **Alarm:** once for each base whose last ABM is fired; if two or three bases run dry close together the alarms play one after another with no overlap; none for a base destroyed with ammo left; log: `Alarm queued` and `Alarm play` lines in matching numbers.
- [ ] **D7** **Game over:** the game-over sound plays once; log: exactly one `[DD][Sfx] GameOver`.
- [ ] **D8** **Balance:** the relative loudness of the sounds is acceptable (the human edits the `Vol...` variables in the Class Defaults of `BP_DDGameMode`); a busy late wave is a pile of blasts but not a wall of noise.
- [ ] **D9** **No gameplay change:** kills, chain reactions, scoring, damage, waves, flyers, pause and game over behave exactly as before.
- [ ] **D10** No errors in the play log (`Blueprint Runtime Error`).

### D-DEVICE. Android (NEEDS-DEVICE)
- [ ] **T-1** Sound comes out of the phone speaker and the headphones; no crackle or dropouts in a busy wave; the frame rate is not noticeably lower than before; the explosion is clearly audible on the phone speaker.

### E. Hygiene
- [ ] **E1** Save All done; PIE stopped; the editor is on `L_Game`; nothing under `Spike` was touched.

---

## 8. Report format (the agent's final message)

```
## Phase 9e report (sound)
**Engine version:** x.y.z
**MCP tools used:** (list)

### Step 0 findings
(the ten items of Step 0; callers of Die for both Blueprints; the game-mode access in each Blueprint; how pause works)

### Sound assets (Step 1)
(each asset: looping, concurrency settings as read back; Fallback used?)

### BP_DDGameMode (Step 2)
(variables; the functions as read back; the three hooks before and after)

### Other Blueprints (Step 3)
(each hook before and after)

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
| Seven sounds behind seven variables and seven volume numbers in `BP_DDGameMode` | The human tunes loudness by editing the `Vol...` numbers; a sound is replaced by re-importing a file over the asset or by changing the `Sfx...Sound` variable. |
| `[DD][Sfx]` log lines | The play log can be audited: each sound event is one line that matches an existing line. |
| `PlaySfx(Sound, Volume, Pitch)` | A later phase that adds a sound (for example the tally tick when the HUD count-up exists) calls this function. |

**Known leftovers that stay (removed in 9g):** the unused `dd_tally_tick` asset, the hidden `Disc` of `BP_Explosion` and `MI_DD_Explosion`, the hidden cubes of `BP_Flyer`, the hidden `Dot` of `BP_TapMarker` and of the missiles, the old `UpdateSmartTail` (uncalled), `StopSpawner`, `bDirectorSilent`, the two disconnected spawn chains in `SpawnRandomEnemy`, the disconnected older fire chain in `LaunchAt`, HUD debug lines 2 to 4, `BP_PlayfieldDebug`, the debug keys, the unused `MI_DD_Enemy` and `MI_DD_Abm`, the `Spike` folder.

---

## 10. Common pitfalls

1. **`Is UISound` left at true** on `PlaySound2D` (the pin may be hidden): the sounds would keep playing while the game is paused.
2. **Playing the explosion sound at spawn instead of in `TryBlastSfx`:** `bPlayerMade` is not final yet, so every blast would sound like the player's.
3. **Putting `SfxHit()` where cleanup also runs** (`ClearEnemies`, `ClearFlyers`, debug keys): a burst of hit sounds at every wave end or reset. Use the placement rule of 3.5 row 3.
4. **Using the disconnected older fire chain in `LaunchAt`:** no launch sound would ever play.
5. **Creating a second end-play event in `BP_Flyer`:** `ReceiveEndPlay` already exists; extend its chain.
6. **Letting the hum outlive its flyer:** the hum component is not owned by the flyer; `ReceiveEndPlay` must fade it out on every path.
7. **Alarm on every `ConsumeAmmo` call or when a base is destroyed:** only the decrease that ends at 0 counts.
8. **`SetTimerByFunctionName` with a misspelled name:** the queue would play one alarm and stall. The name is `PlayNextAlarm`.
9. **Rewriting existing graphs** with the graph-writing tool (rule 11). Only inserted nodes.
10. **Marking D items PASS.** You cannot hear. They are NEEDS-HUMAN and NEEDS-DEVICE.
11. **Touching `Spike`, `dd_tally_tick` or any other frozen asset**, or cleaning up what this document does not list.
