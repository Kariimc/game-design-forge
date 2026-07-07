# GAME DESIGN FORGE — Project Blueprint (v2, rescoped against reality)

> **Read this first.** The original vision ("build a native Windows engine from scratch") is a 5–10 year multi-team effort. This blueprint delivers the same *product* — code-free LEGO-style game assembly, auto-tiling, destruction, guided tutorials — as an **Unreal Engine 5 editor plugin suite**. UE5 gives us for free: DirectX 12 native Windows rendering, Chaos destruction physics, genre starter templates, a shipping-grade asset importer, and a built-in WFC plugin we fork. We build only the three things nobody ships: **(1)** socket-inferred auto-tiling on user assets, **(2)** zero-config destruction presets, **(3)** a reactive node-graph tutorial overlay.
>
> A junior-to-mid developer follows this document top to bottom with zero follow-up questions. Anything machine-specific is marked ⚙ and resolved in `wargames/07-bugs.md`.

---

## 1. Vision & Scope

**Product:** Windows-native desktop editor where a non-programmer drags assets in, paints levels that auto-connect seamlessly, presses Play, and gets a working game in any genre — while a visual node tutorial watches their actions and guides the next step.

**Phase 1 MVP (this document, ~2–4 months solo):**
- One UE5 project ("Forge") with 4 genre templates (FPS, 3D Platformer, Top-Down 2D, RPG-lite).
- `ForgeTiling` plugin: 3D grid-snap placement + socket-matching Wave Function Collapse auto-fill with weighted variation.
- `ForgeDestruction` plugin: one-click "Make Breakable" that auto-fractures any StaticMesh with sane defaults (no user physics config).
- `ForgeTutor` plugin: Slate/UMG node-graph overlay that tracks editor telemetry and advances tutorial steps.

**Explicitly OUT of Phase 1:** custom renderer, custom physics, marketplace, mobile export, multiplayer, auto-*inference* of sockets from raw geometry (Phase 2 R&D — Phase 1 uses a 30-second tagging dialog at import).

---

## 2. Tech Stack & Exact Installation

| Component | Choice | Why |
|---|---|---|
| Engine/Editor base | **Unreal Engine 5.4.x** | Chaos destruction built in; DX12 native; C++ plugin API; free until $1M revenue |
| Language | C++ (plugin logic) + Blueprints (templates) | Blueprints let non-programmers extend; C++ for solver perf |
| UI framework | Slate + UMG (built into UE) | Native editor panels, no Qt dependency |
| Data | JSON via `FJsonObject` + UE DataAssets | Human-diffable tutorial graphs; DataAssets for tile sets |
| Build | Visual Studio 2022 (v17.8+), MSVC v143, Win10/11 SDK | UE5.4 requirement |

### Install steps (executor runs verbatim)
1. Install **Visual Studio 2022 Community** with workloads: *Game development with C++*, *Desktop development with C++*. In Individual Components confirm: **MSVC v143**, **Windows 10 SDK (10.0.22621+)**, **.NET 6 SDK**.
2. Install **Epic Games Launcher** → Unreal Engine tab → install **UE 5.4.x** (⚙ any 5.4 patch; 5.3 works with one API delta noted in wargames).
3. Launcher → Launch UE → New Project → **Games → Blank → C++**, no starter content, name `Forge`, location `C:\Dev\Forge`. Let it generate and open in VS once, build `Development Editor | Win64`, confirm editor launches. **Do not proceed until this baseline builds.**
4. In editor: Edit → Plugins → enable **"Wave Function Collapse"** (Epic built-in, Experimental) — we read its source for reference but ship our own solver (theirs is geometry-script based and undocumented).

---

## 3. Project Directory Structure (complete)

