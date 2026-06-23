# MICTF — Morphology-Informed Contact Template Framework

**Online, data-free retargeting that maintains pad-level contact *after* a precision pinch has been acquired.**

🌐 **Language / 语言:** **English** · [简体中文](README.zh-CN.md)

> *Morphology-Informed Contact Templates for Online Contact-Maintaining Dexterous Retargeting*
> 📄 **Status:** In preparation for IEEE RA-L.
> 💬 Discussion & questions: [GitHub Discussions](../../discussions) · Issues: [GitHub Issues](../../issues)

---

## Why this work exists

Most dexterous retargeting methods — DexPilot-style vector tracking, joint copy, fingertip IK — answer the question **"where should the fingertips go?"** They do this well in free space. But once two pads have touched and the operator keeps moving, these representations have nothing to say about **how the surfaces should keep meeting**. The result is a familiar failure: the fingertips stay near their vector targets, the thumb pad rotates freely about the thumb–fingertip axis, and the contact silently opens up.

MICTF starts from a different question: **what contact should the robot maintain, and what morphology-specific geometry enforces it?**

## The core insight

A 4-DOF robot thumb that is only constrained by a 3D thumb-tip vector has one unconstrained DOF — rotation of the pad about the thumb–fingertip axis. DexPilot's smoothness regularizer parks this free DOF near the seed; it provides no contact-geometric guidance. Follow-up systems like AnyTeleop add orientation regularization to *mitigate* the ambiguity.

MICTF **resolves** it by consuming that free DOF with task geometry: a pad-normal alignment residual that forces the thumb pad to face the opposing surface. Kinematic redundancy — usually a nuisance to be regularized away — becomes the mechanism that *maintains* contact.

This reframes the contribution. The value is not a faster solver or a richer latent; it is recognizing that **post-contact maintenance is a distinct regime** from approach, and that the geometry of the hand's own pads — not a posture prior — is the right thing to constrain.

## What MICTF is

MICTF is an online, data-free retargeting stack for **position-controlled** dexterous hands. It inserts a morphology-derived **contact template** between operator intent and robot execution:

| Layer | Role |
|---|---|
| **Morphology import** | Parse an MJCF hand, infer finger chains, auto-extract pad contact frames from mesh patches, derive per-hand runtime tolerances. |
| **Contact templates** | Discrete, executable objects (open / precision pinch / multi-pinch / closure) carrying contact roles, local pad frames, and solver tolerances. |
| **Bayesian intent filter** | A hidden-Markov forward update selects the active template from Quest 3/WebXR observations, with transitions priced by geometric proximity in the robot's own workspace. |
| **Closed-chain projection** | A bounded nonlinear least-squares solver that maintains the selected contact while the operator keeps moving, with a pad-normal residual that resolves thumb redundancy through contact geometry. |

A runtime state machine explicitly separates **approach** (free-space tracking toward a canonical pose) from **latched hold** (constraint maintenance under moving input). This separation is the structural reason the method does not lose contact the moment the operator drifts from the acquisition pose.

## What it is not

To set honest expectations:

- **Not a grasping or force-closure method.** Maintaining a small pad gap and aligned local frames does not by itself establish wrench-space closure or stable object manipulation.
- **Not for coupled / underactuated hands (yet).** Tendon-driven or synergy-coupled transmissions need an actuator-space execution layer that the present framework treats as future work.
- **Not a learned policy.** There is no training, no demonstrations in the target domain, no simulation distribution. The runtime source of truth is morphology metadata and geometric constraints.

## Supported hands (offline import)

The same import + solver path covers heterogeneous position-controlled hands:

- **Allegro Right**, **Wuji Right** — full VR replay validation
- **LEAP Hand**, **ORCA Right v1**, **Sharpa Wave** — offline import and solver checks
- **Shadow** (tendon topology diagnostic only), **Robotiq 2F-85** (non-opposable diagnostic)

Different hand geometries produce different template counts and runtime scales from the *same* importer — no per-hand hard-coding.

## Results at a glance

Across two replayed hands, MICTF maintains post-contact hold-motion gap at the **single-digit-millimeter** level, while DexPilot-style vector retargeting, joint copy, and fingertip IK drift to **tens of millimeters** or fail to acquire tight contact. Disabling the closed-chain projection collapses performance to baseline levels, confirming the projection layer — not the canonical pose lookup — is doing the work.

Detailed numbers, ablations, and per-hand solver diagnostics will accompany the paper.

## Roadmap

- [x] Morphology import + automatic contact-frame extraction
- [x] Bayesian template intent filter
- [x] Closed-chain contact projection (joint-space + surface extension)
- [x] Per-hand Quest 3 replay baselines
- [ ] Real-hardware validation
- [ ] User study (NASA-TLX, task completion time)
- [ ] Actuator-space execution for coupled/underactuated hands

## Repository

The MICTF research repository (source code, experiment pipelines, and paper drafts) is currently private and will be released alongside the paper. This page tracks the project's public status.

## Citation

A BibTeX entry will be provided upon publication. For now, please reference this page or open a Discussion if you need to cite the work in progress.

## License

Project description and documentation: see [`LICENSE`](LICENSE). Source code release terms will be announced with the code.
