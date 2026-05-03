# 4. TALON-MESH — Counter-UAS Multi-Effector Tracking Coordination (Research Framework)

**An open research and simulation framework for studying multi-target, multi-center, multi-effector coordination problems in counter-UAS perception and decision systems.**

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](#license)
[![Status: Research](https://img.shields.io/badge/Status-Research%20Only-red.svg)](#scope-and-disclaimers)
[![Effector-Agnostic](https://img.shields.io/badge/Effectors-Abstract%20%2F%20Out%20of%20Scope-orange.svg)](#scope-and-disclaimers)

---

## Scope and Disclaimers

TALON-MESH is a **research and education** framework for the *coordination and tracking* problem in counter-unmanned-aircraft (counter-UAS) systems. Counter-UAS is a well-established and openly published academic field with active research at universities, civil aviation authorities, and standards bodies worldwide; this project sits in that academic context.

**TALON-MESH studies and provides:**

- Multi-target, multi-effector *assignment* algorithms (which effector engages which target).
- Multi-center engagement geometry (using PentaTrack predictive centers, an effector array can plan to address multiple high-confidence predicted positions of a single target simultaneously to reduce miss probability).
- Sensor-fusion stacks for tracking small, fast, low-radar-cross-section targets typical of consumer UAS.
- Coordination protocols between fixed-site and aerial counter-UAS platforms.
- A simulation harness for evaluating the above in repeatable scenarios.

**TALON-MESH explicitly does not provide and will not accept contributions for:**

- Designs, schematics, firmware, or software for any actual lethal effector — projectile-firing, directed-energy, kinetic-kill, or otherwise.
- Designs for armed UAS / weaponized drones of any kind.
- Fire-control authority logic, target-engagement-authorization logic, or any code that issues a real-world "engage" command.
- Anti-personnel applications, target classification beyond small UAS, or any extension toward manned aviation, ground vehicles, or persons.
- Export-controlled (ITAR / EAR / Wassenaar) technical data.

**The "effector" abstraction.** The framework treats every effector as an opaque interface: a target ID goes in, a simulated outcome ("track held," "track lost," "simulated effect at time T") comes out. No real-world effector implementation is provided, and the framework will not load or execute one. The interface exists so that the *coordination and assignment* research is reproducible and benchmarkable; it is not an integration point for live hardware. Anyone wishing to extend this work toward physical counter-UAS hardware must independently obtain the appropriate legal, regulatory, FAA / national-aviation, and export-compliance approvals, plus all relevant operator certifications. The maintainers do not endorse, support, or enable such extension.

Counter-UAS operations are heavily regulated in every jurisdiction. In the United States, for example, the operation of counter-UAS systems is generally restricted to specific federal agencies under specific statutory authorities. This repository is a research artifact, not a usable system.

---

## The Research Question

When a defended airspace volume contains *multiple* small UAS, traditional one-target-one-effector systems — which point a single tracker and a single effector at a single threat — saturate quickly. Two open research questions follow:

1. **Multi-target assignment.** Given M effectors and N targets (with N possibly > M), what is the optimal time-evolving assignment that maximizes expected total track-hold or simulated-effect probability under uncertainty?

2. **Multi-center engagement on a single target.** PentaTrack produces, at every frame, a probability-weighted distribution of *predicted* future positions for each target — not a single point. If an effector array has multiple independently-aimable elements, the array can address several of the highest-weight predicted centers *simultaneously*, increasing the probability that at least one element addresses the target's actual future position. This is mathematically related to multi-hypothesis tracking and probabilistic collision avoidance, and it is a natural fit for PentaTrack's architecture.

Both problems are computationally interesting and directly transfer to civilian domains — multi-cobot pick-and-place, multi-drone agricultural coordination, multi-camera sports tracking — without any defense application.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    TALON-MESH Research Stack                 │
├──────────────────────────────────────────────────────────────┤
│  Scenario Layer                                              │
│  ──────────────                                              │
│  Airspace volume • Threat-track datasets • Effector array    │
│  topology • Atmospheric / environmental conditions           │
├──────────────────────────────────────────────────────────────┤
│  Perception Layer                                            │
│  ────────────────                                            │
│  Multi-sensor fusion (radar / acoustic / RF / EO sim)        │
│  PentaTrack per-target predictive center field               │
│  Multi-target track maintenance                              │
├──────────────────────────────────────────────────────────────┤
│  Decision Layer                                              │
│  ────────────────                                            │
│  Multi-target effector-assignment optimizer                  │
│  Multi-center single-target engagement planner               │
│  Latency / confidence / priority weighting                   │
├──────────────────────────────────────────────────────────────┤
│  Effector Abstraction (opaque interface)                     │
│  ─────────────────────────────────────                       │
│  Simulated outcome only — no real effector code              │
├──────────────────────────────────────────────────────────────┤
│  Output                                                      │
│  ──────                                                      │
│  Assignment traces • Outcome statistics • Visualization      │
│  Reproducible benchmark CSVs                                 │
└──────────────────────────────────────────────────────────────┘
```

---

## Multi-Center Engagement (Detailed)

For a single target, PentaTrack produces (after extensions) up to 9 weighted predictive centers per frame at depth 1, and recursively many more at greater depth. The highest-weight centers cluster along the velocity vector; lower-weight centers cover other plausible directions.

A counter-UAS *array* — for instance, a multi-element ground or aerial platform with N independently steerable elements — can be modeled as having N "engagement slots" per cycle. The multi-center engagement planner solves:

> Given N slots and the target's weighted center distribution at the planned engagement time, which N centers should the slots address to maximize total probability mass covered?

This is a weighted set-cover-with-budget problem over the center distribution. The solution naturally degrades gracefully as N decreases: one slot picks the dominant center; two slots pick the dominant plus the most-orthogonal high-weight; and so on. As N grows, additional slots cover lower-probability hedges.

The same math powers multi-target mode: rather than spending all N slots hedging on one target, the planner allocates slots across targets according to a global priority + probability product.

---

## What Lives in `effector/`

The `effector/` directory contains:

- A formal Python ABC (abstract base class) `Effector` with a single method, `simulate_engagement(target_state, t) -> SimulatedOutcome`.
- A reference *simulated* effector that returns probabilistic simulated outcomes drawn from configurable distributions.
- A test suite that exercises the assignment optimizer against the simulated effector.

What it does **not** contain:

- Any code that opens a serial port, network socket, or hardware interface to a real device.
- Any code that constructs, references, or enables a real effector.
- Any conditional branch that could be flipped from "simulation" to "live."

This is enforced by repository policy. Pull requests that introduce hardware integration are out of scope and will be closed.

---

## Repository Layout

```
talon-mesh/
├── docs/
│   ├── guides/
│   ├── theory/
│   │   ├── multi_center_engagement.md
│   │   ├── assignment_optimization.md
│   │   ├── perception_fusion.md
│   │   └── effector_abstraction.md
│   ├── api/
│   └── assets/
├── hardware/                  # PLACEHOLDER — explicit out-of-scope notice
│   └── README.md
├── firmware/                  # PLACEHOLDER
│   └── README.md
├── software/
│   ├── simulator/             # Scenario harness
│   ├── perception/            # Sensor fusion + PentaTrack
│   ├── decision/              # Assignment + multi-center planner
│   ├── effector/              # Opaque ABC + simulated reference
│   ├── viz/                   # 3D scenario visualization
│   ├── cli/
│   ├── sdk/
│   ├── protocol/
│   │   └── protocol_spec.md   # Internal sim-event protocol only
│   └── software.md
├── mechanical/                # PLACEHOLDER
│   └── README.md
├── production/                # NOT APPLICABLE
├── legal/
│   ├── compliance.md          # Counter-UAS regulatory landscape overview
│   ├── research_ethics.md
│   ├── export_control_posture.md
│   └── tos_compliance.md
├── media/
├── README.md
├── LICENSE
└── CHANGELOG.md
```

Every placeholder hardware folder contains: *"This directory is intentionally empty. TALON-MESH is a simulation and research framework. Physical hardware — including sensors, platforms, and effectors of any kind — is out of scope and will not be designed, specified, or distributed in this repository."*

---

## Quick Start

```bash
git clone https://github.com/<user>/talon-mesh.git
cd talon-mesh/software
pip install -e .

# Single-target multi-center engagement study
talon-mesh run scenarios/single_target_array.yaml

# Multi-target assignment study
talon-mesh run scenarios/multi_target_saturation.yaml

# Generate a benchmark report
talon-mesh report --inputs runs/ --output report.html
```

All scenarios are virtual. Outputs are HTML reports, scenario renders, and CSV benchmark data suitable for academic comparison.

---

## Civilian Transfer

The same algorithms in `decision/` directly apply to:

- Multi-cobot pick-and-place where M cobots service a moving conveyor with N items.
- Multi-camera sports tracking where M PTZ cameras must cover N players.
- Multi-drone agricultural inspection where M drones must cover N field plots under time pressure.
- Multi-vehicle dispatch.

The `examples/` directory will include at least one civilian scenario for each algorithm to make this transfer concrete.

---

## Roadmap

- Phase 1 (current): assignment optimizer + simulated effector ABC.
- Phase 2: multi-center engagement planner + PentaTrack integration.
- Phase 3: published benchmark scenarios for reproducible counter-UAS *perception/decision* research.
- Phase 4: civilian-transfer examples (cobot, sports, agricultural).

There is no Phase N for hardware. By design.

---

## License

MIT for code. CC BY 4.0 for documentation. See `LICENSE`, `legal/compliance.md`, and `legal/export_control_posture.md`
