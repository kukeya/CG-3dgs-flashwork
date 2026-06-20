# RoboSplat action-effect consistency — diagnostic results

Realizes the experiment in [docs/robosplat-dig-exp.md](robosplat-dig-exp.md) via the
**ManiSkill physics-replay** path (the document's own Stage-1 fallback), with one
correction forced by what RoboSplat actually is.

## Feasibility finding (read this first)

The document assumes RoboSplat demos can be re-simulated under physics. **They
cannot, as generated.** RoboSplat is a *kinematic* data generator:

- The carrot object exists **only as 3D Gaussians** (`carrot.ply`). There is **no
  collision mesh, no URDF, no rigid body** for it — collision meshes ship only for
  the Panda robot.
- Demos are produced by **motion planning (mplib) + kinematic Gaussian
  compositing**. When the gripper "grasps," the code simply **translates the object
  Gaussians by the end-effector delta**
  ([generate_demo.py:134-140](../baselines/RoboSplat/data_aug/data_aug/generate_demo.py)).
  The object follows the gripper *by construction*; there is no contact, friction,
  or grasp that can physically fail.
- RoboSplat also **re-plans the gripper trajectory for each augmented object pose**
  (the grasp key-frame is offset by the object displacement). So object and gripper
  always move together.

Consequence: replaying a RoboSplat action in a physics sim where the object is
where the demo assumes will **always succeed** — there is nothing to fail. This is
the document's **No-Go #3** ("object-pose augmentation 几乎全部成功") for the naive
H2/H3 hypotheses, and we confirm it (below, nominal = 100%).

The deeper thesis — *visual/kinematic plausibility ≠ action-effect correctness* —
is still real and measurable, and it is exactly this project's verification gap.
We measure it the honest way: RoboSplat assigns a **kinematic success label of 1.0**
to every generated demo regardless of grasp soundness. We replay each demo's action
against a **physical** object whose pose is perturbed by ε (the perception / pose
error any real deployment has) and measure **physical** success.

## Method

We rebuild RoboSplat's exact sapien-PhysX scene + Panda robot + controller +
timestep ([replay_env.py](../src/robosplat_consistency/replay_env.py)) — the same
physics backend (sapien 3.0.3) ManiSkill uses — and **add** a physical rigid-body
object (a box approximating the carrot's grasped cross-section, 3.2×3.2×6.0 cm,
20 g, μ=2.0) plus a ground plane. We then replay the generated joint-space action
trajectory under PhysX and score grasp/lift from the object's true z-trajectory and
robot–object contacts.

- **Stage-1 sanity gate:** at zero offset the replayed action lifts the physical
  box 0.150 m (z 0.030→0.180), matching the source demo's lift target exactly →
  the replay pipeline is faithful (no frame/controller mismatch).
- **Experiment A — nominal grid (Stage 3 as written):** replay all 100 pick_100
  demos with the object at the pose each demo assumes.
- **Experiment B — pose-perturbation robustness (the verification gap):** for a
  center demo and two corner demos, replay the action with the object offset by ε
  over a 13×13 grid (±6 cm in x,y). RoboSplat's label stays 1.0 everywhere.

## Results

`outputs/robosplat_consistency/all_replay_metrics.jsonl` (607 rollouts);
`tables/table_robosplat_consistency.csv`, `tables/table_robosplat_perturb_by_radius.csv`.

| experiment | n | physical success | contact rate | mean max lift |
| --- | --- | --- | --- | --- |
| nominal grid (object where demo assumes) | 100 | **1.000** | 1.00 | 0.150 m |
| ε-perturbation (object pose error) | 507 | **0.053** | 1.00 | 0.012 m |

**Physical success vs object pose error |ε|** (RoboSplat kinematic label = 1.0 for all):

| |ε| (cm) | 0.5 | 1.6 | 2.7 | 3.7 | 4.8 | 5.8 | 6.9 | 8.0 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| physical success | 0.73 | 0.25 | 0.13 | 0.01 | 0.00 | 0.00 | 0.00 | 0.00 |
| RoboSplat label | 1.0 | 1.0 | 1.0 | 1.0 | 1.0 | 1.0 | 1.0 | 1.0 |

- `figures/robosplat_nominal_success_heatmap.png` — uniform green (100%): object-pose
  augmentation **is** action-consistent *when the object is where the demo assumes*.
- `figures/robosplat_success_vs_offset.png` — the headline: physical success decays
  from ~0.73 (sub-cm) to ~0 (>4 cm) while the kinematic label stays flat at 1.0.
- `figures/robosplat_perturb_success_heatmap_demo52.png` — a small success island
  around the assumed pose (tilted along the gripper approach axis); everywhere else
  the demo is kinematically "successful" but physically fails.
- `figures/robosplat_failure_taxonomy.png` — failures are **weak_grasp** (480/507):
  the fingers contact the box but it slips out, not random misses. The gripper opening
  at grasp is 2.9 cm vs a 3.2 cm object, so ~1.5 cm lateral error pushes a finger off
  the object — a physically sensible, monotonic tolerance.

## Go / No-Go (per document §8)

**Mixed, and informative.** Against the document's literal criteria:
- Source demo replays (sanity gate ✅) and the pipeline is trustworthy.
- Object-pose augmentation does **not** fail when the object is at the assumed pose
  (document No-Go #3) — because RoboSplat re-plans per pose. The naive
  "augmentation breaks consistency" framing is **not** supported.
- But the **action-effect verification gap is real and quantified**: a 1.0 kinematic
  success label coexists with physical success that collapses beyond ~1.5 cm of pose
  error. visually/kinematically plausible ≠ physically correct.

**Recommended framing for the paper** (consistent with CLAUDE.md / plan §十一 "次强"):

> RoboSplat-style 3DGS augmentation produces demos whose success is asserted
> *kinematically*; the generated label carries no guarantee of physical
> action-effect correctness. Under realistic object-pose error the kinematically
> "successful" demo fails in physics, motivating a **test-time action-effect
> verifier** rather than trusting generation-time labels.

## Caveats (load-bearing, per CLAUDE.md)

- The physical object is a **box proxy**, not the carrot's true geometry (RoboSplat
  ships no object collision mesh). Absolute tolerances depend on this proxy's
  size/friction; the **qualitative decoupling** (flat 1.0 label vs decaying physical
  success) does not. We do not claim real-robot numbers.
- The grasp is a position-controlled close on the source grasp width; we did not tune
  per-object grasp force. This is a diagnostic of the *generation label*, not a grasp
  policy.
- ε models pose error generically; it is not RoboSplat's own augmentation noise.

## Reproduce

```bash
# rollouts (GPU, robosplat env) — ~6-8 min:
bash scripts/run_consistency.sh src/robosplat_consistency/run_batch.py
# figures + tables (CPU):
bash scripts/run_consistency.sh src/robosplat_consistency/analyze.py
```
