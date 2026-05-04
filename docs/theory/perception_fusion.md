# Perception Fusion — Multi-Modal Counter-UAS Sensing at Engagement Distances

**Project:** TALON-MESH
**Domain:** Counter-UAS perception architecture

## 1. Purpose

This document characterizes TALON-MESH's perception architecture for the counter-UAS application context. Counter-UAS perception faces a distinctive set of challenges — small, low-radar-cross-section, low-altitude, often slow but sometimes very fast targets, in cluttered environments — that distinguishes it from both general air-defense perception (where targets are larger and at higher altitude) and from typical autonomous-driving perception (where targets are larger and at ground level).

The architecture combines mmWave Doppler radar, acoustic-array sensing, RF passive monitoring, and electro-optical sensors. Each modality contributes distinct information about UAS targets; their fusion produces perception capabilities that no single modality achieves alone. This document specifies the per-modality contribution, the fusion structure, and the integration with PentaTrack.

The document is at survey-paper depth; it characterizes the published literature and the architecture's adaptation of it. The simulator implements the architecture; researchers can substitute their own per-modality models or fusion variants for comparison studies.

## 2. The Counter-UAS Perception Problem

Counter-UAS targets exhibit several characteristics that distinguish them from general air-defense and autonomous-driving targets:

- **Small radar cross-section.** Consumer UAS have radar cross-sections orders of magnitude smaller than manned aircraft, requiring sensitive radar architectures.
- **Low altitude.** UAS frequently operate at altitudes below typical air-traffic-management radar coverage; the published literature on low-altitude tracking is the relevant precedent.
- **Variable velocity.** UAS range from near-hover (multirotors holding position) to tens-of-meters-per-second (cruising) to over 100 m/s (high-performance UAS or those in dive maneuvers). The perception architecture must accommodate this entire range.
- **Distinctive acoustic signatures.** Multi-rotor UAS produce characteristic acoustic spectra (rotor-blade-passing frequencies and harmonics) that differ from natural sound sources; published research demonstrates acoustic-classification accuracy at high single-digit-percent error rates.
- **RF emission.** Most UAS emit on identifiable RF frequencies for command-and-control, video downlink, and telemetry. Passive RF monitoring provides classification information.
- **Visual signatures.** Optical sensors detect UAS through their reflectance and motion; published research demonstrates classification accuracy approaching that of human-trained operators.

The perception architecture exploits all of these signature dimensions in a multi-modal fusion structure.

## 3. The Four Primary Modalities

### 3.1 mmWave Doppler Radar

mmWave radar at 24, 60, or 77 GHz, similar to HALO-AD's deployment. Counter-UAS radar configuration emphasizes:

- **Doppler-discrimination of UAS rotors.** A multi-rotor UAS produces a characteristic Doppler-spectrum signature (the rotors' rotational velocity contributes harmonics distinct from the body's translational velocity). Published research demonstrates rotor-Doppler classification.
- **Sensitivity to small RCS.** Counter-UAS radar requires lower-noise-floor receivers and longer integration times than general air-defense radar.
- **Phased-array architectures.** Many counter-UAS radars use phased arrays for fast electronic scanning across the relevant azimuth and elevation.
- **MIMO and cognitive radar.** Published research extends the basic FMCW architecture with multi-input/multi-output and adaptive-waveform capabilities.

In TALON-MESH's role, mmWave radar provides primary track initiation and continuous tracking. Most engagements begin with radar detection.

### 3.2 Acoustic Array

Multi-microphone arrays capture UAS acoustic signatures. Published research:

- **Direction-of-arrival estimation.** Standard array-processing techniques (beamforming, MUSIC, ESPRIT) provide angular localization of the UAS sound source.
- **Classification.** Machine-learning classifiers distinguish UAS from birds, vehicles, and other acoustic sources with high accuracy in the published literature, particularly for multi-rotor UAS.
- **Range estimation.** Acoustic ranging is geometrically constrained but provides corroborating information for track initialization.

In TALON-MESH's role, acoustic provides classification information that radar alone cannot easily provide (radar can detect but distinguishes UAS from large birds with difficulty), and provides perception robustness in conditions where radar may be limited.

