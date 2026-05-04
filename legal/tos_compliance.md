# Terms of Service (ToS) Compliance — TALON-MESH

**Project:** TALON-MESH
**Document Status:** Active
**Last Updated:** 05/04/2026

## 1. Purpose of This Document

This document provides a comprehensive mapping of the TALON-MESH repository’s content against **GitHub’s Terms of Service (ToS)** and **Acceptable Use Policies (AUP)**. It serves as the primary compliance reference for maintainers, contributors, and users.

TALON-MESH operates in a domain (counter-unmanned aircraft systems) that is inherently dual-use. While the research is academically valid and legally protected speech, the implementation details require strict architectural boundaries to ensure platform compliance. This document defines those boundaries.

---

## 2. GitHub Acceptable Use Policies — Compliance Analysis

GitHub’s Acceptable Use Policies prohibit certain categories of content. Below is an analysis of how TALON-MESH adheres to each relevant restriction.

### 2.1. Illegal Content and Activities

**GitHub Policy:**
> "We do not allow content or activities that... facilitate or encourage illegal activity."

**Compliance Stance:**
TALON-MESH is a simulation and research framework. It does not facilitate illegal activity.

1.  **Counter-UAS Authority:** In the United States and most jurisdictions, the operation of counter-UAS systems (jamming, kinetic intercept, hacking) is restricted to specific government agencies. TALON-MESH **does not provide operational software** for these activities. It provides a *simulation harness* for studying the coordination logic.
2.  **No "Instructional" Weapons Content:** The repository does not provide instructions on how to build, modify, or use illegal weapons. The theoretical content (e.g., assignment algorithms, geometry calculations) is standard academic material used in logistics, robotics, and mathematics.

### 2.2. Weapons and Explosives

**GitHub Policy:**
> "We do not allow content that... promotes or provides instructions on how to create weapons... or associated components."

**Compliance Stance:**
TALON-MESH rigorously adheres to the "Simulation-First" architecture to ensure compliance:

1.  **No Physical Design Files:** The `hardware/`, `firmware/`, and `mechanical/` directories are intentionally empty. We do not host PCB layouts for radar exciters, CAD files for kinetic interceptors, or blueprints for directed-energy optics.
2.  **The "Effector Abstraction":** The software defines an "Effector" strictly as an **Abstract Base Class (ABC)**.
    *   **Allowed:** Defining an interface `simulate_engagement(target_state, t) -> SimulatedOutcome`.
    *   **Prohibited:** Defining an interface `fire_laser(target_coords)` or `activate_jammer(signal_freq)`.
    *   **Implementation:** The repository strictly prohibits the inclusion of any driver code, API wrappers, or configuration files that would interface with real-world weapon hardware.
3.  **No Munitions Data:** We do not publish explosive compositions, propellant formulas, or material densities for armor-piercing. The "passive materials research" is limited to referencing standard, commercially available fibers (e.g., UHMWPE) in theoretical contexts.

### 2.3. Active Malware and Exploits

**GitHub Policy:**
> "We do not allow content that... delivers or executes malware... or facilitates malicious activities."

**Compliance Stance:**
The repository contains no executable malware.

1.  **Cyber-Defense Context:** The repository references "cyber takeover" and "RF denial" solely as research domains in the `future_research.md` document. These references are survey-paper depth (describing the academic field).
2.  **No Exploit Code:** TALON-MESH does **not** contain exploit code, zero-day vulnerabilities for drone firmware, or decryption keys for encrypted drone links. The `perception_fusion` module deals with *detecting* signals, not *exploiting* them.
3.  **RF Passive Monitoring:** The repository permits the discussion of *passive* RF monitoring (listening). It strictly excludes *active* RF interference (jamming/spoofing) code, which is regulated under telecommunications law and violates GitHub’s policy on facilitating interference with critical infrastructure.

### 2.4. Site Access and Abuse

**GitHub Policy:**
> "You must not... access the Service or any part of it... via any automated or technical means... to send unsolicited communications."

