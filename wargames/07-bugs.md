# WARGAME 07 — Game Design Forge Phase 1: Bug Terrain & Battle Plan

**ASSUMPTION HEADER:** Fought under: TARGET REPO = greenfield tree defined by `GAME-DESIGN-FORGE-BLUEPRINT.md` (no code exists; the blueprint IS the terrain). MISSION = build Phase 1 to Definition-of-Done (blueprint §7 steps 7, 9, 10, 11) on Windows + UE 5.x with zero human help. A real brief overrides this line.

**Audience:** the executor agent (Claude Code, cheaper model) that will build Phase 1 from `GAME-DESIGN-FORGE-BLUEPRINT.md`. You were not in the planning room. Everything you need is here. Do not ask questions; every fork has a trigger and a route.

**Mission:** take the blueprint from empty machine to Definition-of-Done (blueprint §7: steps 7, 9, 10, 11 pass).

**Ground truth about this plan:** the blueprint's C++ was written against UE 5.4 headers *without a compile pass* (the planner had no Unreal install). Treat every API call as "probably right, verify on first touch." That is not a defect; it is the terrain. Sweeps below pre-map the breaks.

**Standing rules for the executor:**
- After every build, capture the full error list before fixing anything. Fix the FIRST error only, rebuild. MSVC/UBT cascade errors; error #2 is usually a lie.
- Never edit engine source. Every fix lands in `Plugins\Forge*` or `Source\Forge`.
- Commit after every green build with message `wg07: <move id> green`.
- If a counter-move below doesn't resolve its failure in 2 attempts, log it and check the ABORT table before improvising.
- **Paper trail:** work on ONE branch `wg07-phase1`. One commit per fix/move (`wg07: <move id> green`). Update `PROGRESS.md` after every sweep. Append ONE line to `log.md` at mission end.
- **Bounded scope:** fix only what this mission names. Never refactor beyond it. Never retune blueprint constants (thresholds, Voronoi multiplier, cell size convention, restart cap) beyond the counters explicitly written below — anything else you *want* to change goes in PROGRESS.md as a flag, not a commit.

---

## MOVE 0 — GROUND TRUTH (always first, before any mission work)
- **Do:** if any repo exists at the target path, build it and run whatever tests exist BEFORE touching anything. If greenfield (expected), Move 0 = blueprint §2 step 3: the untouched Blank C++ project must build and open.
- **Expected observation:** green build, editor opens, no plugin errors in `LogInit`.
- **If red:** that red is **Bug 0** and outranks the mission. Fix the environment/baseline first (M1.1 counters), or hit Abort A2.

---

## RECON PHASE (read-only, run before any build)

There is no existing repo — greenfield confirmed by the planner. Recon therefore = environment + engine-API recon on YOUR machine.

