<p align="center">
  <img src="assets/banner.png" alt="MICTF Banner" width="100%">
</p>

<h1 align="center">MICTF</h1>

<p align="center">
  <b>Morphology-Informed Contact Template Framework</b><br>
  Online contact-maintaining retargeting for heterogeneous dexterous hands
</p>

<p align="center">
  <a href="https://mohuairan.github.io/mictf-project/"><img src="https://img.shields.io/badge/Project-Page-2ec4b6?logo=github&logoColor=white" alt="Project Page"></a>
  <a href="https://youtu.be/rv6dafFKCII"><img src="https://img.shields.io/badge/Demo-YouTube-red?logo=youtube" alt="YouTube Demo"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-blue.svg" alt="License: MIT"></a>
  <img src="https://img.shields.io/badge/Python-3.10+-3776AB?logo=python&logoColor=white" alt="Python 3.10+">
  <img src="https://img.shields.io/badge/MuJoCo-3.x-orange" alt="MuJoCo 3.8">
  <img src="https://img.shields.io/badge/Status-Paper_in_Preparation-yellow" alt="Status">
</p>

<p align="center">
  <b>Language:</b> <b>English</b> | <a href="README.zh-CN.md">简体中文</a>
</p>

---

## Overview

MICTF is an online, data-free retargeting method for maintaining pad-level contact *after* a precision contact has been acquired. It introduces morphology-derived **contact templates** as an intermediate representation between operator intent and robot execution: each template maps a robot hand model to contact roles, local pad frames, and post-contact kinematic constraints.

The long-term framework vision is zero-shot cross-embodiment teleoperation via contact primitives. This page reflects the current state, which focuses on position-controlled dexterous hands and online post-contact maintenance.

<p align="center">
  <img src="assets\system_overview.png" alt="System Overview" width="100%">
</p>

## Why MICTF

Most online retargeting methods — DexPilot-style vector tracking, joint copy, fingertip IK, AnyTeleop-style pipelines — answer "where should the fingertips go?" They work well in free space. But once two pads have made contact and the operator keeps moving, these representations have nothing to say about how the surfaces should keep meeting. The contact silently opens up.

(Contact-rich optimization methods — contact-invariant control, relaxed-IK with contact constraints — *are* contact-aware, but operate as offline planners or offline trajectory optimizers, not as online retargeting layers that maintain contact under live operator motion. MICTF targets the latter regime.)

This is structural. A 4-DOF robot thumb constrained only by a 3D thumb-tip vector has one unconstrained DOF: pad rotation about the thumb–fingertip axis. DexPilot's smoothness regularizer parks this free DOF near the seed but provides no contact-geometric guidance. MICTF instead consumes that free DOF with task geometry: a pad-normal alignment residual that forces the thumb pad to face the opposing surface. Kinematic redundancy — usually a nuisance to be regularized away — becomes the mechanism that *maintains* contact.

## Architecture

| Layer | Role |
|---|---|
| **Morphology import** | Parse an MJCF hand, infer finger chains, auto-extract pad contact frames from mesh patches, derive per-hand runtime tolerances. |
| **Contact templates** | Discrete, executable objects (open / precision pinch / multi-pinch / closure) carrying contact roles, local pad frames, and solver tolerances. |
| **Bayesian intent filter** | A hidden-Markov forward update selects the active template from Quest 3/WebXR observations, with transitions priced by geometric proximity in the robot's own workspace. |
| **Closed-chain projection** | A bounded nonlinear least-squares solver maintains the selected contact under moving operator input; a pad-normal residual resolves thumb redundancy through contact geometry. |

The execution backend currently assumes independently commandable position-control joints. An actuator-space proxy path for coupled/underactuated transmissions (e.g., tendon-driven, synergy-coupled hands) is **in development** and is not part of the current evaluation.

## Results

Evaluated on real Quest 3/WebXR logs replayed in MuJoCo. Every method variant for a given hand replays the same JSONL input, so method differences are purely algorithmic.

**Post-contact hold gap (primary metric):**

| Hand | MICTF | DexPilot-style | Joint Copy | Fingertip IK |
|---|---|---|---|---|
| Wuji Right | **8.3 mm** | 40.3 mm | 42.4 mm | 45.8 mm |
| Allegro Right | **15.4 mm** | 51.0 mm | 48.2 mm | 55.1 mm |
| LEAP Right | **19.2 mm** | 44.4 mm | 47.3 mm | 49.0 mm |
| ORCA Right v1 | **8.9 mm** | 23.6 mm | 26.1 mm | 28.4 mm |
| Sharpa Wave | **13.5 mm** | 39.7 mm | 41.2 mm | 43.6 mm |

