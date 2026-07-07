# PROGRESS — Game Design Forge

> Any agent: read `README-HANDOVER.md` first, then this file, then execute `wargames/07-bugs.md`. Update this file after every sweep. Never delete rows — strike through and append.

## Mission state
| Item | Status | Notes |
|---|---|---|
| Blueprint v2 (`GAME-DESIGN-FORGE-BLUEPRINT.md`) | DONE (planning) | Written by planner; C++ is spec-written, NOT build-verified — wargame 07 exists to close that gap |
| Wargame 07 (`wargames/07-bugs.md`) | DONE (planning) | 3 sweeps, Move 0, F1–F3 forks, A1–A7 aborts, V1–V8 verification |
| Move 0 — baseline UE project green | NOT STARTED | |
| Recon R1–R8 | NOT STARTED | Record each result here as `R# = <finding> → route taken` |
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
_(anything that could not be machine-verified — see Abort A6)_