| # | RECON NEEDED | Exact check that settles it | Feeds |
|---|---|---|---|
| R1 | UE version actually installed | `dir "C:\Program Files\Epic Games\"` → note `UE_5.x` folder; open `Engine\Build\Build.version`, read `MinorVersion` | Fork F1 |
| R2 | VS toolchain complete | Run `"C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Auxiliary\Build\vcvars64.bat" && cl` → prints MSVC 19.3x; check Win SDK ≥ 10.0.22621 in VS Installer | Move 1.1 |
| R3 | `ApplyExternalStrain` signature in installed engine | Open `Engine\Source\Runtime\Experimental\GeometryCollectionEngine\Public\GeometryCollection\GeometryCollectionComponent.h`, search `ApplyExternalStrain` — record exact params | Sweep 2 M2.4 |
| R4 | `SetDamageThreshold` / `SetPerLevelDamageThreshold` exist as component API | Same header, search both names; if absent, search `DamageThreshold` in `UGeometryCollection` (the *asset*) instead | Fork F3 |
| R5 | `FFractureEngineFracturing::VoronoiFracture` reachable from an editor module | Search `Engine\Source\...\FractureEngine\Public\` for `FractureEngineFracturing.h`; note module name in its `.Build.cs` | Sweep 2 M2.2 |
| R6 | `TBitArray<>` API: does `CountSetBits()` exist in this version | Search `Engine\Source\Runtime\Core\Public\Containers\BitArray.h` for `CountSetBits` | Sweep 1 M1.2 |
| R7 | Disk ≥ 120 GB free, RAM ≥ 16 GB | `wmic logicaldisk get freespace` / Task Manager | Abort A1 |
| R8 | Demo modular meshes available | Confirm Starter Content or any 8-mesh modular kit importable; if none, plan on UE Cube/basic shapes scaled to 200uu as socket-tagged stand-ins (functional, ugly — acceptable for DoD step 7) | Sweep 1 M1.5 |

**Confidence labels used throughout:** `[PROOF]` = follows on paper from the algorithm/spec; `[HIGH]` = verified against public UE docs/source knowledge, version drift possible; `[JUDGMENT]` = planner's best estimate, treat as a hypothesis. Every RECON row above is by definition `[JUDGMENT]` until its check runs. Sweep failure predictions are `[HIGH]` unless labeled otherwise.

**Fork F1 (trigger on R1):** installed engine is 5.4.x → proceed on blueprint as written. 5.3.x → proceed but expect R3/R4 deltas (Chaos component API churned between 5.3→5.4). 5.5+ → same, plus check `TObjectPtr` deprecation warnings-as-errors; if UBT treats warnings as errors, add `bWarningsAsErrors = false;` to plugin Build.cs. **No engine version is a blocker; only the API-recon results change routes.**

---

## "LOOKS BROKEN BUT ISN'T" — do NOT fix these (locked decisions / documented fallbacks)

| What you'll see | Why it's correct — leave it |
|---|---|
| WFC restarts on contradiction instead of backtracking | Locked Phase 1 design (blueprint §4 step 6). Backtracking is scope creep — see Abort A4. `[PROOF]` |
| Breakable actors are Kinematic and totally inert until damaged | Intentional (§5 step 4) — prevents "level crumbles on Play". Do not set Dynamic at spawn. `[PROOF]` |
| "Deformation" is just multi-level fracture, no soft-body | Phase 1 scope decision. Chaos Flesh is Phase 2. Don't add it. `[PROOF]` |
| Socket tagging is a manual dialog at import, not auto-inferred | Phase 2 R&D, explicitly out (§1). Don't attempt geometry inference. `[PROOF]` |
| The `air` tile has no mesh | Mandatory empty tile (M1.5). Not a missing-asset bug. `[PROOF]` |
| Tutor overlay is a card list, not a spline node graph | Acceptable per M3.3 — DoD 10 doesn't require splines. Don't gold-plate. `[PROOF]` |
| Yaw ring table looks "wrong" against your mesh set | Planned empirical calibration (M1.4 f3), not a solver bug. Permute rows, don't rewrite the solver. `[HIGH]` |
| Preset threshold numbers (50/500/5000/50000) feel arbitrary | Design values. Retuning them is out of bounds — flag in PROGRESS.md if they play badly. `[JUDGMENT]` |
| ForgeTutor missing from the packaged game | Correct — Editor-only plugin (V7 asserts its absence). `[PROOF]` |
| IntelliSense red squiggles everywhere | Cosmetic; only UBT output counts (M1.1). `[HIGH]` |

## SWEEP 1 — ForgeTiling (grid snap + WFC solver)

### M1.1 Baseline project builds
- **Do:** blueprint §2 steps 1–3 exactly.
- **Expect:** VS build `Build: 2 succeeded`, editor opens to empty level, `LogInit` shows no plugin errors.
- **Likely failure:** UBT error `Visual Studio 2022 must be updated...` or missing SDK. **Signals:** R2 incomplete. **Counter:** VS Installer → add MSVC v143 + Win10 SDK 10.0.22621, retry. Second-likeliest: project generates but IntelliSense floods red — ignore IntelliSense entirely; only UBT output counts.
- **Gate:** do not pass without a green baseline. Everything after assumes it.

### M1.2 Plugin skeletons compile empty
- **Do:** create the three plugin folders with `.uplugin` + `Build.cs` + one empty module class each. ForgeTiling `Build.cs` deps: `Core, CoreUObject, Engine`. Regenerate projects, build.
- **Expect:** `Build: 5 succeeded` (game + 3 plugins + UnrealEditor target artifacts vary; the number that matters is zero failures), editor lists all three under Edit→Plugins→Project.
- **Likely failure:** `Plugin 'ForgeTiling' failed to load because module could not be found` at editor start. **Signals:** module `Type` in `.uplugin` wrong or `LoadingPhase` missing. **Counter:** runtime plugins → `"Type": "Runtime", "LoadingPhase": "Default"`; ForgeTutor → `"Type": "Editor"`. 
- **Fork F2:** if the editor crashes on boot after adding plugins (not just fails to load), launch with `-NoPlugins`? No — instead delete `Binaries\` + `Intermediate\` in the offending plugin, rebuild. Stale DLLs after Build.cs edits are the #1 cause of boot crashes in plugin dev.

### M1.3 TileSet DataAsset + solver compiles
- **Do:** add `ForgeTileSet.h`, `ForgeWFCSolver.h/.cpp` from blueprint §6.1. Implement `PickMinEntropyCell` and `CollapseCell` per the comment spec (linear scan entropy; weighted roulette).
- **Expect:** clean build; in editor, right-click Content → Miscellaneous → Data Asset → `ForgeTileSet` appears and opens with CellSize/Tiles/BoundarySocket fields.
- **Likely failures, in probability order:**
  1. `FString Sockets[6]` — **UHT rejects raw C-arrays of FString in USTRUCT on some versions.** Signals: UHT error mentioning `Sockets`. Counter: change to `UPROPERTY(EditAnywhere) TArray<FString> Sockets;` with a `PostLoad`/validation that pads to 6; update solver indexing accordingly. (This is the single most likely compile break in the whole plan.)
  2. `TBitArray<>` lacks `CountSetBits` (per R6). Counter: replace with `Cell.CountSetBits()` → manual loop `int n=0; for(auto b:Cell) n+=b;` or `TBitArray::Num()` iteration; 5-line helper `static int32 SetBits(const TBitArray<>&)`.
  3. `TQueue<FIntVector>` include missing → add `#include "Containers/Queue.h"`.
