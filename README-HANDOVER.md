# HANDOVER — Game Design Forge (read me first, then don't ask anyone anything)

**What this is:** planning artifacts for building "Game Design Forge" — a code-free game-making editor — as an Unreal Engine 5 plugin suite. The original planner is gone. Everything you need is in four files.

## The four files and the reading order
1. **This file** — routing.
2. **`PROGRESS.md`** — current mission state, locked decisions, flags. Tells you where the last agent stopped.
3. **`GAME-DESIGN-FORGE-BLUEPRINT.md`** — WHAT to build: stack, install, directory tree, algorithms, the actual C++ boilerplate, build guide, Definition-of-Done. Self-contained; a junior dev can follow it top to bottom.
4. **`wargames/07-bugs.md`** — HOW to survive building it: Move 0, recon checks R1–R8, three sweeps with expected observation / failure / cause / counter per move, forks F1–F3 with triggers, "looks broken but isn't" table, aborts A1–A7, verification runs V1–V8, scoring procedure.

## If you are the executor (Claude Code, any model)
Run `wargames/07-bugs.md` end to end. It is written so you never need to ask a question: ambiguity resolves into a fork trigger or a RECON check. Start at MOVE 0. Requirements: Windows, ~120 GB disk, 16 GB RAM, admin rights to install VS 2022 + UE 5.4.

## If you are a planner picking up after the run
Do the wargame's "AFTER THE RUN — SCORING" section, then author `wargames/08-*.md` for Phase 2 (socket auto-inference, Chaos Flesh deformation, tutorial graph editor — one-liners at blueprint end).

## Hard rules that survive every agent handoff
- Blueprint C++ was **never compiled by the planner** — verify on first touch, per the wargame's standing rules.
- Never edit engine source (Abort A5). Never retune design constants (see PROGRESS.md locked decisions) — flag instead.
- One branch per wargame, one commit per fix, PROGRESS.md per sweep, one relay line in `log.md` per mission.
- Nothing is "verified" unless a V-run passed on a real machine. Otherwise it's MANUAL-PENDING, and you say so.

## Known open risks (honest ledger)
- Chaos programmatic-fracture API surface (R3–R5) is the likeliest fork-fire (F3 Route B: fall back to built-in Fracture Mode = two clicks instead of one; DoD unaffected).
- `FString Sockets[6]` in a USTRUCT is the single most likely compile break (counter pre-written, M1.3).
- Yaw ring calibration is a planned 30-min empirical step, not a defect (M1.4 f3).
