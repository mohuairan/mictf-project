<p align="center">
  <img src="assets/banner.png" alt="MICTF Banner" width="100%">
</p>

<h1 align="center">MICTF</h1>

<p align="center">
  <b>Morphology-Informed Contact Template Framework</b><br>
  Zero-shot cross-embodiment dexterous hand teleoperation via contact primitives
</p>

<p align="center">
  <a href="https://youtu.be/rv6dafFKCII"><img src="https://img.shields.io/badge/Demo-YouTube-red?logo=youtube" alt="YouTube Demo"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-blue.svg" alt="License: MIT"></a>
  <img src="https://img.shields.io/badge/Python-3.10+-3776AB?logo=python&logoColor=white" alt="Python 3.10+">
  <img src="https://img.shields.io/badge/MuJoCo-3.x-orange" alt="MuJoCo 3.x">
  <img src="https://img.shields.io/badge/Status-Paper_in_Preparation-yellow" alt="Status">
</p>

<p align="center">
  <b>Language:</b> <b>English</b> | <a href="README.zh-CN.md">简体中文</a>
</p>

---

## ✨ Highlights

- **Zero-shot hand import** — Drop in any MJCF hand model; contact templates are auto-generated from morphology and transmission structure. No per-hand tuning or demonstration data required.
- **Explicit contact semantics** — Replaces implicit geometric error with structured contact primitives (precision pinch, multi-finger pinch, closure), enabling cross-embodiment transfer of manipulation intent.
- **Bayesian intent inference** — HMM-based posterior estimation with hysteresis and dwell-time constraints. Noise sensitivity tests show **0% template flip rate** across all tested hands.
- **Closed-chain contact projection** — Jointly solves gap closure, normal alignment, and smoothness residuals. Reduces mean contact distance by **74–82%** vs. DexPilot-style baselines on heterogeneous hands.
- **Dual execution paths** — Joint-space closed-chain solver for direct-drive hands; actuator-space proxy solver for coupled/underactuated mechanisms.

---

## 🔍 Motivation

Existing retargeting methods (DexPilot, Joint Copy, Fingertip IK) map **where fingertips should go**, but none explicitly represent **what contact relation should be maintained**. This gap causes three failure modes during cross-embodiment teleoperation:

| Failure Mode | Root Cause |
|:---|:---|
| Thumb base oscillation during open ↔ close transitions | Null-space redundancy in position-only IK |
| Contact normal flipping | No explicit pad-normal alignment constraint |
| Silent contact loss during post-acquisition motion | No closed-chain maintenance mechanism |

MICTF addresses these by lifting the retargeting intermediate representation from geometric trajectories to **contact primitives with physical semantics**.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    INPUT LAYER                          │
│  ┌──────────────────┐    ┌───────────────────────────┐  │
│  │ Quest 3 / WebXR  │    │ Robot Hand Model (MJCF)   │  │
│  │ Hand Tracking     │    │                           │  │
│  └────────┬─────────┘    └──────────┬────────────────┘  │
│           │                         │                    │
│           ▼                         ▼                    │
│  ┌─────────────────┐    ┌──────────────────────────┐    │
│  │ Feature Extract  │    │ Morphology & Transmission│    │
│  │ • pinch distance │    │ Parser (Algorithm 1)     │    │
│  │ • curl angles    │    │ • finger chains & roles  │    │
│  │ • approach vel   │    │ • contact frames (SVD)   │    │
│  │ • contact scores │    │ • actuator/tendon specs  │    │
│  └────────┬─────────┘    └──────────┬────────────────┘  │
└───────────┼──────────────────────────┼──────────────────┘
            │                          │
            │                          ▼
            │               ┌─────────────────────┐
            │               │ Template Enumeration │
            │               │ (Algorithm 2)        │
            │               │ open / pinch / multi  │
            │               │ / closure + canonical │
            │               └──────────┬──────────┘
            │                          │
            ▼                          ▼
┌───────────────────────────────────────────────────────┐
│              INTENT INFERENCE LAYER                   │
│  ┌─────────────────────────────────────────────────┐  │
│  │ Bayesian Template Filter (HMM, Algorithm 3)     │  │
│  │ • energy-based observation likelihood           │  │
│  │ • FK-based transition costs                     │  │
│  │ • posterior hysteresis + min dwell constraint    │  │
│  └──────────────────────┬──────────────────────────┘  │
└─────────────────────────┼─────────────────────────────┘
                          │
                          ▼
┌───────────────────────────────────────────────────────┐
│              EXECUTION LAYER (Algorithm 4)            │
│                                                       │
│  ┌─────────────────┐      ┌────────────────────────┐  │
│  │ Direct Path     │      │ Proxy Path             │  │
│  │ (Allegro, Wuji, │      │ (Amazing Hand, etc.)   │  │
│  │  LEAP, ORCA,    │      │                        │  │
│  │  Sharpa)        │      │ • actuator probing     │  │
│  │                 │      │ • finite-diff Jacobian │  │
│  │ Bounded NLS:    │      │ • damped LS in         │  │
│  │ • gap closure   │      │   actuator space       │  │
│  │ • normal align  │      │ • pair gap solver      │  │
│  │ • smoothness    │      │                        │  │
│  │ • surface UV    │      │                        │  │
│  └────────┬────────┘      └───────────┬────────────┘  │
│           │                           │                │
│           ▼                           ▼                │
│  ┌─────────────────────────────────────────────────┐  │
│  │ Execution State Machine                         │  │
│  │ approach → acquire → track/hold → fallback      │  │
│  └──────────────────────┬──────────────────────────┘  │
└─────────────────────────┼─────────────────────────────┘
                          │
                          ▼
              Joint / Actuator Command
                → MuJoCo / Hardware
