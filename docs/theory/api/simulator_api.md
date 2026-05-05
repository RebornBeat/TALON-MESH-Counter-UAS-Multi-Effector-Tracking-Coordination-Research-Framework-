# TALON-MESH Simulator API Reference

**Implementation:** `talon-cli` binary, `talon-sdk` crate, `talon-simulator::harness`

---

## CLI Interface

### `talon-cli run <scenario.toml>`

Runs a single scenario with the default assignment policy.

**Arguments:**
- `--policy <policy>` — Override scenario-specified assignment policy. Values: `NaiveNearest`, `LosAware`, `MultiCenterOptimal`.
- `--seed <u64>` — Random seed for deterministic simulation.
- `--output-dir <path>` — Override default output directory.
- `--verbose` — Emit per-timestep assignments to stdout.

**Example:**
```bash
talon-cli run scenarios/multi_target_saturation.toml --policy MultiCenterOptimal --seed 1234
```

---

### `talon-cli compare <scenario.toml>`

Runs a scenario with multiple assignment policies and produces a side-by-side comparison report.

**Arguments:**
- `--policies <list>` — Comma-separated list. Values: `NaiveNearest`, `LosAware`, `MultiCenterOptimal`.
- `--output <path>` — Output HTML comparison report.
- `--seed <u64>` — Same seed applied to all policy runs for fair comparison.

**Example:**
```bash
talon-cli compare scenarios/swarm_vs_limited.toml \
  --policies NaiveNearest,LosAware,MultiCenterOptimal \
  --output comparison_report.html
```

---

### `talon-cli report`

Aggregates results from a `runs/` directory into an HTML report.

**Arguments:**
- `--inputs <path>` — Directory containing run subdirectories.
- `--output <path>` — Output path.

---

## Scenario File Format (TOML)

### Top-level fields

```toml
[scenario]
name = "My Counter-UAS Study"
duration_seconds = 60.0
random_seed = 42
```

### Airspace volume

```toml
[airspace]
bounds_m = [2000.0, 2000.0, 500.0]

[[obstructions]]
type = "Box"
position_m = [800.0, 1000.0, 0.0]
dimensions_m = [200.0, 200.0, 40.0]
label = "Building_A"
```

### Effectors

```toml
[[effectors]]
id = "effector_alpha"
position_m = [500.0, 500.0, 20.0]
element_count = 4                          # Independently aimable elements
field_of_regard_degrees = 120.0
slew_rate_deg_per_sec = 45.0
nominal_range_m = 500.0
engagement_cycle_time_ms = 50.0
effector_class = "Generic"                 # Only Generic is provided (sim only)
simulated_miss_probability_base = 0.15    # At nominal range, clear LOS
```

### Threat trajectories

```toml
[[threat_trajectories]]
type = "ConsumerMultiRotor"
entry_m = [0.0, 1000.0, 50.0]
exit_m = [2000.0, 1000.0, 50.0]
velocity_ms = 15.0
maneuver = "Straight"                      # Straight | Evasive | Terrain

[[threat_trajectories]]
type = "CommercialMultiRotor"
entry_m = [0.0, 1500.0, 100.0]
exit_m = [2000.0, 500.0, 80.0]
velocity_ms = 20.0
maneuver = "Evasive"
max_turn_rate_deg_per_sec = 45.0

[[threat_trajectories]]
type = "Decoy"                             # Reduced multi-modal signature
entry_m = [0.0, 500.0, 50.0]
exit_m = [2000.0, 500.0, 50.0]
velocity_ms = 10.0
radar_cross_section_factor = 0.3          # Fraction of ConsumerMultiRotor RCS
acoustic_signature_factor = 0.2
rf_emission = false
```

### Assignment policy

```toml
[assignment]
policy = "MultiCenterOptimal"              # NaiveNearest | LosAware | MultiCenterOptimal
los_weight = 2.0                          # Relative weight of LOS in cost function
geometry_weight = 0.5
slew_weight = 1.0
load_weight = 0.3
los_minimum_threshold = 0.3              # LOS quality below this excluded from assignment
drift_leader_bonus = 0.2                 # Multi-center planner: bonus weight for drift leader center
```

### Perception

```toml
[perception]
modalities = ["MmWaveRadar", "Acoustic", "RfPassive", "ElectroOptical"]
enable_micro_doppler = true              # Radar UAS classification via rotor signature
enable_rf_passive = true                 # Passive RF monitoring simulation
```