**Compliance Stance:**
The repository does not utilize GitHub Actions or automation for purposes other than standard CI/CD (linting, testing code, building documentation). No automated scraping or bandwidth-abuse tools are included.

---

## 3. The "Firewall" Between Research and Implementation

To ensure long-term compliance, TALON-MESH enforces a strict architectural separation between the **Decision Layer** (Software/Algorithm) and the **Execution Layer** (Hardware/Effector).

### 3.1. What We Build (In Scope)
*   **Algorithms:** Optimization routines for multi-target assignment.
*   **Geometry:** Mathematical models for Line-of-Sight and coverage.
*   **Simulation:** Software that mimics the behavior of a defense system in a virtual environment.

### 3.2. What We Do Not Build (Out of Scope / ToS Restricted)
*   **Targeting Systems:** Code that outputs coordinates intended for a live fire-control system.
*   **Weapon Drivers:** Software that communicates with a physical weapon bus (e.g., MIL-STD-1553, RS-232 for fire control).
*   **Lethal Logic:** "Fire" commands or kill-chain execution scripts.

**Compliance Rationale:**
By keeping the "Decision Layer" abstract—where the output is a data structure in a simulation log, not a voltage signal to a servo—we remain within the realm of academic research (Protected Speech) and outside the realm of weapons proliferation (Restricted Content).

---

## 4. User Responsibilities and Contributions

While the repository maintainers enforce strict compliance, the open-source nature of the project requires user adherence.

### 4.1. No "Forking" for Weaponization
Users are prohibited from forking this repository to develop weaponized forks.
*   **Prohibited Action:** Forking TALON-MESH and writing a driver to control a real laser.
*   **Enforcement:** Such forks violate GitHub ToS and are subject to takedown by GitHub. They also violate the project's MIT license if they fail to retain the legal disclaimers.

### 4.2. Contribution Guidelines
All Pull Requests (PRs) are subject to the following checks:
1.  **Hardware Check:** Does the PR introduce hardware-specific drivers? -> **REJECT**.
2.  **Weapon Check:** Does the PR introduce logic to calculate lethal power densities? -> **REJECT**.
3.  **Export Check:** Does the PR introduce controlled technical data (ITAR)? -> **REJECT**.

---

## 5. Dual-Use Justification (Civilian Transfer)

GitHub's policies support the sharing of information that benefits the broader community. TALON-MESH justifies its existence and compliance through robust **Civilian Transfer**.

The algorithms developed for "Counter-UAS" are mathematically identical to those required for:
1.  **Multi-Robot Coordination:** Coordinating warehouse robots to avoid collisions and maximize efficiency.
2.  **Search and Rescue:** Optimizing the coverage of multiple drones searching for a lost hiker.
3.  **Agricultural Management:** Coordinating swarms of pollination or spraying drones.
4.  **Traffic Management:** Dynamic assignment of autonomous vehicles to passenger requests.

The repository explicitly develops and maintains an `examples/civilian/` directory to ensure this beneficial use case is primary and discoverable.

---

## 6. Disclaimer of Liability

TALON-MESH is provided "as is," without warranty of any kind, express or implied. The maintainers explicitly disclaim any liability for damages arising from the use, misuse, or inability to use the software.

The software is a **simulation tool**. It is not certified for use in safety-critical applications. Reliance on this software for real-world defense or safety scenarios is strictly prohibited without independent verification and appropriate regulatory licensing.

---

## 7. Reporting Violations

If you discover content within this repository that you believe violates GitHub’s Terms of Service or applicable laws, please open an issue labeled `[LEGAL-COMPLIANCE]` or contact the maintainers directly. We are committed to immediate remediation of any valid compliance concern.

---

## 8. Document Revision History

*   **v1.0:** Initial creation of ToS compliance document.
*   **v1.1 (Current):** Expanded to include detailed analysis of "Weapons and Explosives" policy, "Effector Abstraction" boundaries, and Civilian Transfer justification. Added specific restrictions on hardware drivers and fire-control logic.