Mean task-frame gap across five hands: **MICTF 8.3–19.2 mm** versus **27.7–76.6 mm** for DexPilot-style and **23.1–55.7 mm** for joint copy. No baseline acquires stable contact (Acquire = 0.000 on all hands); MICTF transitions into held contact on every hand (Acquire 0.222–0.778). DexPilot achieves lower vector error (33.9–47.6 mm vs. MICTF's 54.7–106.8 mm), which is expected: MICTF trades endpoint-vector fidelity for pad alignment. Disabling the closed-chain solver collapses task distance to baseline levels, confirming that the projection layer — not template selection alone — produces the gap reduction.

Full numbers, ablations, and per-hand solver diagnostics will accompany the paper.

## Supported Hands

The same import pipeline covers heterogeneous position-controlled hands — no per-hand hard-coding:

| Hand | DoF | Fingers | Templates | Validation |
|---|---|---|---|---|
| Wuji Right | 20 | 5 | 12 | Quest 3 replay (full baseline) |
| Allegro Right | 16 | 4 | 9 | Quest 3 replay (full baseline) |
| LEAP Right | 16 | 4 | 9 | Quest 3 replay (full baseline) |
| ORCA Right v1 | 16 | 5 | 12 | Quest 3 replay (full baseline) |
| Sharpa Wave | 22 | 5 | 12 | Quest 3 replay (full baseline) |
| Shadow Right | 22 | 5 | 12 | Tendon topology diagnostic |
| Robotiq 2F-85 | 1 | 2 | — | Non-opposable diagnostic |

> **ORCA Right v1** and **Sharpa Wave** were imported successfully with **zero adaptation** — no code changes were made for these hands during development.

## Demo

**Four-hand side-by-side** (Wuji · Allegro · ORCA · Sharpa) — closed-chain contact maintenance under replayed operator motion, 1.5× speed:

<p align="center">
  <img src="assets/multihand_demo.gif" alt="MICTF four-hand closed-chain contact demo" width="90%">
</p>
<p align="center"><em>Each panel: same replayed Quest 3 input, different morphology. Pad-level contact is maintained across all four hands after acquisition.</em></p>

**One trajectory, five hands** — the same Quest 3 hand-tracking replay mapped onto five different morphologies (Wuji · Allegro · LEAP · ORCA · Sharpa), showing that MICTF establishes stable contact across all of them without per-hand tuning:

<p align="center">
  <a href="https://youtu.be/rv6dafFKCII">
    <img src="https://img.youtube.com/vi/rv6dafFKCII/maxresdefault.jpg" alt="MICTF one-trajectory-five-hands demo" width="70%">
  </a>
  <br>
  <a href="https://youtu.be/rv6dafFKCII">Watch on YouTube</a>
</p>

## Scope and Limitations

- **Not a grasp planner.** Maintaining a small pad gap and aligned local frames does not by itself establish wrench-space closure or stable object manipulation.
- **Geometric contact only.** No tactile feedback or contact force estimation in the current loop.
- **Position-controlled hands.** The execution backend assumes independently commandable joints; an actuator-space path for coupled/underactuated transmissions is in development.
- **Simulation-validated.** Hardware experiments and user studies are planned but not yet completed.
- **No training data.** Runtime depends entirely on morphology metadata and geometric constraints.

## Roadmap

- [x] Morphology import + automatic contact-frame extraction
- [x] Contact template enumeration from hand topology
- [x] Bayesian intent filter with FK-based transition costs
- [x] Closed-chain post-contact projection with surface UV
- [x] Five-hand Quest 3 replay baselines
- [ ] Actuator-space proxy path for coupled/underactuated hands
- [ ] Real-hardware closed-loop validation
- [ ] User study (NASA-TLX, task completion time)
- [ ] Tactile contact confidence and richer contact models

## Code

Source code and experiment pipelines are in a private repository, to be released alongside the paper. This page tracks public project status.

## Citation

A BibTeX entry will be provided upon publication. For now, please reference this page or open a Discussion if you need to cite the work in progress.

## License

Documentation and assets: [MIT License](LICENSE). Source code license will be announced with release.