### Atmosphere

```toml
[atmosphere]
profile = "Clear"
```

### Tracking

```toml
[tracking]
pentatrack_depth = 3
enable_diagonals = true
enable_velocity_weighting = true
enable_drift_analysis = true
enable_adaptive_drift = true
enable_object_type = true                # Uses UasClass drift profiles
```

---

## Output Format Reference

### `metrics_summary.json`

```json
{
  "scenario_name": "string",
  "policy": "MultiCenterOptimal",
  "duration_seconds": 60.0,
  "target_count": 8,
  "effector_slot_count": 12,
  "total_engagements": 47,
  "simulated_effects": 31,
  "miss_count": 16,
  "effect_rate": 0.66,
  "mean_los_quality_assigned": 0.82,
  "mean_probability_mass_covered": 0.73,
  "track_loss_count": 2,
  "saturation_onset_target_count": 6
}
```

### `assignment_trace.csv`

One row per engagement per frame:
```
timestamp_ns, effector_id, element_index, target_id, addressed_center_label, center_weight, los_quality, slew_time_ms, simulated_outcome
```

Where `simulated_outcome` is one of: `EffectSimulated`, `Miss_DistanceError`, `Miss_LosOcclusion`, `Miss_SlewTimeout`.

### `outcome_statistics.csv`

Per-engagement outcome with target state:
```
timestamp_ns, target_id, uas_class, target_true_x_m, target_true_y_m, target_true_z_m, target_velocity_ms, outcome, probability_at_outcome
```

### `track_log.csv`

Per-frame per-target tracking:
```
timestamp_ns, target_id, true_x_m, true_y_m, true_z_m, estimated_x_m, estimated_y_m, estimated_z_m, position_error_m, pentatrack_best_center_weight, track_state, modalities_active
```

### `comparison_report.html`

When using `talon-cli compare`, the HTML report includes:
- Side-by-side bar charts: effect rate, mean LOS quality, probability mass covered, compute time
- Per-scenario time-series: effect count vs. target count (saturation curves)
- Policy recommendation with justification
- Civilian transfer equivalents (same metrics for the cobot scenario if included)

---

## SDK Crate API (`talon-sdk`)

```rust
use talon_sdk::prelude::*;

// Load scenario
let config = ScenarioConfig::from_toml("scenarios/multi_target_saturation.toml")?;

// Override assignment policy programmatically
let config = config.with_policy(AssignmentPolicy::MultiCenterOptimal {
    los_weight: 2.0,
    drift_leader_bonus: 0.2,
});

// Run
let mut harness = SimulatorHarness::new(config);
let metrics = harness.run_to_completion()?;

// Access assignment trace
for record in harness.assignment_trace() {
    println!(
        "t={} effector={} center={} los={:.2} outcome={:?}",
        record.timestamp_ns,
        record.effector_id,
        record.center_label,
        record.los_quality,
        record.outcome
    );
}
```

### Custom effector probability models

```rust
use talon_sdk::effector::{SimulatedEffector, EffectorDescriptor};

struct MyCustomEffector {
    base: EffectorDescriptor,
    // Custom probability model parameters
}

impl Effector for MyCustomEffector {
    fn simulate_engagement(
        &self,
        target_state: &TargetState,
        center: &CenterNode,
        t: Timestamp,
    ) -> SimulatedOutcome {
        let distance_m = (target_state.position_m - center.position).norm();
        let p_effect = self.custom_probability(distance_m, target_state.uas_class);
        if rand::random::<f32>() < p_effect {
            SimulatedOutcome::EffectSimulated { effect_at: t, effect_probability: p_effect }
        } else {
            SimulatedOutcome::Miss { reason: MissReason::ProbabilisticMiss }
        }
    }
    // ...
}
```

### Civilian transfer example hook

```rust
// The cobot scenario uses the same infrastructure
let cobot_config = ScenarioConfig::from_toml("scenarios/civilian_cobot_transfer.toml")?;
// "effectors" = robotic arms, "targets" = conveyor items, "obstructions" = workspace collision volumes
// Exactly the same assignment_optimizer.rs runs this scenario without modification
let cobot_metrics = SimulatorHarness::new(cobot_config).run_to_completion()?;
```
