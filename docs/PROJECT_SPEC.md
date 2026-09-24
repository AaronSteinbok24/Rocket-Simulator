# Rocket Simulator: Project Specification

**Status:** Draft v0.3 (2026-09-23), sections 1-12 written
**Owner:** Aaron Steinbok
**Repository:** https://github.com/AaronSteinbok24/Rocket-Simulator
**Language / platform:** MATLAB R2025b, developed with Claude Code via the MATLAB MCP Server

> **How to read this document.** Items marked **[PROPOSED]** are my defaults and need your confirmation. Items marked **[OPEN]** are unresolved decisions. Items marked **[VERIFY]** are equations or constants recalled from a standard reference that must be checked against the source before implementation. Everything else reflects a decision you made.

---

## 1. Purpose and goals

### 1.1 Background and problem

OpenRocket is the standard free design tool for hobby and high-power rockets, but its aerodynamic model is built primarily for subsonic flight. Above roughly Mach 0.8 (the transonic regime, where shock waves begin to form and drag rises sharply) and into supersonic flight, its predictions of drag and stability become unreliable. RASAero II handles supersonic flight better, but it is a closed tool. This project builds an open, understandable simulator whose aerodynamics are designed for supersonic flight from the start.

The idea came out of planning a high-altitude cluster rocket for a competition. The competition is only the inspiration. This is not a competition tool, and no single vehicle drives the requirements.

### 1.2 Primary goal

A MATLAB program that reads a rocket designed in OpenRocket (an `.ork` file), computes its aerodynamics at all speeds with emphasis on supersonic flight, and simulates the flight with as much accuracy as the underlying physics allows.

### 1.3 Secondary goals

- **Portfolio / learning project.** Every model should be understood and documented, not treated as a black box. Each equation cites its source.
- **General purpose.** It should work on any conventional rocket, not one hard-coded vehicle.
- **Subsonic support.** It should also simulate subsonic flight so the tool is comprehensive. This is low priority.

### 1.4 Non-goals

Explicitly out of scope for the semester (see Section 2 for when they might return):

- Graphical user interface (scripts and plots only)
- Multi-stage rockets and strap-on boosters
- Fin flutter and aerodynamic heating analysis
- Altitude-dependent wind profiles
- Real-time or onboard flight-computer use
- Any dependence on a specific competition or rocket

### 1.5 Success criteria

Targets confirmed 2026-09-23. They give "accurate" a measurable meaning. **A small but consistent bias against the reference is acceptable**: it is characterized (S8) and treated as a calibration target rather than a failure.

| # | Criterion | Target |
|---|-----------|--------|
| S1 | Subsonic apogee vs OpenRocket on reference rockets | within 3% |
| S2 | Supersonic Cd(Mach) vs RASAero II, Mach 1.3-3 | within 10% |
| S3 | Supersonic apogee vs RASAero II | within 5% |
| S4 | Transonic region (Mach 0.8-1.3) | difference documented and explained, no fixed target (see 8.6) |
| S5 | Stability margin vs Mach | same sign and trend as RASAero II, values documented |
| S6 | Reproducibility | one command takes an `.ork` plus motor files to results |
| S7 | Code quality | every module has unit tests; every equation cites a source |
| S8 | Bias characterization | signed error vs Mach reported for Cd, apogee, and stability margin; a consistent bias is documented and, where possible, corrected by calibration |

Matching RASAero II means "plausible", not "proven". The final check is real flight data, which is a later addition.

---

## 2. Scope and roadmap

### 2.1 Tiers

**Tier 1: this semester (through December 2026).** Single-stage rockets and motor clusters.
- `.ork` parser: nose cones, body tubes, transitions, trapezoidal fins, motors, masses
- Standard atmosphere, constant wind, launch-site elevation, temperature offset
- Propulsion: thrust curves, propellant mass flow, cluster ignition timing
- Aerodynamics valid subsonic through supersonic: Cd, normal force slope, center of pressure vs Mach
- 1-DOF, then 3-DOF flight to apogee
- Validation against RASAero II and OpenRocket
- Outputs: apogee, max Mach and velocity, stability margin vs Mach
- Unit tests and documentation

**Tier 2: stretch.**
- 6-DOF rigid-body dynamics (attitude, angle-of-attack-dependent aero)
- Recovery events and descent (needed for landing dispersion)

**Tier 3: later.**
- Multi-stage rockets and strap-on boosters
- Fin flutter analysis and aerodynamic heating
- Altitude wind profile, Monte Carlo dispersion
- Additional fin shapes, GUI (MATLAB App Designer)
- Comparison against real altimeter flight data

### 2.2 Why staging and boosters are Tier 3

Clusters are cheap because they mostly sum thrust curves. Staging and strap-ons require separation events, sudden mass and drag changes, and separate aero for each configuration. Each is a project on its own.

### 2.3 Milestones (Tier 1)

A planning estimate of about 12 weeks, confirmed as workable; speed up or slow down week to week.