```

---

## 📊 Results

Evaluated on replayed Quest 3 / WebXR hand-tracking logs in MuJoCo simulation. All methods replay the **same operator input** per hand; differences are purely algorithmic.

### Cross-Method Comparison

| Hand | Method | Mean Distance (mm) | p95 Distance (mm) | 30mm Success Rate | Hold Loss Rate |
|:---|:---|---:|---:|---:|---:|
| **Wuji Right** | **MICTF** | **14.75** | **36.12** | **0.947** | **0.076** |
| Wuji Right | DexPilot-style | 57.56 | 81.34 | 0.059 | 0.827 |
| Wuji Right | Joint Copy | 64.85 | 84.33 | 0.009 | 0.976 |
| Wuji Right | Fingertip IK | 73.98 | 93.90 | 0.015 | 0.993 |
| **Allegro Right** | **MICTF** | **14.65** | **35.44** | **0.937** | **0.000** |
| Allegro Right | DexPilot-style | 80.61 | 91.40 | 0.000 | 1.000 |
| Allegro Right | Joint Copy | 88.64 | 142.60 | 0.023 | 0.938 |
| Allegro Right | Fingertip IK | 102.55 | 166.06 | 0.003 | 0.991 |

### Ablation Study (Key Findings)

| Ablated Module | Effect on Wuji Right | Effect on Allegro Right |
|:---|:---|:---|
| Remove closed-chain projection | Distance: 14.75 → **76.46 mm** | Distance: 14.65 → **100.69 mm** |
| Remove temporal filtering | Hold switches: 0.619 → 0.762 | Posterior confidence drops |
| Fixed contact point (no surface UV) | Solver success: 0.974 → **0.000** | Solver success: 0.990 → 0.465 |

---

## 🤖 Supported Hands

Unified import pipeline — no per-hand code or configuration:

| Hand | DOF | Fingers | Templates | Execution Path | Validation Level |
|:---|---:|---:|---:|:---|:---|
| JackHand | 10 | 3 | 9 | Joint space | Quest 3 replay |
| Allegro Right | 16 | 4 | 12 | Joint space | Quest 3 replay |
| Wuji Right/Left | 20 | 5 | 15 | Joint space | Quest 3 replay |
| LEAP Right | 16 | 4 | 12 | Joint space | Import + solver sweep |
| ORCA Right v1 | 16 | 5 | 15 | Joint space | Import + solver sweep |
| Sharpa Wave | 22 | 5 | 15 | Joint space | Import + solver sweep |
| Amazing Hand | 8 act. / 32 jnt. | 4 | 12 | Actuator proxy | VR replay (162.6s) |
| Robotiq 2F-85 | 2 | 2 | — | Diagnostic only | Non-opposable topology |

> **ORCA Right v1** and **Sharpa Wave** were imported successfully with **zero adaptation** — no code changes were made for these hands during development.

---

## 🎬 Demo

**Four-hand side-by-side** (Wuji · Allegro · ORCA · Sharpa) — closed-chain contact maintenance under replayed operator motion, 1.5× speed:

<p align="center">
  <img src="assets/multihand_demo.gif" alt="MICTF four-hand closed-chain contact demo" width="90%">
</p>
<p align="center"><em>Each panel: same replayed Quest 3 input, different morphology. Pad-level contact is maintained across all four hands after acquisition.</em></p>

Single-clip demo:

<p align="center">
  <a href="https://youtu.be/rv6dafFKCII">
    <img src="https://img.youtube.com/vi/rv6dafFKCII/maxresdefault.jpg" alt="MICTF Demo" width="70%">
  </a>
  <br>
  <a href="https://youtu.be/rv6dafFKCII">▶ Watch on YouTube</a>
</p>

---

## ⚠️ Limitations

- **Not a grasp planner** — No force closure, friction cone, or object stability guarantees.
- **Geometric contact only** — No tactile feedback or contact force estimation in the current loop.
- **Static contact primitives** — Covers contact acquisition and maintenance; in-hand manipulation (rolling, finger gaiting) is future work.
- **Simulation-validated** — Hardware experiments and user studies are planned but not yet completed.
- **No training data required** — Runtime depends entirely on morphology metadata and geometric constraints.

---

## 🗺️ Roadmap

- [x] Morphology import + automatic contact frame extraction
- [x] Template enumeration from hand topology
- [x] Bayesian intent filter with FK-based transition costs
- [x] Closed-chain post-contact projection with surface UV
- [x] Pre-contact guided trajectory generation
- [x] Actuator-space proxy solver for coupled/underactuated hands
- [x] Quest 3 replay evaluation (JackHand, Wuji, Allegro)
- [x] Zero-adaptation import validation (ORCA, Sharpa Wave)
- [x] Multi-hand simultaneous replay (6 hands, same trajectory)
- [ ] Real-hardware closed-loop validation
- [ ] Multi-input device support (data gloves)
- [ ] Object-in-hand task-level evaluation
- [ ] User study (NASA-TLX, task completion time)

---

## 💻 Code

Source code and experiment pipelines are in a private repository, to be released alongside the paper. This page tracks public project status.

---

## 📖 Citation

```bibtex
@article{mo2026mictf,
  title   = {MICTF: Morphology-Informed Contact Template Framework for
             Cross-Embodiment Dexterous Hand Teleoperation},
  author  = {Mo, Huairan},
  journal = {IEEE Robotics and Automation Letters (in preparation)},
  year    = {2026}
}
```

---

## 📄 License

Documentation and assets: [MIT License](LICENSE). Source code license will be announced with release.