- **Verification run V1 (unit, no UI):** add a console command `Forge.WFC.SelfTest` (FAutoConsoleCommand) that builds an in-memory 2-tile set (`floor` all sockets `air_s`... actually: tile A all sockets `s0`, tile B sockets `s1`, no shared sockets) and solves 4×4×1. **Pass looks like:** log prints `WFC selftest: homogeneous grid OK (16 cells, all tile A or all tile B)` and a second case with an unmatchable boundary prints the §4-step-6 error string mentioning the socket pair. If the failure message does NOT name a socket pair, the error-path plumbing is broken — fix before proceeding, it's your main field-debug tool.

### M1.4 AForgeTileVolume + HISM spawn + grid snap
- **Do:** actor with `FIntVector GridDims`, `UForgeTileSet*`, `TMap<FIntVector,int32> OccupiedCells`, a details-panel button (`CallInEditor` UFUNCTION `AutoFill()`), per-mesh HISM components created on demand; grid-snap helper per blueprint §6.1 tail.
- **Expect:** placing the volume in a level and clicking Auto-Fill spawns instances filling the box.
- **Likely failure 1:** button does nothing, log shows solver success. **Signals:** HISM components created but not registered. **Counter:** after `NewObject<UHierarchicalInstancedStaticMeshComponent>`, call `RegisterComponent()` and `AttachToComponent(RootComponent, ...)`; instances via `AddInstance(FTransform, /*bWorldSpace=*/false)`.
- **Likely failure 2:** tiles spawn but interpenetrate/gap. **Signals:** cell size ≠ mesh footprint or pivot not at mesh min-corner/center consistently. **Counter:** decide pivot convention = mesh CENTER; spawn at `(Cell + 0.5) * CellSize` minus volume origin; log first 3 transforms and eyeball against bounds. Do not "fix" by nudging constants; fix the convention.
- **Likely failure 3:** rotated variants visibly wrong (wall sockets match but mesh faces wrong way). **Signals:** the yaw ring table order doesn't match your mesh axis convention. **Counter:** create a single L-corner test tile, place all 4 yaws side by side manually, and if wrong, the fix is reordering `RingSrcForYaw` rows — trial the 4 cyclic permutations; exactly one produces a closed square from 4 corners. RECON cannot settle this (depends on the art's forward axis); it is a planned 30-minute empirical calibration, not a bug.