| # | Milestone | Rough weeks | Done when |
|---|-----------|-------------|-----------|
| M0 | Environment setup | done | Claude Code, MATLAB link, GitHub repo working |
| M1 | 1-DOF sim core | 1 | matches OpenRocket subsonic with constant Cd |
| M2 | `.ork` parser | 2-3 | reproduces OpenRocket's geometry, mass, and CG |
| M3 | Subsonic aero module | 3-5 | Cd and CP match OpenRocket subsonic |
| M4 | Supersonic aero module | 5-8 | Cd(M) within target of RASAero II above Mach 1.3 |
| M5 | Transonic blend and 3-DOF | 8-10 | continuous Cd through Mach 1; wind and launch angle work |
| M6 | Validation report and cleanup | 11-12 | success criteria S1-S8 evaluated and documented |

---

## 3. User workflow and inputs

### 3.1 Workflow

1. Design the rocket in OpenRocket and save the `.ork` file.
2. Download the motor thrust curves (`.eng` files, e.g. from ThrustCurve.org). The `.ork` file stores only the motor designation, not the thrust curve.
3. Fill in a run configuration script (Section 3.2).
4. Run one command. Results are returned as a struct and plotted.

### 3.2 Run configuration

| Input | Notes |
|-------|-------|
| `.ork` file path | the rocket geometry and masses |
| motor files and ignition delays | one entry per motor in the cluster |
| launch-site elevation | metres above sea level |
| temperature offset | kelvin, relative to standard atmosphere |
| wind vector | constant, east-north components in m/s |
| rail length and launch angle | needed from 3-DOF onward |
| fidelity level | 1-DOF, 3-DOF (later 6-DOF) |

### 3.3 Outputs

Time histories of altitude, velocity, Mach number, thrust, mass, center of gravity, center of pressure, and stability margin, plus summary values: apogee, max Mach, max velocity, rail departure velocity, minimum stability margin and where it occurs.

---

## 4. Architecture

### 4.1 Pipeline

```
 .ork file ──> parse_ork ──┐
                           ├──> rocket struct ──> aero module ──┐
 .eng files ─> parse_eng ──┘                                    │
                                        propulsion module ──────┼──> dynamics (ODE) ──> results
                       atmosphere + wind ───────────────────────┘
```

Each block is a separate module with a defined interface, so it can be tested alone and swapped later. The aero module in particular can be improved without touching the rest.

### 4.2 Repository layout

```
Rocket-Simulator/
├── README.md
├── CLAUDE.md               instructions for Claude Code
├── docs/PROJECT_SPEC.md    this document
├── src/
│   ├── ork/                parse_ork.m, parse_eng.m
│   ├── atmosphere/         isa_atmosphere.m, wind model
│   ├── aero/               drag and stability components
│   ├── propulsion/         thrust_total.m, mass and CG vs time
│   ├── dynamics/           equations of motion, integrators
│   └── utils/
├── tests/                  MATLAB unit tests
├── validation/             comparisons against RASAero II and OpenRocket
├── examples/               sample .ork files and run scripts
└── data/motors/            .eng thrust curves
```

Code style is plain functions and structs rather than classes, for readability and easy testing.

### 4.3 Core data structures (all SI)

**`rocket` struct** (produced by `parse_ork`):
- `name`, `Dref_m` (reference diameter), `Sref_m2` (reference area), `length_m`
- `nose`: shape, length, base diameter, shape parameter
- `bodyTubes(i)`, `transitions(i)`: dimensions and axial position
- `fins`: count, root chord, tip chord, span, sweep, thickness, axial position
- `finish`: surface roughness (used in skin friction)
- `dryMass_kg`, `dryCG_m`, and inertias when available
- `motors(i)`: mount position and designation

**`motor` struct** (produced by `parse_eng`): `name`, `t`, `F`, `cumI`, `propMass_kg`, `casingMass_kg`, `burnTime_s`, `totalImpulse_Ns`, plus `delay_s` set at run time.

**`results` struct**: time vector, state history, and the derived quantities in Section 3.3.

---

## 5. Conventions

- **Units:** SI only, everywhere, internally and in outputs. Angles in radians internally.
- **Axial positions:** measured from the nose tip, positive aft (same as OpenRocket after component offsets are resolved).
- **Reference geometry:** reference diameter is the maximum body diameter; reference area is the circle on that diameter.
- **Stability margin:** (x_cp - x_cg) / D_ref, in calibers, positive when the center of pressure is aft of the center of gravity.
- **Trajectory frame:** launch-centered East-North-Up, z up.
- **Body frame (6-DOF, later):** x along the rocket axis pointing forward, right-handed.
- **Coding standards:** header comment on every function (purpose, inputs with units, outputs with units, source), no global variables, every model equation cites its reference. The MATLAB MCP Server provides official MATLAB coding guidelines that Claude Code can consult.
- **Testing:** every function gets at least one test. Physics functions are checked against hand calculations or published values.
- **Verification:** every equation or constant marked [VERIFY] is checked against its cited source by Aaron before it is implemented, and the result is recorded in the Verification log (Section 14.1). Nothing marked [VERIFY] is treated as settled until logged.
- **Version control:** commit at each working stage with a descriptive message.

