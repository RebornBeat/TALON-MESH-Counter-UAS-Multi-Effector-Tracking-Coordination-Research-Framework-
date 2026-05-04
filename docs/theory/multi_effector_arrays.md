# Multi-Effector Arrays — Probabilistic Coverage of Predictive-Center Fields

**Project:** TALON-MESH
**Domain:** Multi-element coordination algorithms

## 1. The Problem

A target tracked by PentaTrack at depth 1 is represented by 5 (or 9, with diagonals enabled) weighted predictive centers covering the plausible positions the target may occupy at the next prediction time. A single-element effector can address only one of these centers, accepting the probability mass at that center as its expected coverage of the target's actual future position.

A multi-element effector array can address multiple centers simultaneously. The question is which centers, and how to choose, to maximize the joint probability that at least one element addresses the actual future position.

This document specifies the formulation, the algorithms TALON-MESH's `software/decision/` module uses to solve it, and the performance characterization.

## 2. Formulation

Let:
- *C* = {*c*₁, *c*₂, ..., *cₙ*} be the predictive centers for a single target (n ∈ {5, 9, ...}).
- *p*ᵢ be the probability weight of center *cᵢ* (Σ *pᵢ* = 1, with caveats from PentaTrack's velocity weighting).
- *N* be the number of array elements available.
- *S* ⊆ *C* be a subset of centers to address (|*S*| ≤ N).

The objective is to choose *S* to maximize the probability that the target's actual future position is "covered" by at least one of the addressed centers.

Under a simplifying independence assumption (the centers are non-overlapping in their coverage regions, which is approximately true for the PentaTrack geometry), the joint coverage probability is:

P(covered | S) = Σᵢ∈S pᵢ

In this formulation, the optimal *S* is the *N* highest-weight centers. This is the baseline policy.

## 3. Beyond the Baseline

The independence assumption breaks in several practically-relevant ways, each producing a more interesting optimization:

### 3.1 Overlapping Coverage Regions

Two centers close together cover overlapping regions of physical space. Addressing both is less valuable than addressing one of them plus a third center elsewhere. The objective becomes a *submodular* function over *S*:

P(covered | S) = ∫ region(s) · pdf(target_position) ds

where `region(S)` is the union of coverage regions of the centers in *S*. Submodular maximization with cardinality constraint is well-studied; the greedy algorithm achieves a (1 - 1/e) ≈ 63% approximation to the optimum, and TALON-MESH's implementation uses this as the default.

### 3.2 Per-Element Constraints

Real array elements have characteristic-specific constraints:

- **Field of regard.** Each element can only address centers within its FoR.
- **Slew time.** Re-aiming an element from one center to another takes time; if the time exceeds the prediction horizon, the assignment is infeasible.
- **Range.** Each element has a maximum effective range.

These constraints turn the assignment into a constrained submodular maximization. TALON-MESH's implementation uses a constraint-respecting greedy algorithm, with the constraints as hard filters on the candidate set at each greedy step.

### 3.3 Multi-Target Mode

When the array faces *T* > 1 targets simultaneously, the formulation generalizes:

maximize: Σ_targets (priority · P(covered | S_target))
subject to: Σ_targets |S_target| ≤ N (total array elements)

This is a *combined* assignment problem: how to allocate array capacity across targets, then within each target's allocation, which centers to address. The published assignment-problem literature (auction algorithms, Hungarian algorithm variants for the cross-target step) combined with the submodular-maximization literature for the within-target step provides the algorithmic substrate.

## 4. Velocity-Weighted Center Distributions

PentaTrack's velocity-weighted prediction skews the center weights toward the target's current motion direction. The multi-effector assignment exploits this:

- For a slow-maneuvering target, weights cluster sharply around the central predicted position. The first 1-2 highest-weight centers cover most of the probability mass; subsequent elements have diminishing returns. The array's optimal strategy is to use those subsequent elements on *other* targets if available, or to hold them in reserve.
- For an aggressively-maneuvering target, weights spread across many centers. The array's optimal strategy is to address several centers on this target, accepting that the per-target allocation must come at the expense of other-target coverage.

The simulator characterizes this tradeoff as a function of target maneuverability and array size.

## 5. Drift-Aware Assignment

PentaTrack's drift analysis tracks which predicted center has been most-accurate over recent frames (the drift leader). The assignment optimizer can use this signal to weight centers beyond their nominal probability:

effective_weight(c) = nominal_weight(c) · drift_leader_bonus(c)

The bonus increases the weight of the recent drift leader, biasing the assignment toward continuity with recent-past accuracy. The simulator quantifies the value of this bias across scenarios.

## 6. Multi-Hypothesis Targets

Targets that have not yet been classified — early in their tracks, before the object-type-awareness module has produced a confident classification — produce broader predictive-center distributions because the target's per-class drift profile is unknown. The assignment optimizer can either:

- **Wait for classification.** Allocate fewer elements to unclassified targets, accepting lower per-target coverage in exchange for not committing array capacity prematurely.
- **Hedge across classifications.** Allocate elements to centers that are high-weight under multiple plausible target classes.

Both strategies are policy-configurable. The simulator characterizes each across scenarios.

## 7. The Saturation Boundary

When the number of targets exceeds the number of array elements, the system cannot address every target. The decision becomes:

- Which targets receive array capacity?
- Which receive reduced or no capacity?

This boundary touches questions of priority, classification confidence, and policy that are not purely algorithmic. The repository implements priority-weighted greedy allocation as a baseline (highest-priority targets receive array capacity first; lowest-priority targets receive zero); other policies are documented in `docs/theory/future_research.md` as research-domain questions.

The maintainers note that saturation-response policy in real counter-UAS contexts is a regulatory and operator-certification question, not a software-decision question. The repository does not implement policies that would purport to make those decisions; the simulator quantifies the consequences of different policies under controlled scenarios.

## 8. Civilian Transfer

The multi-element-coverage formulation transfers directly to:

- **Multi-cobot pick-and-place.** M cobots service N items on a conveyor; the cobots' coverage regions overlap; the assignment must cover the highest-priority items with the available cobots. Same submodular-with-budget formulation, different physical interpretation.
- **Multi-camera sports tracking.** M cameras must cover N players; cameras' fields of view overlap; the assignment must keep all (or as many as possible) players in view. Same formulation.
- **Multi-arm robotic surgery.** Multiple arms must cooperate on a procedure; arms' workspaces overlap; coordination is the multi-element-on-single-target case. Same formulation.
- **Multi-server load balancing.** Servers' service regions in a request-feature space overlap; assignment must minimize maximum load.

The civilian-transfer examples in `examples/civilian/` will demonstrate at least one transfer per algorithmic family.

## 9. Performance Characterization

The simulator produces:

- **Coverage-vs-N plots.** As array size increases, what fraction of total target probability mass is addressable?
- **Saturation curves.** At what target count does coverage degrade? How gracefully?
- **Algorithm comparison.** Greedy vs. optimal (where computable) vs. learned policies.
- **Civilian-transfer benchmark scenarios.** The same algorithms on the same scenarios across counter-UAS and civilian framings.

These outputs are CSV deliverables suitable for academic comparison.