### M1.5 Demo tile set + real fill
- **Do:** per R8, import/tag 8 meshes (floor, wall, wall-corner in/out, doorway, ceiling, pillar, air/empty). `air` tile (no mesh, all sockets `air_s`) is MANDATORY — without an empty tile most boxes are unsolvable.
- **Expect (this is DoD step 7):** 20×20×3 Auto-Fill produces a seamless enclosed room with visible variation, < 2s, zero contradictions across seeds 1–20 (loop a test command).
- **Likely failure:** frequent 10-restart failures. **Signals:** over-constrained set (e.g. no straight-wall↔corner mate). **Counter:** read the error's socket pair, add the missing mate or relax boundary to allow `wall_s` on edges. If failures are rare (<1 in 20 seeds) ship it; restarts are the Phase 1 design.

---

## SWEEP 2 — ForgeDestruction (one-click breakable + damage pipeline)

### M2.1 Module links against Chaos
- **Do:** ForgeDestruction `Build.cs` deps: `Core, CoreUObject, Engine, GeometryCollectionEngine, Chaos, ChaosSolverEngine, FieldSystemEngine`. Compile with just the header included, empty component.
- **Expect:** green build.
- **Likely failure:** unresolved externals mentioning `Chaos::` or `FGeometryCollection`. **Signals:** missing module in deps (Chaos split modules differ by version). **Counter:** grep the engine for the unresolved symbol's header, open that module's `.Build.cs`, add its module name. Mechanical; ≤3 iterations.

### M2.2 Editor "Make Breakable" action
- **Do:** separate `ForgeDestructionEditor` module (`Type: Editor`), deps add `UnrealEd, ToolMenus, FractureEngine, GeometryCollectionEditor, AssetTools`. Register a level-viewport context-menu entry that: creates `UGeometryCollection` asset next to the source mesh, appends the static mesh, runs Voronoi with `Clamp(Volume_m3*12, 8, 96)` sites, swaps actor, attaches `UForgeBreakable`.
- **Expect:** right-click a StaticMeshActor → "Forge: Make Breakable" appears; clicking produces a new `AGeometryCollectionActor` in place, visually identical until damaged, and a `GC_<mesh>` asset in the content browser.
- **Likely failure 1 (HIGH, per R5):** `FGeometryCollectionConversion` / `FFractureEngineFracturing` are editor-private or renamed in your engine version — planner could not verify. **Signals:** compile can't find the header, or link fails despite module dep. **Fork F3 trigger:** if 30 minutes of grepping doesn't yield a public path, **take Route B: manual-fracture fallback** — the context action instead opens UE's built-in Fracture Mode with the asset pre-created, and a one-page doc step tells the user "click New → Fracture → Voronoi → Fracture (defaults are pre-set by Forge)". Zero-config degrades to two-click; DoD step 9 is unaffected. Log the downgrade prominently in the final report.
- **Likely failure 2:** fractured actor's pieces fall immediately on PIE. **Signals:** initial state Dynamic. **Counter:** ensure component sets Kinematic in BeginPlay AND the actor's GeometryCollection component has `Simulate Physics` respecting it; if still falling, anchor the bottom-most bone via Fracture Mode's Anchored flag (per-asset, one click, scriptable later).

### M2.3 Runtime damage → wake → strain
- **Do:** `UForgeBreakable` per blueprint §6.2, adjusted to the R3/R4 recon results.
- **Expect:** PIE, apply 1000 damage ×1 to a Glass wall → shatters; Stone wall needs ~5 hits; log line per hit `[Forge] <actor> dmg=<n> acc=<n>`.
- **Likely failure 1:** `SetDamageThreshold`/`SetPerLevelDamageThreshold` don't exist on the component (R4). **Counter:** set thresholds on the **asset** instead: `GC->GetRestCollection()`-derived `UGeometryCollection` has `DamageThreshold` TArray + `PerLevelDamageThreshold` bool — set at bake time in the editor action, not at BeginPlay. Move the preset numbers there; runtime component keeps only accumulation+wake.
- **Likely failure 2:** `ApplyExternalStrain` signature mismatch (R3). **Counter:** match recon'd signature; if the method is absent entirely, Route B: spawn a transient `UFieldSystemComponent` and apply a `URadialFalloff` strain field at impact point — this is the documented Chaos path and always exists. (~25 lines; UE "Fields" docs pattern.)
- **Likely failure 3:** `OnTakeAnyDamage` never fires. **Signals:** template projectile doesn't call `ApplyDamage`. **Counter:** UE FPS template projectile applies impulse, not damage — add `UGameplayStatics::ApplyPointDamage` in `BP_Projectile` hit event (template content edit, 2 minutes).

