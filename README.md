<p align="center">
  <img src="assets/banner.png" alt="MICTF — Morphology-Informed Contact Template Framework" width="100%">
</p>

# MICTF — Morphology-Informed Contact Template Framework

**Online, data-free retargeting that maintains pad-level contact *after* a precision pinch has been acquired.**

🌐 **Language / 语言:** **English** · [简体中文](README.zh-CN.md)

> *Morphology-Informed Contact Templates for Online Contact-Maintaining Dexterous Retargeting*
> 📄 **Status:** In preparation for IEEE RA-L.
> 💬 Discussion & questions: [GitHub Discussions](../../discussions) · Issues: [GitHub Issues](../../issues)

---

## Why this work exists

Most dexterous retargeting methods — DexPilot-style vector tracking, joint copy, fingertip IK — answer the question **"where should the fingertips go?"** They work well in free space. But once two pads have touched and the operator keeps moving, none of these representations has an explicit mechanism to **keep the surfaces meeting**. The result: the fingertips stay near their vector targets, but the contact silently opens up.

This happens for a structural reason. A 4-DOF robot thumb constrained only by a 3D tip-position vector has one unconstrained DOF — rotation of the pad about the thumb–fingertip axis. DexPilot's smoothness regularizer parks this free DOF near the seed; it provides no contact-geometric guidance. Follow-up systems (AnyTeleop, Bunny-VisionPro) add orientation regularization to *mitigate* the ambiguity.

MICTF starts from a different question: **what contact should the robot maintain, and what morphology-specific geometry enforces it?**

## Core insight

> Any retargeting map from a low-dimensional task target to a redundant kinematic chain leaves a null space. Regularization-based approaches fill it with posture priors. **MICTF fills it with the task's own contact geometry**, so that under-determined DOF serve contact maintenance rather than drifting toward an arbitrary default.

Concretely, MICTF **resolves** the thumb's free DOF by consuming it with a pad-normal alignment residual that forces the thumb pad to face the opposing surface. Kinematic redundancy — usually a nuisance to be regularized away — becomes the mechanism that *maintains* contact.

This shifts the contribution. The value is not a faster solver or a richer latent, but recognizing that:

1. **Post-contact maintenance is a distinct operating regime** from approach — retargeters that apply the same free-space cost in both regimes lose contact the moment the operator moves.
2. **The hand's own pad geometry — not a posture prior — is the right constraint** for resolving redundancy during that regime.
3. **The contact point itself can be an optimization variable** — MICTF's surface extension co-optimizes joint angles and pad-surface UV coordinates, converting a hard feasibility problem into a softer fit over the joint-surface product space.

## System architecture

```
Quest 3 hand joints
        │
        ▼
┌─────────────────────┐
│  Palm-frame obs.    │  Compute thumb–finger distances, curl, contact-location scores
└────────┬────────────┘
         ▼
┌─────────────────────┐
│  Bayesian template  │  HMM forward update; transitions priced by FK-based
│  intent filter      │  geometric proximity in the robot's own workspace
└────────┬────────────┘
         ▼
┌─────────────────────┐     ┌──────────────────────┐
│  Execution state    │────▶│  Approach mode:      │  Free-space tracking toward canonical pose
│  machine            │     │  Hold mode:          │  Constraint maintenance under moving input
└────────┬────────────┘     └──────────────────────┘
         ▼
┌─────────────────────┐
│  Closed-chain       │  Bounded NLS with pad-normal + gap + tracking residuals;
│  projection solver  │  surface UV co-optimization when metadata available
└────────┬────────────┘
         ▼
  Bounded joint command → MuJoCo / hardware
```

| Layer | Role |
|---|---|
| **Morphology import** | Parse MJCF, infer finger chains, auto-extract pad contact frames from mesh patches via SVD, derive per-hand tolerances. |
| **Contact templates** | Discrete executable objects (open / precision pinch / multi-pinch / closure) carrying contact roles, pad frames, and solver tolerances. |
| **Bayesian intent filter** | HMM forward update over templates; transition costs derived from geometric workspace proximity, not hand-tuned penalties. |
| **Closed-chain projection** | Bounded NLS solver that maintains the selected contact; pad-normal residual resolves thumb redundancy through contact geometry rather than posture priors. |

