# PROGRESS — Game Design Forge

> Any agent: read `README-HANDOVER.md` first, then this file, then execute `wargames/07-bugs.md`. Update this file after every sweep. Never delete rows — strike through and append.

## Mission state
| Item | Status | Notes |
|---|---|---|
| Blueprint v2 (`GAME-DESIGN-FORGE-BLUEPRINT.md`) | DONE (planning) | Written by planner; C++ is spec-written, NOT build-verified — wargame 07 exists to close that gap |
| Wargame 07 (`wargames/07-bugs.md`) | DONE (planning) | 3 sweeps, Move 0, F1–F3 forks, A1–A7 aborts, V1–V8 verification |
| Move 0 — baseline UE project green | NOT STARTED | |
| Recon R1–R8 | MANUAL-PENDING | Attempted 2026-07-07: environment is Linux (`uname`: `Linux ... x86_64 GNU/Linux`), no `cl.exe`, no Epic Games install path — confirmed absent, not assumed. R1–R8 all require reading UE engine headers / VS toolchain on a real Windows box. Needs: a Windows 10/11 machine, VS2022 (MSVC v143 + Win10 SDK 10.0.22621+), UE 5.4.x installed, ≥120GB disk / ≥16GB RAM. Re-run recon there — do not proceed to Move 0/Sweep 1 until this row is filled with real findings. |
| Sweep 1 — ForgeTiling | NOT STARTED | |
| Sweep 2 — ForgeDestruction | NOT STARTED | |
| Sweep 3 — ForgeTutor / templates / package | NOT STARTED | |
| V1–V8 verification | NOT STARTED | Record pass/fail + confidence per V |
| Scoring pass (07 §AFTER THE RUN) | NOT STARTED | Deltas → front matter of wargames/08 |
| Cold re-run (V1–V8, ≥3 days later) | NOT STARTED | |

## Locked decisions (terrain, not targets)
Restart-not-backtrack WFC; kinematic-until-damaged breakables; fracture-only "deformation"; manual socket dialog (no inference); preset damage numbers; card-list tutor UI; ForgeTutor editor-only. Full rationale: wargame 07 "LOOKS BROKEN BUT ISN'T" table.

## Flags (wanted changes that are OUT of bounds this mission)
_(executor appends here instead of committing them)_

## MANUAL-PENDING
- **Recon R1–R8** (2026-07-07): tried in this session's sandbox — Linux, no Windows/UE5/VS toolchain present, confirmed via `uname` + path checks (see repo history). Cannot be run here at all. Needs a real Windows machine per the row above. This is Abort A6 territory: nothing below is verified until this unblocks.