### M2.4 Debris hygiene
- **Do:** timer → `CrumbleActiveClusters()` (or, if absent this version, `SetVisibility(false)` + destroy actor when all bodies asleep).
- **Expect:** 60s of repeated destruction in PIE holds > 60 fps in a Development build with ≤ 3 walls broken concurrently; `stat chaos` shows body count returning to baseline after each fade.
- **Likely failure:** fps collapse on first break. **Signals:** site count too high or collision per-shard convex too dense. **Counter:** halve the Voronoi multiplier (12→6), set shard collision to `Implicit Box` at bake; re-verify.

---

## SWEEP 3 — ForgeTutor + templates + package (integration terrain)

### M3.1 Telemetry bus
- **Do:** `UForgeTelemetry : UEditorSubsystem` with `TMulticastDelegate<void(FName, const TMap<FString,FString>&)>` + counters map. ForgeTiling fires `Forge.TilePlaced` after each successful placement/AutoFill (count = instances). Hook `FEditorDelegates::BeginPIE` → `Editor.PIEStarted`.
- **Expect:** console command `Forge.Telemetry.Dump` prints counters; placing tiles increments `Forge.TilesPlaced`.
- **Likely failure:** ForgeTiling is a Runtime module and can't see an Editor subsystem. **Signals:** compile error including editor headers from runtime module. **Counter:** guard with `#if WITH_EDITOR` around the fire-site and add `UnrealEd` to ForgeTiling's `PrivateDependencyModuleNames` **inside** `if (Target.bBuildEditor)` in Build.cs. This exact pattern is also the packaging fix in M3.4 — get it right once here.

### M3.2 Graph runtime + JSON load
- **Do:** `UForgeTutorGraph` parses blueprint §6.3 schema (`FJsonSerializer`), node/edge maps, state machine per node `type`.
- **Expect:** unit console command `Forge.Tutor.SelfTest` loads `first_level.json`, simulates events `Forge.TilePlaced`×10 → `Editor.PIEStarted`, prints traversal `n_welcome → n_place → n_branch → n_play` and, with only 3 placements, the false branch `n_branch → n_hint_place → n_place`. **Pass = both traversal lines exact.**
- **Likely failure:** JSON file not found in packaged/staged paths. **Counter:** load via `IPluginManager::Get().FindPlugin("ForgeTutor")->GetBaseDir() / "Resources/Tutorials/..."` — never relative paths.

### M3.3 Overlay UI
- **Do:** editor tab (register via `FGlobalTabmanager`) hosting a UMG/Slate panel: vertical card list is ACCEPTABLE for Phase 1 if spline connectors overrun budget — cards + "current step" pulse + edge list as text meets DoD step 10. Do not sink days into pretty splines; that is Phase 2 polish.
- **Expect:** Window menu shows "Forge Tutor"; performing real editor actions advances the highlighted card live.
- **Likely failure:** events fire but UI doesn't update. **Signals:** delegate bound on game thread vs slate tick. **Counter:** marshal via `AsyncTask(ENamedThreads::GameThread, ...)` and drive UI from a 0.25s ticker polling graph state rather than direct delegate→widget calls.

### M3.4 Genre templates + package (DoD 11)
- **Do:** build the 4 template maps per blueprint §3 (duplicate UE template pawns, retune constants). Then Package Project → Windows.
- **Expect:** package succeeds; `Forge.exe` runs Map_Start; tiling meshes present; breakable wall works; **no ForgeTutor code in the package**.
- **Likely failure 1 (HIGH):** packaging fails at UBT: editor-only includes in runtime modules. **Signals:** cook log names the module. **Counter:** the M3.1 `WITH_EDITOR` pattern, applied to every flagged file.
- **Likely failure 2:** packaged level empty where HISM tiles were. **Signals:** instances were editor-transient (added but volume not marked dirty/saved). **Counter:** after AutoFill call `Modify()` on the volume + mark package dirty, resave map; confirm instances serialize by reopening the editor before packaging.
- **Likely failure 3:** Chaos wall inert in package. **Signals:** damage BP edit (M2.3 c3) only in editor world or asset unsaved. **Counter:** verify `BP_Projectile` change saved; test in Standalone PIE mode first (catches 90% of package-only issues in 1/10th the time).