### 3.3 RF Passive Monitoring

Passive monitoring of UAS RF emissions:

- **Command-and-control link.** Most UAS use specific RF protocols for operator-vehicle communication (e.g., specific frequency bands and modulations). Identifying the protocol identifies the UAS class.
- **Video downlink.** UAS with video capabilities transmit on identifiable channels with identifiable characteristics.
- **Telemetry.** Position and status telemetry, where unencrypted, provides direct UAS state information.
- **Direction-finding.** Multi-antenna passive monitoring provides direction-of-arrival of the UAS.

In TALON-MESH's role, RF passive monitoring provides classification beyond what radar or acoustic alone can provide, plus advance warning when UAS appear in the RF environment before they appear in the radar field. Active interference with these signals is regulated and outside repository scope (per `legal/compliance.md`).

### 3.4 Electro-Optical Sensors

Visible-light cameras, near-IR, mid-wave IR, long-wave IR, and event-based cameras. Each spectral band has characteristic strengths:

- **Visible-light.** Highest spatial resolution; effective in good lighting; classification accuracy comparable to human-trained operators in published research.
- **Near-IR.** Effective in low-light conditions; less affected by glare than visible-light.
- **Mid-wave and long-wave IR.** Effective at night; thermal contrast distinguishes UAS from background; particularly effective at detecting UAS battery and motor heat.
- **Event-based.** Microsecond-scale change detection; suitable for fast UAS detection; sparse output reduces processing burden.

In TALON-MESH's role, electro-optical provides high-resolution classification and refinement of tracks initiated by radar or RF monitoring. Multiple electro-optical sensors at different spectral bands provide robustness across lighting and weather conditions.

## 4. The Fusion Architecture

Modalities feed into a fusion pipeline structurally similar to HALO-AD's `sensor_fusion.md` architecture but with counter-UAS-specific considerations:

### 4.1 Detection Layer

Each modality processes its input in its native form:

- **Radar.** Range-Doppler maps with CFAR detection, Doppler-spectrum analysis for rotor-classification.
- **Acoustic.** Direction-of-arrival estimation, classifier output.
- **RF passive.** Protocol-classification output, direction-of-arrival, signal-strength.
- **Electro-optical.** Detection and classification per spectral band.

### 4.2 Track-Level Fusion

Modality-specific detections fuse into tracks. Each track integrates contributions from its modalities:

- A track initiated by RF passive may be refined by radar (which adds geometric position) and electro-optical (which adds visual classification).
- A track initiated by radar may be refined by acoustic (which adds class confirmation) and electro-optical (which adds visual classification).

The fusion algorithm is similar to HALO-AD's track-level fusion but with classification weighting that emphasizes the multi-modal classifier ensemble.

### 4.3 Classification Layer

Counter-UAS classification is multi-class — distinguishing among UAS classes (multi-rotor vs. fixed-wing vs. specific platforms), distinguishing UAS from non-UAS targets (birds, balloons, debris), and detecting threat-relevant characteristics (payload-carrying, autonomous vs. operator-controlled). The classification layer combines per-modality classifier outputs with appropriate weighting.

### 4.4 PentaTrack Integration

Fused tracks feed PentaTrack as the predictive-tracking substrate. PentaTrack's parameter choices for the counter-UAS context (per `physics_models.md`): recursion depth 2-3, velocity-weighted prediction with cosine strategy, diagonal centers enabled (UAS maneuver in arbitrary directions), object-type awareness with per-UAS-class drift profiles, drift analysis essential.

## 5. Modality Complementarity for Counter-UAS

| Dimension | Radar | Acoustic | RF Passive | EO |
|---|---|---|---|---|
| Detection range (typical) | Long | Short | Variable | Long |
| Classification confidence | Moderate | High | Very High | High |
| All-weather | Yes | Mostly | Yes | Limited |
| Low-altitude effective | Yes | Yes | Yes | Yes |
| Static-target observability | Limited | No | If transmitting | Yes |
| Adversarial-resistance | Moderate | Moderate | High (passive) | Limited |

Each modality has weak points covered by others. The fusion architecture's value is the *complementarity*.

## 6. Adversarial Considerations

