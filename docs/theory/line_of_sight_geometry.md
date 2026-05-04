# Line-of-Sight Geometry — A Primary Constraint in Multi-Effector Assignment

**Project:** TALON-MESH
**Domain:** Geometric constraints in assignment optimization

## 1. The Problem

A naive assignment optimizer minimizes target-to-effector distance, with the implicit assumption that the shortest path between effector and target is unobstructed. In real engagement geometries this assumption fails routinely: terrain, structures, other targets, and atmospheric obscurants all interrupt lines of sight. An effector with the shortest path to the target is useless if the path is blocked.

This document specifies how TALON-MESH's `software/decision/` module incorporates line-of-sight as a primary constraint rather than a post-hoc filter.

## 2. Line-of-Sight as a Geometric Predicate

For each (effector, target) pair, define:

```
LOS(effector, target) = True if no obstruction lies on the segment from effector to target
                       False otherwise
```

The predicate is computed against the simulator's static obstruction map (terrain, structures from the scenario library) and the dynamic obstruction map (other targets, transient obscurants).

Computing LOS is a ray-traversal against a voxelized world; the simulator implements this via standard ray-marching against the scenario's voxel grid.

## 3. Beyond Binary LOS

In practice, LOS is not binary. Partial obstructions, atmospheric obscurants, and target shape produce a continuum:

```
LOS_quality(effector, target) ∈ [0, 1]
```

where 0 is fully obstructed and 1 is fully clear. The simulator computes LOS quality as the product of:

- **Geometric LOS fraction.** The fraction of the effector aperture that has clear paths to the target.
- **Atmospheric transmission.** The Beer-Lambert transmission along the path (see `software/simulator/atmospheric_models.md`).
- **Target visibility.** The fraction of the target presented to the effector (some target poses present less observable area).

The combined LOS quality is a continuous-valued input to the assignment cost function.

## 4. Assignment Cost Function

The cost function for assigning effector *e* to target *t* combines:

```
cost(e, t) = base_cost(e, t)
           - LOS_quality(e, t) · LOS_weight
           - geometric_advantage(e, t) · geometry_weight
           + slew_time(e, current_target, t) · slew_weight
           + ...
```

Each term is policy-configurable. The default policy weights LOS quality heavily — an effector with LOS quality below 0.3 is effectively excluded from candidate assignments regardless of its other characteristics.

## 5. Geometric Advantage

Beyond binary LOS, the *angle of approach* matters:

- An effector approaching the target from a direction the target is not aware of gains a geometric advantage.
- An effector approaching the target from above (mast-elevated) gains advantage against ground-shadowed targets.
- An effector approaching the target along its velocity vector vs. perpendicular to it produces different engagement geometries.

The simulator parameterizes geometric-advantage as a multi-factor score combining angle of approach, elevation differential, and target-pose factors. The assignment cost function uses this score; the policy weights are configurable.

## 6. Multi-Effector LOS Coordination

When multiple effectors might engage a single target, the LOS calculus extends:

- **Diversity of approach angles.** Two effectors approaching from different angles provide engagement from different perspectives, reducing the value of countermeasures effective against any single angle.
- **Mutual occlusion.** Effectors that lie along each other's paths to the target obstruct each other.
- **Coordinated handoff.** As a target moves, the LOS quality from each effector changes; assignment may need to transition from effector A to effector B as the geometry evolves.

These multi-effector considerations turn assignment into a joint optimization over (effector, target, time) rather than a per-frame independent optimization.

## 7. Mast-Elevation Effects on LOS

Mast elevation, beyond its effects on coverage geometry documented in HALO-AD's `coverage_geometry.md`, has direct LOS effects:

- A mast-elevated effector has a higher-elevation viewing angle on most ground-confined targets, reducing the fraction of the target obscured by structures.
- A mast-elevated effector has direct LOS to targets in valleys and behind low structures that are invisible to ground-level effectors.
- Multiple masts at different elevations produce diverse LOS coverage of the same target volume.

The simulator parameterizes mast height and computes LOS quality across a heterogeneous-elevation array. The output is a coverage-vs-mast-height curve for each scenario.

## 8. Adversarial LOS Manipulation

Adversarial scenarios can manipulate LOS:

- **Smoke and obscurants.** Atmospheric attacks on optical or LiDAR effectors. Less effective against radar.
- **Terrain hugging.** Targets that fly along terrain features to maintain LOS denial.
- **Use of structures.** Targets that maneuver behind buildings or other structures.
- **Decoy positioning.** Decoys that occupy LOS paths from effectors to high-value real targets.

The scenario library includes adversarial scenarios across each category. The simulator quantifies how the assignment optimizer's policy degrades against each adversarial scenario.

## 9. Civilian Transfer

The line-of-sight assignment framework transfers to:

- **Multi-camera sports tracking.** Cameras whose paths to a player are blocked by other players or structures cannot contribute to that player's coverage.
- **Multi-robot inspection of structures.** Robots whose paths to inspection points are blocked by structure geometry cannot inspect those points.
- **Multi-drone agricultural inspection.** Drones whose paths to inspection points are blocked by foliage or terrain cannot inspect those points.
- **Wildlife camera networks.** Cameras whose views are obstructed by vegetation cannot capture the animals of interest.

In each case the assignment optimizer's LOS-aware policy outperforms the naive distance-minimizing policy by a quantifiable margin. The civilian-transfer examples will demonstrate the margin across application domains.

## 10. Implementation Notes

The LOS computation is the simulator's most expensive per-frame operation in scenarios with many effectors and many targets. The implementation uses:

- Spatial hashing of obstruction geometry for fast ray-traversal.
- Caching of (effector, target) LOS results across nearby frames, with invalidation on relevant geometry changes.
- Hierarchical bounding volumes for early-exit on obviously-unobstructed pairs.

These optimizations keep the LOS computation tractable for scenarios with 10² effectors and 10³ targets at simulator-real-time rates on commodity research workstations.
