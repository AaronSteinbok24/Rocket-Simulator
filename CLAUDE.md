# Rocket Simulator: instructions for Claude Code

A MATLAB (R2025b) flight simulator that reads a rocket designed in OpenRocket (`.ork`) and simulates it with aerodynamics built for **supersonic** flight, where OpenRocket is unreliable above about Mach 0.8. General-purpose tool and a portfolio/learning project by Aaron Steinbok. It is not tied to any competition.

## The source of truth

**`docs/PROJECT_SPEC.md` is the authoritative plan.** Before working on any module, read the relevant section of the spec. Do not load the whole file every time; use this map:

| Working on | Read |
|------------|------|
| Goals, scope, what is in or out | Sections 1-2 |
| Inputs, outputs, run configuration | Section 3 |
| Structure, data structures, repo layout | Section 4 |
| Units, frames, coding rules | Section 5 |
| Atmosphere, gravity, wind | Section 6 |
| Motors, mass, CG | Section 7 |
| Drag, normal force, center of pressure | Section 8 |
| Equations of motion | Section 9 |
| Solver, tolerances | Section 10 |
| Validation | Section 11 |
| Tests | Section 12 |

If the spec and the code disagree, or a decision seems wrong, **say so and ask** rather than silently choosing. Do not change decisions recorded in the spec without asking Aaron.

## Current status

Milestone M0 (setup) is done. The repo contains the spec and this file. Source code is built stage by stage per the milestones in Section 2.3, starting with M1 (1-DOF sim core validated against OpenRocket).

## Hard rules

1. **SI units only**, everywhere. Angles in radians internally.
2. **Never invent numerical constants or equations.** Anything marked `[VERIFY]` in the spec is not settled. Implement it only after Aaron has checked it against the cited source, and then record the result in the Verification log (spec Section 14.1). If you need a constant that is not in the spec, say so and propose a source.
3. **Every physics function** cites its source in the header comment and has units for all inputs and outputs.
4. **Write tests alongside code.** Every module gets a test in `tests/` (MATLAB function-based unit tests, named `test_<module>.m`). Run the tests before every commit. Use the MATLAB MCP server (`run_matlab_test_file`, `check_matlab_code`) to run code and tests.
5. **Plain functions and structs**, no classes, no global variables.
6. **Stage the work.** Establish and validate fundamentals before adding complexity. Each stage has a checkpoint in the spec.

## Working style

- Aaron wants to understand the *why* behind each piece. Explain reasoning in plain terms alongside changes, step by step, rather than just producing code.
- Keep changes small and easy to review. Show what you plan to do before big changes.
- Commit at each working stage with a descriptive message. Never force-push. Do not push without Aaron's go-ahead.
- Follow the MATLAB coding guidelines available from the MATLAB MCP server (`guidelines://coding`).

## Environment

- Windows, PowerShell. Repo at `C:\dev\rocketsim`.
- MATLAB R2025b, connected through the MATLAB MCP server (registered as `matlab`, working folder is the repo).
- Reference tools for validation: OpenRocket (subsonic only) and RASAero II.