Counter-UAS perception faces adaptive adversaries. Documented adversarial trends:

- **Low-observable UAS.** Reduced radar cross-section, reduced acoustic signature, reduced RF emission. Multi-modal fusion partially compensates.
- **Maneuvering.** Aggressive maneuvers on detection. PentaTrack's adaptive-drift extension addresses this.
- **Swarming.** N-on-M saturation. The decision-layer addresses this; perception layer must support multi-target tracking.
- **Decoys.** Decoy UAS or chaff to absorb counter-UAS effort. Multi-modal fusion partially compensates (decoys typically lack the full multi-modal signature of real UAS).
- **Sensor-domain attack.** Optical jamming, RF jamming, acoustic confusion. Multi-modal redundancy is the primary mitigation.
- **Networked operations.** Multiple UAS coordinating attacks. Game-theoretic and multi-agent learning frameworks; research-domain.

The simulator's scenario library includes adversarial-perception scenarios across each category. Researchers can study fusion-architecture robustness against specific attack vectors.

## 7. Multi-Mast / Multi-Sensor Track Fusion

When multiple counter-UAS perception nodes observe the same UAS, their per-node tracks fuse into a shared track. Same algorithmic substrate as HALO-AD's multi-mast track fusion, with counter-UAS-specific classification considerations.

## 8. Civilian Transfer

The perception fusion architecture transfers to civilian airborne-vehicle perception:

- **Air-traffic-management cooperative-surveillance integration.** Combining the ATM transponder data (cooperative aircraft) with radar (uncooperative aircraft) is structurally similar.
- **Wildlife-monitoring of flying species.** Multi-modal monitoring of bats, birds, and insects in the airspace shares many architectural elements; the parameter regime differs.
- **Sports-photography drone tracking.** Cinematography and broadcast UAS at large events use multi-modal positioning systems with similar architectures.
- **Search-and-rescue UAS coordination.** Multiple UAS searching for ground targets use perception architectures structurally analogous to the counter-UAS architecture inverted.
- **Agricultural-UAS swarm coordination.** Coordinated multi-UAS spraying and survey systems use multi-modal positioning with similar architectures.

The civilian-transfer examples will demonstrate at least one example per civilian domain.

## 9. References

- Published counter-UAS literature: Counter-UAS-specific papers in IEEE Transactions on Aerospace and Electronic Systems; conference proceedings of IEEE International Conference on Unmanned Aircraft Systems.
- Published mmWave radar for small targets: literature cited above for HALO-AD plus counter-UAS-specific research.
- Published acoustic-classification of UAS: papers in Applied Acoustics, JASA, and conference proceedings of ICASSP.
- Published RF passive monitoring of UAS: papers in IEEE Transactions on Vehicular Technology, conference proceedings of MILCOM (open-publication subset).
- Published electro-optical UAS detection: papers in IEEE Transactions on Pattern Analysis and Machine Intelligence; conference proceedings of CVPR, ICCV.
- Published multi-sensor-fusion for low-altitude tracking: papers in conference proceedings of FUSION.

The maintainers update this references section as the literature evolves.

## 10. Composition with Other Theory Documents

Perception fusion is one layer of TALON-MESH's overall framework:

- `multi_effector_arrays.md` — Consumes perception-layer outputs (PentaTrack predictive-center fields) for multi-effector engagement formulation.
- `line_of_sight_geometry.md` — Consumes perception-layer track outputs for line-of-sight-aware assignment.
- `future_research.md` — Documents adjacent research domains for which the perception layer would provide upstream input.

The simulator integrates all of these; this document specifies the perception substrate on which the integration rests.

## 11. Summary

Counter-UAS perception combines mmWave Doppler radar, acoustic arrays, RF passive monitoring, and electro-optical sensors in a multi-modal fusion architecture parameterized for the counter-UAS regime. Modality complementarity provides robustness against adversarial conditions and graceful degradation under environmental variation. The fusion architecture composes with PentaTrack for predictive tracking and with the multi-effector-arrays document for engagement planning.

The civilian-transfer applications span air-traffic management, wildlife monitoring, sports broadcasting, search-and-rescue, and agricultural UAS coordination. The framework is general; the parameter regime is counter-UAS-specific.