```
C:\Dev\Forge\
├── Forge.uproject
├── Source\
│   └── Forge\                        # game module (runtime)
│       ├── Forge.Build.cs
│       ├── ForgeGameMode.h/.cpp
│       └── Characters\               # per-genre controllers (C++ base, BP children)
├── Plugins\
│   ├── ForgeTiling\
│   │   ├── ForgeTiling.uplugin
│   │   └── Source\ForgeTiling\
│   │       ├── ForgeTiling.Build.cs
│   │       ├── Public\
│   │       │   ├── ForgeTileSet.h        # DataAsset: tiles + sockets + weights
│   │       │   ├── ForgeGridSnap.h       # placement component
│   │       │   └── ForgeWFCSolver.h      # the solver
│   │       └── Private\ (matching .cpp)
│   ├── ForgeDestruction\
│   │   └── Source\ForgeDestruction\
│   │       ├── Public\ForgeBreakable.h   # one-click breakable component
│   │       └── Private\ForgeBreakable.cpp
│   └── ForgeTutor\
│       └── Source\ForgeTutor\
│           ├── Public\ForgeTutorGraph.h  # node schema + runtime
│           ├── Private\ForgeTutorGraph.cpp
│           └── Resources\Tutorials\*.json
├── Content\
│   ├── Templates\
│   │   ├── FPS\  Platformer3D\  TopDown2D\  RPG\
│   │   │   └── (Map_Start.umap, BP_Player, BP_GameMode, camera rig, physics preset)
│   ├── TileSets\Demo\                # 8 demo modular meshes (floor, wall, corner…)
│   └── ForgeUI\                      # tutor overlay widgets
└── wargames\
    └── 07-bugs.md                    # executor battle plan (companion file)
```

Genre templates are pure Content (maps + Blueprints): each template map has a `BP_GameMode` (spawn rules), `BP_Player` (movement tuned per genre: FPS = 600 walk speed + mouse look; Platformer = 1200 jump Z + spring-arm chase cam; TopDown = fixed 60° ortho-ish cam + click/WASD; RPG = TopDown + interaction trace). Duplicating UE's shipped template pawns and retuning is the fastest correct path — do not write movement from scratch.

---

## 4. Procedural Auto-Tiling Architecture

**Model:** *simple-tiled WFC with socket matching* (not the overlapping model — sockets are deterministic, debuggable, and match modular art workflows).

**Data model:** every tile face gets a **socket ID string** per direction (+X, −X, +Y, −Y, +Z, −Z). Two tiles may be adjacent along an axis iff `tileA.socket(+axis) == tileB.socket(−axis)`. Convention: symmetric sockets end in `s` (e.g. `wall_s`); asymmetric pairs use `f`/`m` suffixes (`door_f` mates only with `door_m`). Rotations: each tile declares `bAllowYawRotations`; the importer bakes 4 yaw variants by rotating the socket array (+X→+Y→−X→−Y).

**Ingestion (Phase 1):** drag mesh into `Content/TileSets/<Set>` → editor utility widget pops a 6-field socket dialog + weight slider → writes a row into the set's `UForgeTileSet` DataAsset, auto-computes grid cell size from the mesh bounds of the first tile (all tiles in a set must share footprint; mismatches are auto-scaled to fit and flagged).