---

## 6. Environment models

> **Convention for this and later sections.** Equations that I wrote from memory of a standard reference are marked **[VERIFY]**. Before implementing one, check the constants and form against the cited source and record any correction here. Treat this document as the plan, and the source as the authority.

### 6.1 Atmosphere (ISA with temperature offset)

**Physics.** Pressure is set by the weight of the air above (hydrostatic balance), and density follows from the ideal gas law:

```
dP/dh = -rho * g0 = -P * g0 / (R * T(h))        rho = P / (R * T)
```

Temperature varies linearly with height inside each layer at a lapse rate L, which gives closed-form solutions per layer:

```
L != 0 :  T = Tb + L (h - hb),   P = Pb (Tb / T)^(g0 / (L R))
L == 0 :  T = Tb,                P = Pb exp( -g0 (h - hb) / (R Tb) )
```

| Layer | Base altitude (m) | Lapse rate (K/m) | Base temp (K) |
|-------|-------------------|------------------|---------------|
| Troposphere | 0 | -0.0065 | 288.15 |
| Tropopause | 11,000 | 0 | 216.65 |
| Stratosphere 1 | 20,000 | +0.001 | 216.65 |
| Stratosphere 2 | 32,000 | +0.0028 | 228.65 |
| Stratopause | 47,000 | 0 | 270.65 |

Constants: `g0 = 9.80665 m/s^2`, `R = 287.05287 J/(kg K)`, sea-level pressure `P0 = 101325 Pa`. Valid 0-47 km; inputs outside are clamped.

**Temperature offset (dT).** A hot or cold day shifts the whole column. The correct treatment applies dT to every layer's base temperature and then **recomputes the layer base pressures by chaining upward from sea level**. Adding dT only at the end, while keeping standard pressures, is wrong: a warmer column has a larger scale height, so pressure at a given altitude changes too. The error grows with altitude and is not negligible at rocket apogees. *(The current `isa_atmosphere.m` has no dT input; adding it is a planned change.)*

**Geopotential vs geometric altitude [PROPOSED].** The ISA tables are defined in geopotential altitude `H = Re h / (Re + h)`. Convert before evaluating the layers. It is one line and removes a small but systematic error.

**Site elevation.** Rocket altitude in the state is above ground level (AGL). The atmosphere is queried at `h_site + h_AGL`. Sea-level pressure defaults to 101325 Pa with an optional override.

### 6.2 Air properties

```
speed of sound     a  = sqrt(gamma R T),         gamma = 1.4
dynamic viscosity  mu = mu0 (T/T0)^1.5 (T0 + S)/(T + S)     (Sutherland)
                       mu0 = 1.716e-5 Pa s, T0 = 273.15 K, S = 110.4 K      [VERIFY]
Reynolds number    Re = rho V L / mu
```