## What it is not

- **Not a grasping or force-closure method.** Small pad gap + aligned frames ≠ wrench-space closure or stable object manipulation.
- **Not for coupled / underactuated hands (yet).** Tendon-driven or synergy-coupled transmissions need an actuator-space execution layer (future work).
- **Not a learned policy.** No training, no target-domain demonstrations. Runtime source of truth is morphology metadata and geometric constraints.

## Results

Evaluated across **5 hands × 3 sessions per hand** (15 VR replay experiments), all from real Quest 3/WebXR logs replayed in MuJoCo:

| Hand | MICTF gap mean | Best baseline gap | MICTF Gap@10 | MICTF Acq. |
|------|---------------|-------------------|--------------|------------|
| Wuji Right | **8.3 mm** | 40.3 mm (Joint copy) | 0.788 | 0.722 |
| Allegro Right | **15.4 mm** | 51.0 mm (Joint copy) | 0.803 | 0.667 |
| LEAP Right | **19.2 mm** | 44.4 mm (DexPilot) | 0.484 | 0.533 |
| ORCA Right v1 | **8.9 mm** | 23.6 mm (Joint copy) | 0.841 | 0.278 |
| Sharpa Wave | **13.5 mm** | 39.7 mm (Joint copy) | 0.563 | 0.278 |

Key observations:
- **DexPilot wins on its own metric** (vector error): 22–48 mm vs MICTF's 59–107 mm. This is expected — MICTF trades vector fidelity for pad alignment.
- **MICTF wins on contact maintenance** (gap, hold-gap): 2–4× lower gap across all 5 hands.
- **No baseline acquires stable contact** (Acq. = 0.000 for DexPilot/IK on all hands). MICTF is the only method that transitions into held contact.
- **Disabling the closed-chain solver** collapses gap to baseline levels, confirming the projection layer — not template selection alone — is doing the work.

The trade-off is explicit and structural: DexPilot achieves better endpoint vector matching; MICTF achieves pad-level contact at the cost of visual similarity to the operator's hand posture.

## Supported hands

The same import path covers heterogeneous position-controlled hands — no per-hand code:

| Hand | DOF | Fingers | Templates | Validation |
|------|-----|---------|-----------|-----------|
| Allegro Right | 16 | 4 | 9 | VR replay (3 sessions) |
| Wuji Right | 20 | 5 | 12 | VR replay (3 sessions) |
| LEAP Right | 16 | 4 | 9 | VR replay (3 sessions) |
| ORCA Right v1 | 16 | 5 | 12 | VR replay (3 sessions) |
| Sharpa Wave | 22 | 5 | 12 | VR replay (3 sessions) |
| Shadow Right | 22 | 5 | 12 | Tendon topology diagnostic |
| Robotiq 2F-85 | 8 | 4 | 0 | Non-opposable diagnostic |

## Demonstrations

| Segment | What it shows | Link |
|---|---|---|
| Closed-chain hold | Pad-level contact maintained while the operator continues to move after acquisition | [▶ Watch on YouTube](https://youtu.be/rv6dafFKCII) |

<p align="center">
  <a href="https://youtu.be/rv6dafFKCII">
    <img src="https://img.youtube.com/vi/rv6dafFKCII/maxresdefault.jpg" alt="MICTF closed-chain hold demo" width="70%">
  </a>
</p>

## Roadmap

- [x] Morphology import + automatic contact-frame extraction
- [x] Bayesian template intent filter with FK-based transition costs
- [x] Closed-chain contact projection (joint-space + surface UV co-optimization)
- [x] Per-hand Quest 3 replay baselines (5 hands × 3 sessions)
- [ ] Real-hardware validation
- [ ] User study (NASA-TLX, task completion time)
- [ ] Actuator-space execution for coupled/underactuated hands

## Repository

The MICTF research repository (source code, experiment pipelines, and paper drafts) is currently private and will be released alongside the paper. This page tracks the project's public status.

## Citation

A BibTeX entry will be provided upon publication. For now, please reference this page or open a Discussion if you need to cite the work in progress.

## License

Project description and documentation: see [`LICENSE`](LICENSE). Source code release terms will be announced with the code.
