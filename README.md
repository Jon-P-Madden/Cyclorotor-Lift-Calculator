# Cyclorotor Lift Calculator

> Blade Element Momentum Theory engine with animated kinematics visualization and optional Claude AI analysis layer. Single-file HTML — no build step, no backend, no dependencies beyond a browser.

---

## Screenshots

| Rotor Visualization | Pitch Curve & Results |
|---|---|
| ![Rotor animation showing 4 blades with AoA color coding and thrust vector](screenshots/rotor-animation.png) | ![Primary results panel with thrust, power, and figure of merit](screenshots/results-panel.png) |

| Stall Warning Active | Claude AI Analysis |
|---|---|
| ![Blade AoA warning with stall fraction and design flags](screenshots/stall-warning.png) | ![Claude engineering analysis panel with optimization suggestions](screenshots/ai-analysis.png) |

---

## Problem Statement

Cyclorotors (cyclogyros) are fundamentally different from axial rotors: blades rotate around a horizontal axis while simultaneously pitching via an eccentricity mechanism, generating thrust that can be vectored continuously through 360°. The aerodynamics are non-trivial — lift is a function of azimuthal position, pitch law, blade geometry, and rotor speed interacting simultaneously — and there are no consumer-grade tools that model the physics correctly for small-scale builders.

Existing resources are either academic papers (equations without tooling) or oversimplified online calculators that treat cyclorotors like conventional propellers. The result: builders size hardware empirically, iterate through physical builds, and get poor first-article performance with no clear path to optimization.

**Core problem:** No accessible, physics-accurate design tool exists for cyclorotor builders at the hobbyist/small UAV scale.

---

## Design Constraints

| Constraint | Decision |
|---|---|
| Must run without a server or build pipeline | Single-file HTML — zero infrastructure |
| Must be usable offline at the bench | No CDN dependencies; all code self-contained |
| Physics must be credible to engineers | Full BEMT integration, not simplified coefficient lookup |
| Must reflect actual blade kinematics visually | Canvas animation at true RPM, AoA-colored blades |
| AI analysis must be optional, not required | Claude API layer is additive; tool is fully functional without it |
| Must support both unit systems | SI and Imperial inputs/outputs with live conversion |

**What was explicitly not built (scope control):**
- Forward-flight model (hover only — the highest-value case for initial sizing)
- Structural/stress analysis
- Multi-rotor interference modeling
- Reynolds number correction curves (user-selectable airfoil presets cover the practical range)

---

## Architecture

```
cyclorotor-calculator.html
│
├── Physics Engine (pure JS)
│   ├── ISA density model       — altitude → ρ
│   ├── BEMT integrator         — 720-step azimuthal integration
│   │   ├── Pitch law           — θ(φ) = θ_max · cos(φ − ψ)
│   │   ├── Lift model          — Clα · AoA with post-stall flat-top
│   │   ├── Drag model          — Cd0 + induced (finite span, Oswald ~0.88)
│   │   └── Force assembly      — time-averaged Fx, Fy, torque per blade
│   └── Actuator disk           — induced power, Figure of Merit
│
├── Visualization (Canvas 2D)
│   ├── Rotor animation         — live RPM, N blades, AoA color coding
│   ├── Thrust vector           — updates post-calculation
│   └── Pitch curve             — θ(φ) with live blade position markers
│
├── UI Layer
│   ├── Inputs                  — geometry, motion, pitch, airfoil, environment
│   ├── Airfoil presets         — flat plate, NACA 0008/0012/0015, AG04
│   ├── Warning system          — stall fraction, tip Mach, solidity limits
│   └── Unit toggle             — SI ↔ Imperial with live field conversion
│
└── AI Analysis Layer (optional)
    └── Anthropic API           — full params + results → engineering assessment
```

**Physics model notes:**

The integrator uses the eccentricity pitch law `θ(φ) = θ_max · cos(φ − ψ)`, which accurately represents a mechanical eccentricity mechanism where the pitch pivot is offset from the rotor center. The phase angle `ψ` shifts the thrust vector without changing magnitude — this is the cyclorotor's thrust vectoring mechanism.

Forces are computed in global frame coordinates and time-averaged over one revolution:
- Lift acts radially inward (toward rotor center) for positive AoA
- Drag opposes tangential blade motion
- Net vertical force = integral of radial lift components around the azimuth

Induced power uses the actuator disk model with projected frontal area `A = 2R·b`. This underestimates induced losses slightly at high blade loadings but is appropriate for initial sizing.

---

## Outputs

| Output | Description |
|---|---|
| Thrust (N / lbf / kg-force) | Total time-averaged rotor thrust in hover |
| Power Required (W / HP) | Profile drag power + actuator disk induced power |
| Figure of Merit | Hover efficiency: P_ideal / P_total (0–1) |
| Torque (N·m / lb·ft) | Shaft torque — drive sizing input |
| Thrust Vector Angle | Degrees from vertical — reflects phase angle setting |
| Max Blade AoA | Peak angle of attack in the revolution |
| Stall Fraction | % of revolution where blades exceed stall angle |
| Disk Loading | Thrust per unit projected area |
| Power Loading | Thrust per unit power — efficiency metric |
| Rotor Solidity | Nbc / (πR) — blade area fraction |