---

## ABORT CONDITIONS

| ID | Condition | Action |
|---|---|---|
| A1 | R7 fails (disk/RAM below floor) | STOP before install; report hardware gap. Nothing downstream is salvageable on an under-spec machine. |
| A2 | M1.1 baseline (untouched Blank C++ project) won't build after 3 toolchain-repair attempts | STOP; environment is broken in a way outside mission scope. Report exact UBT log tail. |
| A3 | Fork F3 Route B ALSO fails (built-in Fracture Mode can't produce a working Geometry Collection at all) | Engine install corrupt → verify/reinstall engine once; if it persists, STOP Sweep 2, ship Sweeps 1+3, report destruction as blocked with the fracture-mode repro. |
| A4 | WFC solver correct in SelfTest but real sets fail >50% of seeds even after socket audit (M1.5 counter applied) | Do NOT implement backtracking (scope creep). Ship with restart cap raised to 50 + the failure-listing UX; flag "needs constraint-authoring pass" in report. |
| A5 | Any counter-move requires editing engine source to work | Forbidden. Take the nearest Route B; if none exists, log and continue past that sub-feature. |
| A6 | You cannot run a verification at all (no PIE, no package, headless-only env) | Land NOTHING as "verified". Label every affected item **MANUAL-PENDING** in PROGRESS.md, say so plainly in the report. Never bluff a pass. |
| A7 | Abort triggered mid-sweep | Revert surgically: `git revert` the failing move's commits only — never reset the branch — then report. |

## FINAL VERIFICATION RUNS (executor performs ALL, in order; each states pass)

| V | Run | Pass looks like |
|---|---|---|
| V1 | `Forge.WFC.SelfTest` | homogeneous-grid OK line + named-socket-pair failure line (M1.3) |
| V2 | Auto-Fill 20×20×3 Demo set, seeds 1–20 scripted | ≥19/20 solve, each <2s, visual: closed seamless room, ≥2 mesh variants visible on floor |
| V3 | Grid-snap manual place: attempt an incompatible tile against an existing wall | red face wireframe shown, placement refused; compatible tile shows green and lands exactly on grid (`fmod(loc, CellSize)==0` logged) |
| V4 | PIE destruction matrix: Glass/Wood/Stone walls, count hits-to-break with 1000-dmg projectile | Glass=1, Wood=1, Stone=5, ±0; shards gone by t+9s; `stat fps` ≥60 during |
| V5 | `Forge.Tutor.SelfTest` | both traversal lines exact (M3.2) |
| V6 | Live tutor: fresh editor, open tutor, follow on-screen steps only | overlay advances through all 5 nodes without console intervention; timeout hint appears if you idle 2 min on n_place |
| V7 | Package + run `Forge.exe` | Map loads, V2-room visible, V4 Stone case reproduces in-game, no crash in 5 min play; `ForgeTutor` absent from `<Build>\Forge\Plugins\` |
| V8 | Clean-machine doc test (or fresh clone + deleted Binaries/Intermediate as proxy) | blueprint §7 steps 1–11 executable verbatim; any deviation you needed gets patched INTO the blueprint doc before final report |

**Report format on completion:** per-sweep status, forks taken (F1–F3 routes), counters consumed, V1–V8 pass/fail with confidence labels, any A-condition or MANUAL-PENDING items. That report + the patched blueprint is the handover.

**Paper trail on completion:** branch `wg07-phase1` with one commit per fix; `PROGRESS.md` updated per sweep; one line appended to `log.md`: `wg07 | <date> | sweeps 1-3 | V-passes: n/8 | forks: <list> | aborts: <list or none>`.

---

## AFTER THE RUN — SCORING (the wargame is not done until this is)
1. Compare your report **move by move** against each move's *expected observation* above. Every delta (observed ≠ expected, counter that didn't work, failure not predicted) is a scoring line.
2. Write the deltas into the front matter of the NEXT wargame file (`wargames/08-*.md`) as "inherited terrain."
3. Land results in `PROGRESS.md` and the one-line relay in `log.md`.
4. Cold re-run: **≥3 days later**, re-execute the V1–V8 block on the untouched branch. Any rot (pass→fail) is Bug 0 of the next mission.