**Solver algorithm (exactly what the code does):**
1. Init `Wave[cell] = bitmask of all tile-variants` for every cell in the requested box.
2. Apply boundary constraint: cells on the box edge remove variants whose outward socket ≠ `air_s` (configurable).
3. Loop: pick the uncollapsed cell with **minimum entropy** (fewest remaining variants; Shannon-weighted, ties broken randomly with the run seed).
4. **Collapse** it: pick one variant by weighted random (weights = variation control — same socket signature, different meshes = seamless variety).
5. **Propagate**: BFS from the collapsed cell; for each neighbor, remove variants whose facing socket has no surviving mate in the source cell's remaining set. Enqueue any neighbor that changed.
6. **Contradiction** (cell hits 0 variants): backtrack is overkill for Phase 1 — restart with `seed+1`, max 10 restarts, then fail loudly listing the offending socket pair (this error message is the #1 debugging tool; do not skip it).
7. All cells collapsed → spawn `HISM` (Hierarchical Instanced Static Mesh) instances per variant at `cellIndex * CellSize`, yaw per variant.

Complexity: O(cells × tiles × 6) per propagation wave — a 30×30×4 box with 40 variants solves in well under a second in a Development build.

---

## 5. Deformation & Destruction Pipeline

**Approach:** UE5 **Chaos Destruction** — Geometry Collections with Voronoi fracture — wrapped so the user never sees physics settings. True runtime soft-body deformation is out (Phase 2, Chaos Flesh); Phase 1 "deformation" = multi-level fracture, which reads as deformation-then-shatter and is what every shipped game does.

**"Make Breakable" one-click pipeline (what `UForgeBreakable` / the editor action does):**
1. User right-clicks any StaticMeshActor → **Forge: Make Breakable** (+ material preset: Wood/Stone/Glass/Metal).
2. Editor action builds a `UGeometryCollection` from the mesh via `FGeometryCollectionConversion`, runs **Voronoi fracture** — site count auto-scaled to mesh volume (`clamp(volume_m³ × 12, 8, 96)`), two cluster levels for large meshes (big chunks → shards).
3. Applies preset: damage thresholds per cluster level (Glass 50/10, Wood 500/100, Stone 5000/800, Metal 50000/9000), internal material auto-assigned (flat gray slot 1 if none), collision = implicit convex, `bEnableClustering=true`.
4. Replaces the actor with a `AGeometryCollectionActor`, initially **anchored/kinematic** — it does nothing until damaged (prevents the classic "level crumbles on Play" bug).
5. At runtime, impacts above threshold release strain → Chaos breaks clusters; our component adds impulse-scaled break events, debris lifetime (shards sleep → fade+destroy after 8s) to cap particle counts.

Trigger logic pipeline: `OnChaosBreakEvent` → accumulate damage per cluster → if `damage > threshold` propagate remaining strain to children (Chaos does this natively; we only set the numbers).

---

## 6. Phase 1 MVP Code Boilerplate

Full compilable sources live in the plugin folders per §3. The three core files are reproduced here in full so this document stands alone. (⚙ Written against UE 5.4 API; the wargame file maps every likely compile break to its fix.)

### 6.1 Grid snap + adjacency check — `ForgeGridSnap.h` (+ solver core)

```cpp
// Plugins/ForgeTiling/Source/ForgeTiling/Public/ForgeTileSet.h
#pragma once
#include "CoreMinimal.h"
#include "Engine/DataAsset.h"
#include "ForgeTileSet.generated.h"

UENUM() enum class EForgeDir : uint8 { PX, NX, PY, NY, PZ, NZ };

USTRUCT(BlueprintType)
struct FForgeTileDef {
    GENERATED_BODY()
    UPROPERTY(EditAnywhere) TObjectPtr<UStaticMesh> Mesh;
    // Socket IDs indexed by EForgeDir: PX,NX,PY,NY,PZ,NZ
    UPROPERTY(EditAnywhere) FString Sockets[6];
    UPROPERTY(EditAnywhere, meta=(ClampMin="0.01")) float Weight = 1.f;
    UPROPERTY(EditAnywhere) bool bAllowYawRotations = true;
};

UCLASS(BlueprintType)
class FORGETILING_API UForgeTileSet : public UDataAsset {
    GENERATED_BODY()
public:
    UPROPERTY(EditAnywhere) float CellSize = 200.f; // auto-set at import
    UPROPERTY(EditAnywhere) TArray<FForgeTileDef> Tiles;
    UPROPERTY(EditAnywhere) FString BoundarySocket = TEXT("air_s");
};
```

```cpp
// Plugins/ForgeTiling/Source/ForgeTiling/Public/ForgeWFCSolver.h
#pragma once
#include "CoreMinimal.h"
#include "ForgeTileSet.h"

// A "variant" = (tile index, yaw 0..3). Built once per solve.
struct FForgeVariant { int32 TileIdx; int32 Yaw; float Weight; FString Sockets[6]; };

class FORGETILING_API FForgeWFCSolver {
public:
    // Returns true on success; Out maps flat cell index -> variant index.
    bool Solve(const UForgeTileSet& Set, FIntVector GridDims, int32 Seed,
               TArray<int32>& OutCells, FString& OutError);
    const TArray<FForgeVariant>& GetVariants() const { return Variants; }

    // The adjacency check: may variant A sit on the Dir side of variant B?
    static bool Compatible(const FForgeVariant& A, const FForgeVariant& B, EForgeDir DirFromBToA) {
        static const int32 Opp[6] = {1,0,3,2,5,4};
        return B.Sockets[(int32)DirFromBToA] == A.Sockets[Opp[(int32)DirFromBToA]];
    }
private:
    TArray<FForgeVariant> Variants;
    void BuildVariants(const UForgeTileSet& Set);   // bakes 4 yaw rotations of the XY socket ring
    bool Propagate(FIntVector Dims, TArray<TBitArray<>>& Wave, FIntVector From, FString& Err);
    int32 PickMinEntropyCell(const TArray<TBitArray<>>& Wave, FRandomStream& Rng) const;
    int32 CollapseCell(TBitArray<>& Cell, FRandomStream& Rng) const; // weighted pick
};
```

```cpp
// Plugins/ForgeTiling/Source/ForgeTiling/Private/ForgeWFCSolver.cpp  (core loop; helpers per header)
#include "ForgeWFCSolver.h"

void FForgeWFCSolver::BuildVariants(const UForgeTileSet& Set) {
    Variants.Reset();
    // XY socket ring rotates PX->PY->NX->NY under +90° yaw; Z sockets fixed.
    static const int32 RingSrcForYaw[4][4] = { {0,1,2,3},{3,2,0,1},{1,0,3,2},{2,3,1,0} };
    for (int32 T = 0; T < Set.Tiles.Num(); ++T) {
        const FForgeTileDef& Tile = Set.Tiles[T];
        const int32 YawCount = Tile.bAllowYawRotations ? 4 : 1;
        for (int32 Y = 0; Y < YawCount; ++Y) {
            FForgeVariant V; V.TileIdx = T; V.Yaw = Y; V.Weight = Tile.Weight / YawCount;
            const int32* Src = RingSrcForYaw[Y];
            for (int32 R = 0; R < 4; ++R) V.Sockets[R] = Tile.Sockets[Src[R]];
            V.Sockets[4] = Tile.Sockets[4]; V.Sockets[5] = Tile.Sockets[5];
            Variants.Add(MoveTemp(V));
        }
    }
}

bool FForgeWFCSolver::Solve(const UForgeTileSet& Set, FIntVector D, int32 Seed,
                            TArray<int32>& Out, FString& Err) {
    BuildVariants(Set);
    const int32 N = D.X * D.Y * D.Z, VN = Variants.Num();
    if (VN == 0) { Err = TEXT("Tile set is empty."); return false; }
    for (int32 Attempt = 0; Attempt < 10; ++Attempt) {
        FRandomStream Rng(Seed + Attempt);
        TArray<TBitArray<>> Wave; Wave.Init(TBitArray<>(true, VN), N);
        // Boundary constraint
        auto Idx = [&](int32 X,int32 Yc,int32 Z){ return X + Yc*D.X + Z*D.X*D.Y; };
        bool bContradiction = false;
        for (int32 Z=0; Z<D.Z && !bContradiction; ++Z)
        for (int32 Yc=0; Yc<D.Y && !bContradiction; ++Yc)
        for (int32 X=0; X<D.X && !bContradiction; ++X) {
            const bool Edge[6] = { X==D.X-1, X==0, Yc==D.Y-1, Yc==0, Z==D.Z-1, Z==0 };
            TBitArray<>& Cell = Wave[Idx(X,Yc,Z)];
            for (int32 V=0; V<VN; ++V) if (Cell[V])
                for (int32 F=0; F<6; ++F)
                    if (Edge[F] && Variants[V].Sockets[F] != Set.BoundarySocket) { Cell[V]=false; break; }
            if (Cell.CountSetBits()==0) bContradiction = true;
            else if (!Propagate(D, Wave, FIntVector(X,Yc,Z), Err)) bContradiction = true;
        }
        // Observe/collapse loop
        while (!bContradiction) {
            const int32 C = PickMinEntropyCell(Wave, Rng);
            if (C < 0) { // all collapsed — emit
                Out.SetNum(N);
                for (int32 i=0;i<N;++i) Out[i] = Wave[i].Find(true);
                return true;
            }
            const int32 Chosen = CollapseCell(Wave[C], Rng);
            TBitArray<> Only(false, VN); Only[Chosen]=true; Wave[C]=Only;
            const FIntVector P(C % D.X, (C / D.X) % D.Y, C / (D.X*D.Y));
            if (!Propagate(D, Wave, P, Err)) bContradiction = true;
        }
    }
    Err = FString::Printf(TEXT("WFC failed after 10 restarts. Likely unmatched socket pair — %s"), *Err);
    return false;
}

bool FForgeWFCSolver::Propagate(FIntVector D, TArray<TBitArray<>>& Wave, FIntVector Start, FString& Err) {
    static const FIntVector Off[6] = {{1,0,0},{-1,0,0},{0,1,0},{0,-1,0},{0,0,1},{0,0,-1}};
    auto Idx = [&](FIntVector P){ return P.X + P.Y*D.X + P.Z*D.X*D.Y; };
    TQueue<FIntVector> Q; Q.Enqueue(Start);
    FIntVector P;
    while (Q.Dequeue(P)) {
        const TBitArray<>& Src = Wave[Idx(P)];
        for (int32 F=0; F<6; ++F) {
            const FIntVector NPos = P + Off[F];
            if (NPos.X<0||NPos.Y<0||NPos.Z<0||NPos.X>=D.X||NPos.Y>=D.Y||NPos.Z>=D.Z) continue;
            TBitArray<>& Nb = Wave[Idx(NPos)];
            bool bChanged = false;
            for (int32 V=0; V<Nb.Num(); ++V) {
                if (!Nb[V]) continue;
                bool bSupported = false;
                for (int32 S=0; S<Src.Num() && !bSupported; ++S)
                    if (Src[S] && FForgeWFCSolver::Compatible(Variants[V], Variants[S], (EForgeDir)F))
                        bSupported = true;
                if (!bSupported) { Nb[V]=false; bChanged=true; }
            }
            if (Nb.CountSetBits()==0) {
                Err = FString::Printf(TEXT("contradiction at (%d,%d,%d) face %d"), NPos.X,NPos.Y,NPos.Z,F);
                return false;
            }
            if (bChanged) Q.Enqueue(NPos);
        }
    }
    return true;
}
// PickMinEntropyCell: linear scan, entropy = sum(w)*log(sum(w)) - sum(w*log w); skip 1-bit cells; -1 if none.
// CollapseCell: weighted roulette over set bits using Variants[V].Weight.
// (Both are 15-line functions; implement exactly as described — no cleverness.)
```

Grid snap (`UForgeGridSnap`, an editor-mode helper): on placement tick, `Snapped = FMath::GridSnap(Location, TileSet->CellSize)`; before confirming placement, run `Compatible()` against the 6 occupied neighbors from the level's `TMap<FIntVector,int32> OccupiedCells`; incompatible faces render a red wireframe box on the offending face, compatible placement renders green. That map is owned by `AForgeTileVolume`, the actor that also owns the HISM components and calls the solver for its box when the user clicks **Auto-Fill** in its details panel.

### 6.2 Damage → deform → fracture — `ForgeBreakable`

```cpp
// Plugins/ForgeDestruction/Source/ForgeDestruction/Public/ForgeBreakable.h
#pragma once
#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "GeometryCollection/GeometryCollectionComponent.h"
#include "ForgeBreakable.generated.h"

UENUM(BlueprintType) enum class EForgeMaterialPreset : uint8 { Glass, Wood, Stone, Metal };

UCLASS(ClassGroup=(Forge), meta=(BlueprintSpawnableComponent))
class FORGEDESTRUCTION_API UForgeBreakable : public UActorComponent {
    GENERATED_BODY()
public:
    UPROPERTY(EditAnywhere, BlueprintReadWrite) EForgeMaterialPreset Preset = EForgeMaterialPreset::Stone;
    UPROPERTY(EditAnywhere) float DebrisLifetime = 8.f;

    virtual void BeginPlay() override {
        Super::BeginPlay();
        GC = GetOwner()->FindComponentByClass<UGeometryCollectionComponent>();
        if (!GC) { UE_LOG(LogTemp, Error, TEXT("[Forge] %s: no GeometryCollection — run 'Make Breakable' first."), *GetOwner()->GetName()); return; }
        ApplyPreset();
        GC->SetNotifyBreaks(true);
        GC->OnChaosBreakEvent.AddDynamic(this, &UForgeBreakable::OnBreak);
        GC->SetObjectType(EObjectStateTypeEnum::Chaos_Object_Kinematic); // inert until hit
        GetOwner()->OnTakeAnyDamage.AddDynamic(this, &UForgeBreakable::OnDamage);
    }

    UFUNCTION() void OnDamage(AActor* Damaged, float Damage, const UDamageType*, AController*, AActor* Causer) {
        if (!GC) return;
        Accumulated += Damage;
        if (Accumulated >= Thresholds[0]) {
            GC->SetObjectType(EObjectStateTypeEnum::Chaos_Object_Dynamic); // wake: "deform" phase
            const FVector From = Causer ? Causer->GetActorLocation() : GetOwner()->GetActorLocation();
            GC->ApplyExternalStrain(0, From, 200.f, 1, 1.f, Accumulated);  // drive Chaos strain model
        }
    }

    UFUNCTION() void OnBreak(const FChaosBreakEvent& E) {
        // Shards: fade & clean up to cap sim cost.
        FTimerHandle H;
        GetWorld()->GetTimerManager().SetTimer(H, [this]{ if (GC) GC->CrumbleActiveClusters(); }, DebrisLifetime, false);
    }
private:
    UPROPERTY() TObjectPtr<UGeometryCollectionComponent> GC;
    float Accumulated = 0.f;
    float Thresholds[2] = {5000.f, 800.f};
    void ApplyPreset() {
        switch (Preset) {
            case EForgeMaterialPreset::Glass: Thresholds[0]=50;    Thresholds[1]=10;   break;
            case EForgeMaterialPreset::Wood:  Thresholds[0]=500;   Thresholds[1]=100;  break;
            case EForgeMaterialPreset::Stone: Thresholds[0]=5000;  Thresholds[1]=800;  break;
            case EForgeMaterialPreset::Metal: Thresholds[0]=50000; Thresholds[1]=9000; break;
        }
        GC->SetPerLevelDamageThreshold(true);
        GC->SetDamageThreshold({Thresholds[0], Thresholds[1]});
    }
};
```

The **editor-side** "Make Breakable" action (`ForgeDestructionEditor` module, a `UToolMenus` context entry) performs §5 steps 2–4: `FGeometryCollectionConversion::AppendStaticMesh(...)` into a new `UGeometryCollection` asset, then `FFractureEngineFracturing::VoronoiFracture(...)` with the volume-scaled site count, then swaps the actor. Module deps: `GeometryCollectionEngine`, `FractureEngine`, `Chaos`, `ChaosSolverEngine` (editor module adds `GeometryCollectionEditor`, `ToolMenus`, `UnrealEd`).

### 6.3 Visual node logic connector — data schema

```json
// Plugins/ForgeTutor/Resources/Tutorials/first_level.json
{
  "graphId": "tut.first_level",
  "version": 1,
  "entry": "n_welcome",
  "nodes": [
    { "id": "n_welcome",  "type": "info",
      "title": "Build your first room",
      "body": "Drag the Demo tile set into the viewport.",
      "anchor": { "panel": "ContentBrowser", "highlight": "TileSets/Demo" },
      "pins": { "out": ["p_w_done"] } },
    { "id": "n_place",    "type": "await_action",
      "title": "Place a floor tile",
      "listen": { "event": "Forge.TilePlaced", "match": { "tileSet": "Demo" }, "count": 1 },
      "timeoutSec": 120, "onTimeout": "n_hint_place",
      "pins": { "in": ["p_p_in"], "out": ["p_p_done"] } },
    { "id": "n_hint_place","type": "hint",
      "body": "Click Auto-Fill on the Tile Volume to place a whole room at once.",
      "pins": { "in": ["p_h_in"], "out": ["p_h_out"] } },
    { "id": "n_branch",   "type": "branch",
      "condition": { "telemetry": "Forge.TilesPlaced", "op": ">=", "value": 10 },
      "pins": { "in": ["p_b_in"], "outTrue": ["p_b_t"], "outFalse": ["p_b_f"] } },
    { "id": "n_play",     "type": "await_action",
      "title": "Press Play", "listen": { "event": "Editor.PIEStarted" },
      "pins": { "in": ["p_pl_in"], "out": [] } }
  ],
  "edges": [
    { "from": "p_w_done", "to": "p_p_in" },
    { "from": "p_p_done", "to": "p_b_in" },
    { "from": "p_b_t",    "to": "p_pl_in" },
    { "from": "p_b_f",    "to": "p_h_in" },
    { "from": "p_h_out",  "to": "p_p_in" }
  ]
}
```

Runtime (`UForgeTutorGraph`): loads JSON → `TMap<FString,FTutNode>`; a single `FForgeTelemetry` bus (editor subsystem) broadcasts named events — `ForgeTiling` fires `Forge.TilePlaced`, PIE delegates fire `Editor.PIEStarted`, etc. Node types: `info` (advance on Next click), `await_action` (advance when listened event matches), `branch` (evaluate telemetry counter), `hint` (toast, auto-advance). The overlay is a UMG widget drawn in an editor tab: nodes as rounded cards, edges as spline connectors, current node pulsing. Non-goal: a full graph *editor* — Phase 1 tutorials are hand-authored JSON.

---

## 7. Build & Deploy Guide (sequential, no gaps)

```
1. Complete §2 installs; baseline Blank C++ project builds and opens.        [gate]
2. Copy Plugins\ForgeTiling, ForgeDestruction, ForgeTutor into C:\Dev\Forge\Plugins\.
3. Right-click Forge.uproject → "Generate Visual Studio project files".
4. Open Forge.sln → configuration "Development Editor | Win64" → Build (Ctrl+Shift+B).
   Expected: "Build: 4 succeeded". Any error → wargames/07-bugs.md Sweep 1/2 counters.
5. Launch editor (F5). Edit→Plugins: confirm three Forge plugins Enabled. Restart if prompted.
6. Content: create Content\TileSets\Demo — import 8 modular meshes (any kit-bash set with
   shared 200uu footprint; UE "Starter Content" architecture meshes work after the socket
   dialog). Tag sockets per §4 convention.
7. Place AForgeTileVolume in Templates\FPS\Map_Start, size 20×20×3, assign Demo set,
   click Auto-Fill. Expected: seamless room, no gaps, variation across floors.
8. Right-click any wall section → Forge: Make Breakable → preset Stone.
9. Press Play (PIE). Shoot/hit the wall (FPS template projectile does 1000 dmg ×6 hits).
   Expected: wall wakes, fractures into chunks, shards clean up after 8s.
10. Window → Forge Tutor → load first_level.json. Expected: overlay tracks steps 1–5 live.
11. Ship a standalone build: Platforms → Windows → Package Project → C:\Dev\ForgeBuild.
    Expected: Forge.exe launches Map_Start in the packaged game, tiling + destruction intact
    (Tutor plugin is Editor-only and correctly absent from the package).
```

**Definition of done for Phase 1:** steps 7, 9, 10, 11 all pass on a clean machine following only this document.

**Phase 2 pointers (one line each):** socket auto-inference = mesh boundary-loop hashing at import; true deformation = Chaos Flesh; tutorial graph *editor* = reuse UE's `UEdGraph` framework.