---

## Airfoil Presets

| Preset | Clα (/rad) | Cd₀ | Stall |
|---|---|---|---|
| Flat plate (thin) | 5.50 | 0.015 | 11° |
| NACA 0008 | 5.90 | 0.010 | 12° |
| NACA 0012 | 5.73 | 0.012 | 14° |
| NACA 0015 | 5.50 | 0.013 | 16° |
| AG04 (low Re) | 6.10 | 0.009 | 13° |

Custom values can be entered directly for any airfoil with known polars.

---

## AI Analysis Layer

With an Anthropic API key, the tool passes the complete parameter set and all calculated results to Claude with a system prompt tuned for cyclorotor engineering context. The analysis covers:

- Design viability assessment
- Performance bottleneck identification
- Specific parameter optimization suggestions
- Physical boundary warnings (compressibility, solidity limits, model validity)

The AI layer is additive — the calculator is fully functional without it. When used, it adds interpretation that the physics engine cannot provide: tradeoff reasoning, design intent alignment, and suggestions that require understanding the broader context of what the builder is trying to achieve.

---

## Usage

1. Open `cyclorotor-calculator.html` in any modern browser
2. Enter rotor geometry, RPM, pitch parameters, and airfoil properties
3. Auto-calculate is on by default — results update on any input change
4. Watch the blade visualization for AoA color feedback (green → amber → red)
5. Check the pitch curve panel — blade dots show current azimuthal positions
6. For AI analysis: paste an Anthropic API key and click **Analyze Design**

No installation. No account. No data leaves your machine except Anthropic API calls (if used).

---

## Known Model Limitations

- **Hover only** — no forward flight, no climb/descent induced velocity correction
- **No Reynolds number correction** — relevant at chord Re < 50,000; use AG04 preset as a proxy for low-Re regimes
- **No blade-to-blade interference** — model validity degrades above solidity ~0.35; a warning is shown
- **Post-stall model is simplified** — flat-top Cl, empirical Cd rise; do not rely on results where stall fraction > 20%
- **No compressibility correction** — tip Mach warning shown above M = 0.30
- **Eccentricity pitch law assumed** — other pitch mechanisms (cam, servo) will have different θ(φ) shapes

---

## Roadmap

### Phase 1 — Current
- [x] BEMT hover model with 720-step azimuthal integration
- [x] Animated blade kinematics with AoA color coding
- [x] Pitch curve visualization with live blade markers
- [x] SI / Imperial unit toggle
- [x] Airfoil presets + custom entry
- [x] Warning system (stall, Mach, solidity)
- [x] Claude AI analysis layer

### Phase 2 — Planned
- [ ] Parametric sweep: thrust and FM vs. θ_max at fixed RPM (find optimal pitch amplitude)
- [ ] RPM sweep: power curve from zero to max RPM
- [ ] Export results to CSV
- [ ] Save/load design configurations (localStorage)

### Phase 3 — Stretch
- [ ] Induced velocity correction (iterative momentum theory for hover accuracy)
- [ ] Reynolds number Cl/Cd correction curves
- [ ] Multi-rotor layout: define rotor positions, compute net thrust/torque balance
- [ ] Forward flight model (advance ratio, asymmetric blade loading)

### Phase 4 — Research
- [ ] Compare BEMT output against experimental data (user-supplied thrust stand measurements)
- [ ] Blade structural loading estimates (centrifugal force, bending moment at root)
- [ ] Noise estimation (blade passage frequency, tip vortex model)

---

## Why This Exists in a Portfolio

This tool was built because I'm actively developing a cyclorotor-based aircraft and needed a design loop faster than physical iteration. The problem is real, the physics model is non-trivial, and the tooling gap is genuine.

From a TPM standpoint, this project demonstrates the pattern that matters: identifying a concrete problem, defining scope constraints that make it buildable, making deliberate architecture tradeoffs (BEMT accuracy vs. forward-flight complexity), and shipping something useful before solving everything.

The AI layer is the same pattern as the [TPM Application Engine](../tpm-application-engine) — deterministic computation handles what can be computed, AI handles interpretation and reasoning that requires context the model doesn't have.

---

## Privacy

- No data is stored or transmitted except Anthropic API calls (rotor parameters + results sent in request body)
- No analytics, no telemetry, no external scripts
- API key is held in memory for the session only — not written to localStorage or disk

---

## License

MIT — use it, modify it, build on it.

---

## References

- Wheatley, J.B. (1934). *An Aerodynamic Analysis of the Autogiro Rotor with a Comparison Between Calculated and Experimental Results.* NACA TR-487.
- Iosilevskii, G. & Levy, Y. (2006). *Aerodynamics of the Cyclogiro.* AIAA Journal, 44(12).
- Benedict, M. et al. (2013). *Fundamental Understanding of the Cyclocopter Concept for Micro Air Vehicle Applications.* Journal of the American Helicopter Society.
