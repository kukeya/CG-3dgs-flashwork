# Same Visual, Different Physics — diagnostic results

Implements [docs/svdp-plan.md](svdp-plan.md) (the "Same Visual, Different Physics"
experiment). This is the **fair** version of the earlier ε-pose-perturbation test:
it never moves the object, never feeds the robot a wrong position, never changes
anything the robot or the RoboSplat visual pipeline observes. It changes **only**
hidden physical properties of the object inside PhysX.

> Same rendered 3DGS observation · same object pose · same camera · same robot
> trajectory · same RoboSplat visual/kinematic success label (= 1.0).
> **Only friction / mass / center-of-mass / hidden collision geometry differ.**

## Headline result

RoboSplat stamps **every** generated demo with success label 1.0. Replaying the
identical action against the same-looking object, physical pick success ranges from
**0.43 to 1.0** depending purely on hidden physics the rendering cannot encode.

`tables/table_same_visual_diff_physics.csv`,
`outputs/same_visual_different_physics/` (440 rollouts, 10 representative demos).

| hidden property varied | RoboSplat label | physical success (100 N idealized grasp) | physical success (5 N realistic grasp) | main failure |
| --- | --- | --- | --- | --- |
| none (nominal) | 1.0 | 1.00 | 1.00 | — |
| **friction** μ∈[0.05,1.5] | 1.0 | 0.93 | **0.43** | unstable_lift / slip |
| **mass** ∈[20,400] g | 1.0 | 0.92 | **0.70** | weak_grasp |
| **center-of-mass** offset ∈[0,2] cm | 1.0 | **0.58** | **0.64** | unstable_lift (tilt) |
| **hidden collision width** | 1.0 | 0.96 | **0.60** | unstable_lift |

Figure `figures/visual_label_vs_physical_success.png` is the headline; per-axis
curves (`friction_vs_success.png`, `mass_vs_success.png`, `com_offset_vs_success.png`,
`collision_width_vs_success.png`) and `failure_taxonomy.png` are alongside.

## What the two grasp regimes show (important nuance)

We replay each sweep at two gripper-force regimes, because the regime itself is part
of the finding:

- **100 N (RoboSplat's original `force_limit`, robot_util.py:68).** An idealized,
  very strong clamp. It is so strong that **friction, mass and collision width
  barely matter** (0.92–0.96) — the grasp brute-forces a hold. Only a shifted
  center-of-mass breaks it (0.58), because no clamp force cancels the lift torque
  from an off-center mass → the object tilts 90° and slips.
- **5 N (a realistic compliant pinch).** Now every hidden property is load-bearing
  and the same visual/label produces clearly divergent outcomes. Friction follows a
  textbook Coulomb-slip threshold (success ≈0 at μ=0.05, →1.0 by μ≈1.0); heavier
  objects slip; off-center mass and wrong contact width fail.

So the experiment delivers two messages at once: (1) visual realism cannot encode
the physics that decides the outcome, and (2) **even the physics model's own
idealizations** (an over-strong grasp) can hide the gap — exactly why a generation-
time or visual label cannot certify action-effect correctness.

## Failure modes are structural, not random

Under the realistic grasp, failures are dominated by `weak_grasp`, `slip_out`, and
`unstable_lift` (tilt/drop) — contact-dynamics mechanisms, in the physically expected
direction (low μ → slip; heavy → weak/drop; off-center → tilt). Not controller bugs:
the nominal config is 100% at both regimes, and the source-demo replay sanity gate
(15 cm lift) passed before any sweep.

## Go / No-Go (per plan §11–12)

**GO**, with the realistic-grasp regime as the headline:

1. nominal replay succeeds (sanity ✅);
2. hidden physics changes while the rendered observation is byte-identical ✅;
3. RoboSplat visual/kinematic label is 1.0 for all 440 rollouts ✅;
4. physical success varies strongly with friction / mass / COM / collision ✅;
5. many high-visual-plausibility, low-physical-success cases exist (e.g. μ=0.1 at
   5 N: label 1.0, physical success 0.0) ✅;
6. failures are physical mechanisms (slip / tilt / weak grasp), not bugs ✅.

The one caveat the plan flags (§12): under the **idealized 100 N** grasp, friction
and mass mostly *don't* matter. We do not hide this — we report it as the second
half of the finding (the idealization masks the gap), and use the realistic grasp +
COM-offset (which fails in *both* regimes) as the robust evidence.

## Recommended paper statement

> RoboSplat generates visually realistic, spatially consistent demonstrations whose
> success is asserted in image/kinematic space. We construct a Same-Visual-Different-
> Physics diagnostic: holding the rendered observation, object pose, camera and robot
> trajectory fixed, we vary only the object's hidden friction, mass, center-of-mass,
> or contact geometry in physics. Although all cases share an identical visual and
> kinematic success label, physical execution success ranges from 0.43 to 1.0, with
> structured slip/tilt/weak-grasp failures. Visual plausibility therefore cannot
> certify action-effect consistency, motivating a test-time, physics-aware
> action-effect verifier.

## Caveats (load-bearing, per CLAUDE.md)

- Object is a **box/cylinder proxy** (RoboSplat ships no carrot collision mesh).
  Absolute thresholds depend on the proxy; the qualitative decoupling (flat 1.0
  label vs physics-dependent success) does not. No real-robot claims.
- The grasp-force regimes (100 N vs 5 N) bracket idealized↔realistic; they are a
  modeling choice, reported explicitly, not tuned per object.
- Sweep run on 10 representative demos (grid center/edges/corners). Trends are
  consistent across them; scaling to all 100 is a cheap follow-up if needed.

## Reproduce

```bash
bash scripts/run_consistency.sh src/robosplat_consistency/run_svdp.py       # ~8 min, GPU
bash scripts/run_consistency.sh src/robosplat_consistency/analyze_svdp.py   # figures+table
```

## Success / failure videos

Physics-replay rollouts rendered from the sapien scene (Panda + the physical proxy),
with a green/red lift-progress bar, in
`outputs/same_visual_different_physics/videos/`:

- `demo52_mu1.0_200g_SUCCESS.mp4` — box lifted 14.8 cm.
- `demo52_mu0.1_200g_FAIL.mp4` — low friction, slips (0.4 cm).
- `demo52_mu0.3_400g_FAIL.mp4` — too heavy, no lift.
- `demo52_com15mm_FAIL.mp4` — off-center mass, tilts and drops.
- `demo52_mu0.1_200g_100N_SUCCESS.mp4` — same μ=0.1 but the idealized 100 N grasp
  holds it (shows the idealization masking the gap).

Side-by-side comparisons (RoboSplat's own photoreal render, which always *looks*
successful with label 1.0, next to the physics replay of the **same** demo) in
`outputs/same_visual_different_physics/videos/compare/`:

- `compare_success.mp4` — render vs replay that lifts.
- `compare_lowfriction_fail.mp4` — render looks identical, replay slips.
- `compare_com_fail.mp4` — render looks identical, replay tilts/drops.

Rebuild videos: `bash scripts/run_consistency.sh src/robosplat_consistency/render_cases.py`
then `bash scripts/compose_compare.sh`.