Sources: White, *Fluid Mechanics* (Sutherland's law); Anderson, *Modern Compressible Flow* (speed of sound). Constant gamma is adequate to about Mach 4-5.

### 6.3 Gravity and reference frame

```
g(h) = g0 (Re / (Re + h))^2,      Re = 6,371,000 m
```

The trajectory frame is a flat launch-centered East-North-Up frame. Earth's rotation (Coriolis) and curvature are neglected, which is valid for flights of tens of kilometres of range. Both are candidates for a later refinement only if validation shows a need.

### 6.4 Wind

Version 1 uses a constant wind vector, `w = [wE, wN, 0]` in ENU, with no vertical component and no altitude variation. **Convention: `w` is the direction the air moves toward.** Meteorological wind direction is the direction the wind comes *from*, so any input in that form must be converted.

All aerodynamics use the airspeed relative to the air:

```
v_rel = v_rocket - w,       V = |v_rel|,       Mach = V / a,       q = 0.5 rho V^2
```

*Why this matters:* drag and Mach depend on airspeed, not ground speed. A headwind raises drag at the same ground speed. Tier 3 replaces the constant wind with an altitude profile.

---

## 7. Propulsion

### 7.1 Thrust

Each motor provides a thrust curve `F(t)` from an `.eng` file. Thrust is linearly interpolated between the tabulated points, is zero before ignition, and is zero after burnout. Total thrust is the sum over motors, each with its own clock (`t - delay`), so staggered cluster ignition needs no special logic. The parser prepends `(0, 0)` if the file starts after `t = 0`.

### 7.2 Mass flow

Propellant burns in proportion to thrust:

```
mdot_i(t) = m_prop,i * F_i(t) / I_total,i      (equivalent to a constant Isp)
```

Remaining mass is `m_i(t) = m_casing + m_prop (1 - I(t)/I_total)`, with `I(t)` the cumulative impulse. Real specific impulse varies slightly through a burn, which is neglected.

**Why thrust is just an applied force.** The rocket equation `m dv/dt = T - D - mg` is correct with the measured thrust `T` because thrust already includes the momentum of the exhaust. No extra `mdot * v` term appears when `v` is the rocket's own velocity.

### 7.3 Thrust vs ambient pressure

`.eng` curves are measured at sea-level static conditions. In flight, lower ambient pressure raises thrust by about `(P0 - Pa) * A_exit`. The nozzle exit area is not in the `.eng` file, and most of the burn happens at low altitude, so this is **neglected in version 1**. It can matter for long burns at altitude and is a candidate refinement.

### 7.4 Mass properties

- **CG:** mass-weighted position of the dry rocket plus each motor's casing and remaining propellant. Version 1 places each motor's mass at a fixed point at the centre of the motor **[PROPOSED]**. The true propellant CG shifts as grains burn. This affects stability margin during the burn.
- **Inertia (needed from 6-DOF):** each motor is a cylinder of mass `m`, radius `r`, length `L`, about its own CG: `Ixx = 0.5 m r^2`, `Iyy = Izz = m (3 r^2 + L^2) / 12`, then shifted to the rocket CG with the parallel-axis theorem.
- **Dry mass and CG** come from the `.ork` parser and must be validated against OpenRocket's reported values (see Section 13).

### 7.5 Motor data checks

The parser checks that the time vector is strictly increasing, that thrust is non-negative, and that integrated impulse is consistent with the motor's published class. Motors are matched to the `.ork` designation by name.

---

## 8. Aerodynamics

This is the core of the project and the reason it exists. The target is accuracy from subsonic through about Mach 4, with the weakest region (transonic) treated honestly.

### 8.1 Definitions and structure

```
q = 0.5 rho V^2
CD = D / (q Sref)        CN = N / (q Sref)        CN_alpha = d(CN)/d(alpha) per radian
x_cp = axial position (from nose tip) where the normal force acts
```

Regimes: **subsonic** M < 0.8, **transonic** 0.8 <= M <= 1.2, **supersonic** 1.2 < M <= 4. Hypersonic flight (M > 4) is out of scope **[PROPOSED]**.

At zero angle of attack (all of 1-DOF, and 3-DOF with the quasi-steady assumption) drag is built up from components, all referenced to `Sref`:

```
CD0 = CD_friction + CD_nose/body wave + CD_fins + CD_interference + CD_base
```

| Component | Method (planned) | Main sources | Confidence |
|-----------|------------------|--------------|------------|
| Skin friction | flat-plate turbulent + compressibility + roughness | Schlichting; White; Van Driest | good |
| Nose/body wave drag | tangent-cone / modified Newtonian | Anderson; NACA Rep. 1135 | good (moderate at low supersonic) |
| Fin drag | Ackeret linear theory + sweep + blunt LE | Anderson; Hoerner | moderate |
| Base drag | empirical vs Mach, power-on/off | Hoerner | moderate |
| Interference | empirical factor | Hoerner | low; calibrate |
| Transonic region | anchor-and-blend, area rule | Whitcomb; Ashley & Landahl | lowest |
| Normal force / CP | Barrowman + linearized supersonic | Barrowman; Anderson | good subsonic, moderate above |

### 8.2 Skin friction

**Physics.** Air sticks to the surface, and the resulting shear on the wetted area is friction drag. It dominates subsonic drag and remains large supersonically.

Boundary-layer state depends on Reynolds number. Rocket flights reach very high Re, so the baseline assumes **fully turbulent flow from the nose**, appropriate for a real (slightly rough) surface. Laminar and transition models can be added later.

```
turbulent, incompressible:   Cf = 0.455 / (log10 Re)^2.58                       [VERIFY]  (Prandtl-Schlichting)
laminar (reference):         Cf = 1.328 / sqrt(Re)                              (Blasius)
fully rough limit:           Cf = (1.89 + 1.62 log10(x/k))^-2.5                 [VERIFY]  k = roughness height
```

The rough limit is the floor: use `max(smooth, rough)` at each Reynolds number. Compressibility reduces Cf with Mach; the target method is Van Driest II, with a simpler empirical factor first **[VERIFY]**. Body friction drag is `Cf * S_wet / Sref` times a fineness-ratio form factor that accounts for thickness pressure drag (Hoerner) **[VERIFY constants]**. Fin friction counts both faces of every panel with a mean-chord Reynolds number.

Friction depends on Reynolds number, which changes with altitude and speed, so it is evaluated during the flight rather than tabulated (see 8.9).

### 8.3 Nose and body wave drag

**Physics.** Supersonic flow past the nose forms a shock. Passing through it costs stagnation pressure, which shows up as **wave drag**. It rises sharply near Mach 1, then falls as Mach increases because the shock lies closer to the body and the pressure rise is weaker relative to dynamic pressure.

Planned methods, in order:

- **Baseline: tangent-cone method.** For a cone of half-angle `delta`, the surface pressure follows from the exact **Taylor-Maccoll** solution (numerical ODE, or tabulated). A curved nose (ogive, parabolic, Haack) is approximated as a stack of cones whose local half-angle equals the local surface slope, and the pressure is integrated over the surface.
- **Attached-shock limit.** If the nose is too blunt for the Mach number (`delta > delta_max(M)`), the shock detaches. Switch to **modified Newtonian** theory: `Cp = Cp_max sin^2(theta)`, with `Cp_max` from the stagnation pressure behind a normal shock (Rayleigh pitot).
- **Transitions:** a conical shoulder (diameter increasing) produces an oblique shock; a **boattail** (diameter decreasing) produces a Prandtl-Meyer expansion and a pressure *decrease*. A straight body tube contributes no wave drag at zero incidence.

Sources: Anderson, *Modern Compressible Flow*; Ames Research Staff, NACA Report 1135 (oblique shock and cone tables); Taylor & Maccoll (1933).

### 8.4 Fin drag

**Physics.** Fins are thin wings. Supersonically, thickness drives wave drag, and blunt leading edges add stagnation-pressure drag.

- **Thickness wave drag:** Ackeret linear theory: for thin sections, wave drag scales with `(t/c)^2 / sqrt(M^2 - 1)`. Sweep helps: a swept leading edge sees only the Mach component normal to it, `M cos(LE sweep)`. If that stays subsonic (leading edge inside the Mach cone), wave drag falls sharply.
- **Leading-edge bluntness:** a rounded or square leading edge takes a normal-shock stagnation pressure on its frontal area. Estimate with the Rayleigh pitot pressure ratio times the frontal area, scaled by `cos^2(sweep)` **[VERIFY]**.
- **Trailing edge:** a blunt trailing edge adds base drag.
- **Interference:** the fin-body junction adds drag. Start with an empirical multiplier on fin friction and **calibrate against RASAero II**.
- **`.ork` input:** OpenRocket stores fin cross-section type (square, rounded, airfoil) and thickness. The parser must read both because they change the leading- and trailing-edge terms.

Sources: Anderson (Ackeret theory); Hoerner, *Fluid-Dynamic Drag*.

### 8.5 Base drag

**Physics.** The blunt aft end leaves a low-pressure separated wake behind the rocket, giving a pressure drag on the base.

```
CD_base = K_b(M) * (A_base / Sref)
    subsonic  K_b ~ 0.12 + 0.13 M^2          [VERIFY]
    supersonic K_b ~ 0.25 / M                 [VERIFY]
```

**Power-on vs power-off.** With the motor burning, the exhaust plume fills part of the wake and base drag is much smaller. Start with the effective base area `A_base - A_nozzle_exit` (summed over all nozzles in a cluster) while thrust is nonzero, and calibrate the correction against RASAero II's power-on and power-off curves. The sim must therefore look up separate power-on and power-off drag.

### 8.6 The transonic region

Between roughly Mach 0.8 and 1.2, local shocks form on the body and fins, drag rises steeply toward a peak near Mach 1, and simple theory breaks down. **This is the least reliable region for any method short of CFD or wind tunnel data**, and OpenRocket's known weakness. Results here carry the largest error bars, and the validation report must say so.

Planned approach:

1. Compute `CD(0.8)` with the subsonic methods and `CD(1.2)` with the supersonic methods.
2. Estimate the peak from the **area rule**: near Mach 1, wave drag of a slender body depends on its axial cross-sectional area distribution `S(x)` (including fin cross-section). The von Kármán slender-body result is `D_wave/q = -(1/2pi) * integral integral S''(x) S''(xi) ln|x - xi| dx dxi` **[VERIFY]** (Ashley & Landahl).
3. Join the three anchors with a monotone smooth interpolant (PCHIP) so `CD(M)` is continuous with a continuous slope.
4. Tune the peak value against RASAero II. Agreement there is a plausibility check, not proof.

### 8.7 Normal force and center of pressure

The center of pressure sets **stability**, and its Mach dependence is a central output of the project.

**Subsonic (Barrowman method)** for small angle of attack. Sum component normal-force slopes, and the CP is their weighted average:

```
CN_alpha,total = sum_i CN_alpha,i             x_cp = sum_i (CN_alpha,i * x_i) / CN_alpha,total
nose cone:      CN_alpha = 2                   x_cp = 0.466 L (ogive), 0.5 L (cone)   [VERIFY]
transition:     CN_alpha = 2 [ (d_aft/d_ref)^2 - (d_fore/d_ref)^2 ]
                x_cp = x_fore + (L/3) [ 1 + (1 - d_fore/d_aft) / (1 - (d_fore/d_aft)^2) ]     [VERIFY]
body tube:      no lift (Barrowman)
fins:           CN_alpha = K_fb * 4 n (s/d_ref)^2 / (1 + sqrt(1 + (2 l_m / (C_r + C_t))^2))    [VERIFY]
                K_fb = 1 + r / (s + r)       (fin-body interference)
```

Here `n` is the fin count, `s` the span, `l_m` the mid-chord length, `C_r`, `C_t` the root and tip chords, `r` the body radius at the fins. Source: Barrowman & Barrowman (1966) and Barrowman (1967).

**Compressibility.** Below Mach 0.8, scale fin lift with a Prandtl-Glauert factor (`1/sqrt(1 - M^2)`). Above Mach 1, use linearized supersonic theory: 2D lift slope `4 / sqrt(M^2 - 1)` per radian with a finite-span correction. Nose and body values follow slender-body theory at low speed and need Mach-dependent correction supersonically (Missile DATCOM methods are the reference) **[VERIFY]**.

*Why the CP moves.* Fin and nose lift respond differently to Mach number, so the weighted average shifts, typically aft supersonically. Since stability margin is `(x_cp - x_cg) / D_ref`, this changes the margin exactly where the rocket flies fastest.

### 8.8 Angle-of-attack effects (from 3-DOF/6-DOF)

For small angles, induced drag adds about `CN_alpha * alpha^2` to `CD0`, and forces resolve into axial and normal components. In 3-DOF version 1 the rocket is assumed aligned with the relative wind (`alpha = 0`), so this term is zero and is documented as a limitation. Body cross-flow at larger angles (a nonlinear normal force, roughly proportional to `alpha |alpha|`) is a Tier 2 refinement.

### 8.9 Implementation strategy

- **Tabulate** the Mach-only quantities (wave drag, base drag, fin drag, `CN_alpha`, `x_cp`) on a Mach grid from 0 to 5, with finer spacing across 0.8-1.3, separately for power-on and power-off. Interpolate during the flight with PCHIP.
- **Evaluate friction during the flight** because it depends on Reynolds number (altitude and speed).
- Regenerate tables when the rocket changes.
- Output `CD(M)`, `CN_alpha(M)`, `x_cp(M)` arrays for the validation plots.

### 8.10 Sanity checks

- `CD` is positive, finite, and continuous across Mach 0.8, 1.0, and 1.2.
- Friction matches published flat-plate values for a test plate.
- Cone wave drag matches the NACA 1135 / Taylor-Maccoll tables.
- Fin-only supersonic drag matches the 2D Ackeret result for a thin section.
- Total body wave drag should not fall below the **Sears-Haack** minimum for the same length and volume, `D/q = (9 pi^2 / 2) V^2 / l^4` **[VERIFY]**, as a linear-theory sanity bound (caveats: it assumes slender pointed bodies).
- Component-by-component comparison against RASAero II for the same geometry.

---

## 9. Flight dynamics

### 9.1 1-DOF: vertical flight

State `y = [h, v]` (altitude AGL, vertical velocity).

```
dh/dt = v
dv/dt = (T(t) - D) / m(t) - g(h),     D = 0.5 rho v |v| CD(M) Sref
```

`v |v|` makes drag oppose motion automatically, on ascent and descent. **Launch-pad constraint:** while `T < m g` and the rocket is at rest, the ground holds it (`dy = 0`). Apogee is the event `v = 0` crossing downward.

### 9.2 3-DOF: point mass in three dimensions

State: position `r = [E, N, U]` and velocity `v`. All aerodynamics use `v_rel = v - w`.

```
e_v  = v_rel / |v_rel|                              (direction of the relative wind)
e_b  = e_rail while on the rail;  e_b = e_v after departure      (quasi-steady weathercocking)
m(t) dv/dt = T(t) e_b - D e_v + m g_vec,           g_vec = [0, 0, -g(h)]
```

**Rail phase.** The rocket is constrained to slide along the rail direction `e_rail = [sin(theta) sin(psi), sin(theta) cos(psi), cos(theta)]`, with `theta` the angle from vertical and `psi` the azimuth clockwise from north. Along the rail, `s'' = (T - D)/m - g cos(theta)`. The rocket stays on the pad until `T` exceeds the along-rail weight component. **Rail departure velocity** (when `s >= rail length`) is a key output for launch safety. Aerodynamics use the relative wind even on the rail.

**After the rail.** The rocket is assumed to point along the relative wind at all times. That neglects the short-period pitch oscillation, so it slightly misestimates the first moments off the rail. It captures the main effect of wind (the rocket "weathercocks" and drifts) without any rotational dynamics.

**Consistency test.** With vertical launch, zero wind, and no angle terms, 3-DOF must reproduce 1-DOF exactly.

### 9.3 6-DOF: rigid body (Tier 2)

Newton-Euler equations in the body frame, with attitude as a quaternion (avoids the singularity of Euler angles at vertical, which matters for a rocket):

```
m (dv_b/dt + omega x v_b) = F_aero + F_thrust + m g_b
I (domega/dt) + omega x (I omega) = M_aero + M_thrust
dq/dt = 0.5 * q (x) [0, omega]
```

Aerodynamic normal force and moment come from `CN(M, alpha)` and `x_cp` (moment about the CG is `CN (x_cg - x_cp)`), plus **pitch-damping** and roll terms (fin cant, roll damping). Variable mass adds a **jet-damping** moment proportional to `mdot`. Inertia changes with propellant burn (Section 7.4). Full derivation to be written when Tier 2 begins.

---

## 10. Numerical methods

- **Solver:** `ode45` (adaptive Runge-Kutta) to start. `ode113` and fixed-step RK4 are alternatives if profiling or validation suggests them.
- **Phase-based integration.** Thrust curves are piecewise data with sharp edges (ignition, burnout, ignition delays). Rather than forcing a tiny maximum step, split the flight into **phases** between known event times (ignition, burnout, rail departure) and restart the integrator at each. This is faster and more accurate than the `MaxStep` workaround used in the stage 1 code.
- **Events:** rail departure, motor ignition/burnout times, apogee (`v_z = 0` going downward), and ground impact later.
- **Tolerances:** start at `RelTol = 1e-8`, `AbsTol = 1e-8`.
- **Convergence test:** halve the tolerances and confirm apogee changes by less than 0.01%. Do this for every validation case.
- **Aero speed:** the coefficient tables from 8.9 keep the ODE right-hand side cheap.
- **Reproducibility:** fixed inputs give identical results; no randomness until Monte Carlo (Tier 3).


## 11. Validation plan

### 11.1 Approach

Validation is layered, so that when something disagrees we know *where* to look:

| Level | What is checked | Against |
|-------|-----------------|---------|
| L0 | Individual equations and functions | analytic results, published tables |
| L1 | Single aero components (friction, nose wave drag, fin drag, base drag, CP contributions) | OpenRocket's component analysis (subsonic), RASAero II (supersonic), hand calculation |
| L2 | Whole-rocket coefficients `CD(M)`, `CN_alpha(M)`, `x_cp(M)` | RASAero II, OpenRocket (subsonic range only) |
| L3 | Whole-flight results: apogee, max Mach, max velocity, burnout velocity, time to apogee | RASAero II, OpenRocket (subsonic range only) |
| L4 (later) | Real flight data | your own altimeter logs |

### 11.2 Reference tools

- **OpenRocket:** run simulations with matching settings and export the flight data. Its component analysis view reports per-component drag and center of pressure at a chosen Mach number, which is what L1 needs. Trusted only below about Mach 0.8. **[VERIFY]** what your installed version exports.
- **RASAero II:** aerodynamic analysis (drag, normal force, CP vs Mach) and a flight simulation. **[OPEN]** confirm exactly what can be exported and in what form; this is the first task of milestone M4. You already have it installed.

### 11.3 Test matrix

Three self-designed reference rockets, each in a single-motor and a cluster variant:

| Case | Target regime | Purpose |
|------|---------------|---------|
| R1 | low subsonic (Mach < 0.6) | check the core against OpenRocket, where it is trustworthy |
| R2 | about Mach 1.5 | first supersonic and transonic-crossing check |
| R3 | about Mach 2.5 | higher supersonic; held out for validation (see 11.6) |

Conditions per rocket: standard sea level with no wind (baseline); a high-elevation hot day (tests the atmosphere offset); and a constant-wind case (3-DOF onward). Rocket designs are created when needed.

### 11.4 Matching settings

Before comparing, make sure both tools are computing the *same problem*:

- Same geometry, dry mass, and CG (check the parsed CG against OpenRocket's reported value first)
- Same motor curves and ignition timing
- Same atmosphere, launch elevation, wind, and launch angle
- Same **reference area** and reference length for coefficients (a mismatch here looks like a drag error but is not)
- Same surface finish assumption
- Power-on vs power-off drag identified correctly on each side

### 11.5 Metrics

The signed error is `e = (mine - reference) / reference`. Report it in three Mach bins: **subsonic** (below 0.8), **transonic** (0.8-1.2), and **supersonic** (1.2-4), giving the mean (bias) and RMS for each. Also report apogee, max Mach, max velocity, and minimum stability margin. This directly supports success criterion S8: a consistent bias is characterized rather than hidden.

### 11.6 Calibration without fooling yourself

Some empirical factors (fin-body interference, plume effect on base drag, transonic peak) will need tuning against RASAero II. Rules:

1. Every calibration factor must be **physically motivated** and documented, not a free fudge factor.
2. Factors are **global**: one value for all rockets, never re-tuned per case.
3. Fit on the calibration set (R1, R2) and **evaluate on the held-out R3**. If R3 is not predicted well, the model is overfitted to the references.
4. Agreement with RASAero II means "plausible", not "proven": it is another model, not ground truth.

### 11.7 When results disagree

Work through the causes in order, cheapest first: (1) inputs (geometry, mass, CG, motor parsed correctly); (2) definitions (reference area, coefficient conventions, units); (3) which component disagrees (L1); (4) which Mach regime disagrees; (5) only then suspect the model. Record what was found in the validation report.

### 11.8 Deliverables

A validation report per major milestone: M3 (subsonic vs OpenRocket), M4 (supersonic vs RASAero II), and M6 (final, all criteria S1-S8), each with plots of `CD(M)`, `CN_alpha(M)`, `x_cp(M)`, and flight histories overlaid on the references.

### 11.9 Later: real flight data

An altimeter flight gives the first true ground truth (apogee, velocity if recorded). Known error sources to account for: actual motor performance vs the published curve, real wind, launch angle, rocket mass and CG as flown, and altimeter accuracy (especially in the transonic region, where pressure sensors are disturbed by shocks). Tier 3.

---

## 12. Testing strategy

### 12.1 Levels

- **Unit tests:** one test file per module using MATLAB's function-based unit-testing framework (`test_<module>.m` in `tests/`).
- **Analytic checks:** cases with a known exact answer (12.2).
- **Regression tests:** stored reference outputs with tolerances, so a change that shifts results is noticed.
- **Integration tests:** an example `.ork` plus motors run end to end.
- **Convergence tests:** solver-tolerance and step-size checks (Section 10).

Tests run before every commit. Claude Code can run them through the MATLAB MCP Server.

### 12.2 Analytic sanity checks

| Check | Expected result |
|-------|-----------------|
| ISA at 0 m | 288.15 K, 101325 Pa, about 1.225 kg/m^3 |
| ISA at 11 km | 216.65 K, about 22632 Pa |
| Ballistic flight, no drag, no thrust, constant g | apogee = `v0^2 / (2 g)` |
| Rocket equation, constant thrust, no drag, constant g | velocity gain from mass ratio and burn time matches `Isp g0 ln(m0/mf) - g t_burn` |
| Terminal velocity in descent | `sqrt(2 m g / (rho CD S))` |
| Mass conservation | initial minus final mass equals total propellant |
| Impulse conservation | integral of thrust equals the motor's total impulse |
| Cluster equivalence | N identical motors ignited together match one motor with N times the thrust and propellant |
| 1-DOF vs 3-DOF | identical results for vertical launch, no wind |
| Wind symmetry | a wind and its reverse produce mirrored downrange drift |

### 12.3 Per-module tests

| Module | Tests |
|--------|-------|
| `parse_ork` | geometry, mass, CG match OpenRocket; malformed file fails clearly |
| `parse_eng` | time increasing, non-negative thrust, impulse consistent with class |
| `isa_atmosphere` | table values above; continuity at layer boundaries; temperature offset behaves |
| aero components | Section 8.10 sanity checks |
| propulsion | analytic checks above; cluster timing |
| dynamics | analytic checks above; convergence test |

### 12.4 Practices

- A bug fix comes with a test that would have caught it.
- Tolerances are stated in the test, with the reason.
- Published reference values used in tests carry their source.

## 13. Risks and open questions

| Item | Notes |
|------|-------|
| Transonic accuracy | expected to be the weakest region for any method short of CFD; results here carry the largest error bars |
| `.ork` mass and CG | OpenRocket computes component masses from materials and geometry; the parser must reproduce this, verify against OpenRocket's reported mass and CG **[OPEN]** |
| RASAero II data access | confirm how to extract Cd, CN, and CP vs Mach for comparison **[OPEN]** (first task of M4) |
| Schedule | Tier 1 is ambitious for about 12 weeks alongside coursework; Tier 2 is a stretch |
| Reference rockets | resolved: you will design your own test rockets in OpenRocket when needed (aim for roughly a low-speed, a ~Mach 1.5, and a ~Mach 2.5 case) |
| Equation constants | items marked [VERIFY] in Sections 6-10 must be checked against sources when implemented |

## 14. References and verification

### 14.1 Verification log

One row per [VERIFY] item, filled in when it is checked against the source.

| Item | Section | Source checked | Date | Result / correction |
|------|---------|----------------|------|---------------------|
| (none yet) | | | | |

### 14.2 Reference list

Every equation in Sections 6-10 will be listed here with its full source as it is verified.

---

## Decision log

| Date | Decision |
|------|----------|
| 2026-09-23 | General-purpose tool; competition was inspiration only |
| 2026-09-23 | Hybrid aero: build from theory, validate against RASAero II |
| 2026-09-23 | Fidelity staged 1-DOF, 3-DOF, 6-DOF |
| 2026-09-23 | Tier 1 = single-stage and clusters; staging, boosters, flutter, heating deferred |
| 2026-09-23 | SI units only |
| 2026-09-23 | Atmosphere: ISA plus constant wind, site elevation, temperature offset |
| 2026-09-23 | Workflow: OpenRocket `.ork` in, script-driven, GUI later |
| 2026-09-23 | Success criteria, milestone timing, and repo layout confirmed; consistent bias acceptable |
| 2026-09-23 | Reference rockets: self-designed test rockets, created when needed |
| 2026-09-23 | Aaron verifies all numerical constants and equations against sources before use; logged in 14.1 |
| 2026-09-23 | Environment: geopotential conversion, dT applied to whole column; 3-DOF assumes rocket aligned with relative wind |
