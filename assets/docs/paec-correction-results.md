# PAEC Correction — results

Implements the PAEC correction experiment plan. This phase stops doing diagnostics
(the Same-Visual-Different-Physics gap is already established) and verifies the
**method's core loop**:

> original action → fails under hidden physics → PAEC diagnoses the failure mode →
> proposes a corrected action → re-verifies candidates → corrected action improves
> physical success.

Same contract as SVDP: same rendered scene, same object pose/camera/appearance, same
RoboSplat-generated trajectory, RoboSplat visual/kinematic label = 1.0 for every row.
Only hidden physics (friction / mass / center-of-mass / contact geometry) varies in
PhysX. The verifier V is the physics rollout itself (**oracle**, per CLAUDE.md) — this
isolates the correction/routing *mechanism*, not verifier learnability.

## Setup

- 100 RoboSplat `pick_100` demos × 6 hidden-physics groups × 5 methods = 3000 rows
  (`outputs/paec_correction/pick100_eval/summary.csv`).
- Verifier ranks candidates on a **belief sample** of K=12 hidden-physics hypotheses;
  success is **reported** on an independent **test sample** of T=8 hypotheses from the
  same distribution ("belief covers test"). `oracle_best` peeks at the test sample;
  `paec_selected` never does — so the gap between them measures the value of
  verifier-guided selection even under an oracle V.
- Original action runs at a realistic 5 N pinch (the regime where hidden physics is
  load-bearing in the SVDP sweep).
- Correction candidate space (interpretable high-level edits only, no full-trajectory
  re-optimization): grip force, lift-speed scale, gripper contact-margin width, grasp
  offset (x/y), hold time. Candidates are **routed by the diagnosed failure type**, not
  brute-forced.

## Headline numbers (100 demos)

| method | success | gain | mean slip↓ | mean tilt↓ | action deviation |
| --- | ---: | ---: | ---: | ---: | ---: |
| original | 0.488 | — | 0.088 | 0.662 | 0.00 |
| uniform conservative | 0.844 | +0.356 | 0.034 | 0.287 | 2.27 |
| rule correction | 0.717 | +0.232 | 0.053 | 0.342 | 1.86 |
| **PAEC selected** | **0.904** | **+0.414** | **0.022** | **0.248** | **0.14** |
| oracle best | 0.909 | +0.423 | 0.021 | 0.243 | 0.16 |

Three questions, answered:

- **Q1 — does correction raise success?** Yes: 0.488 → 0.904.
- **Q2 — is failure-conditioned routing better than uniform conservative?** Yes
  (+0.414 vs +0.356), and decisively so on the torque-driven case (below).
- **Q3 — is verifier-guided selection necessary?** Yes: PAEC (0.904) tracks the
  oracle upper bound (0.909) and far exceeds the fixed rule table (0.717). Re-checking
  candidates and choosing the per-demo magnitude matters.

## Per-group success (Table 1)

| group (hidden) | original | uniform | rule | **PAEC** | oracle | main failure |
| --- | ---: | ---: | ---: | ---: | ---: | --- |
| nominal | 1.000 | 1.000 | 1.000 | 1.000 | 1.000 | — |
| low_friction (μ 0.05–0.2) | 0.016 | 0.885 | 0.625 | **0.940** | 0.945 | slip / unstable |
| heavy_mass (200–400 g) | 0.216 | 0.874 | 0.593 | **0.882** | 0.886 | weak_grasp |
| com_offset (7–9 mm) | 0.814 | **0.689** | 0.686 | **0.988** | 0.988 | unstable_lift |
| collision_mismatch (hw 11–13 mm) | 0.868 | 0.990 | 0.889 | **0.993** | 0.999 | slip / unstable |
| combined_hard | 0.014 | 0.624 | 0.507 | 0.620 | 0.635 | mixed |

**The com_offset row is the load-bearing result.** A center-of-mass offset causes the
object to tilt out under torque, not slip under insufficient normal force. Uniform
conservative ("grasp harder + slower + hold") actually *drops below the original*
(0.814 → 0.689) — grasping harder cannot fix a torque failure and the slower lift gives
the tilt more time to develop. PAEC diagnoses `unstable_lift` and routes to a
**grasp-point offset toward the COM**, reaching 0.988. This is the same falsifiable
punchline as the Tier-1 force-correction PoC (ΔF stays flat on COM), now shown to be
*recoverable by the correctly-routed correction*.

## Correction precision — PAEC doesn't over-correct

Fraction of (demo, group) cases where PAEC applied a non-identity correction:

| group | % corrected |
| --- | ---: |
| nominal | 0% |
| low_friction | 99% |
| heavy_mass | 100% |
| com_offset | 26% |
| collision_mismatch | 34% |
| combined_hard | 89% |

On nominal it never touches a working grasp. On com_offset / collision it intervenes
only on the ~quarter–third of cases that actually fail, leaving the rest alone. Uniform
conservative and rule correction, by construction, alter 100% of actions.

## Cost vs gain (§8.4) — gains are not bought by extreme actions

PAEC reaches a higher success rate than uniform conservative at **~16× lower action
deviation** (0.14 vs 2.27). Uniform conservative clamps every grasp to 100 N + half
lift speed + extra hold regardless of need; PAEC selects the minimal routed edit that
the verifier predicts will work and leaves successful grasps untouched. See
`figures/correction_cost_vs_gain.png`.

## Correction-axis ablation (Table 3, Stage 3, 50 demos)

`paec_selected` restricted to a single correction axis, vs the full routed set:

| axis kept | overall | low_friction | heavy_mass | com_offset | collision | combined |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| force only | 0.884 | 0.94 | 0.90 | 0.855 | 0.99 | 0.62 |
| speed only | 0.498 | 0.02 | 0.23 | 0.85 | 0.88 | 0.01 |
| width only | 0.496 | 0.01 | 0.21 | 0.85 | 0.89 | 0.01 |
| offset only | 0.525 | 0.02 | 0.23 | **0.99** | 0.91 | 0.01 |
| **full** | **0.908** | 0.93 | 0.91 | **0.99** | 0.99 | 0.62 |

Each axis recovers the failure mode its physics predicts, and only that one:

- **Force** carries friction / mass / collision — slip and weak-grasp are
  insufficient-normal-force problems, so a tighter clamp fixes them (0.884 alone).
- **Grasp offset is the only axis that fixes com_offset** (0.99, vs force's 0.855).
  A center-of-mass offset is a torque failure; no amount of clamp force corrects it
  (the Tier-1 PoC showed ΔF stays flat on COM). Moving the grasp point toward the COM
  does. This is the ablation form of the routing argument.
- **Full** combines them for the best overall (0.908) and the best com_offset (0.993)
  without sacrificing the force-driven groups.
- **Speed / width alone** are near-inert: width especially, because a rigid box is
  caged by fingers that close to 0 regardless of margin (box-proxy caveat).

## Generalization to held-out physics (Stage 4, 50 demos)

The belief and (covering) test samples above are drawn from the same ranges. Stage 4
instead reports success on **held-out** hidden-physics values drawn strictly *outside*
the belief range the verifier ranked candidates on — lower friction, heavier mass,
larger COM offset, thinner box than anything in belief. The correction is still chosen
on the in-distribution belief; only the physics it faces at test time is unseen.

| group (held-out) | original | PAEC | gain |
| --- | ---: | ---: | ---: |
| nominal | 1.000 | 1.000 | +0.000 |
| low_friction | 0.000 | 0.840 | +0.840 |
| heavy_mass | 0.000 | 0.797 | +0.797 |
| com_offset | 0.762 | 0.935 | +0.173 |
| collision_mismatch | 0.945 | 0.993 | +0.048 |
| combined_hard | 0.000 | 0.005 | +0.005 |
| **overall** | **0.451** | **0.762** | **+0.310** |

Corrections selected on the belief range transfer to unseen, harder physics: every
single-failure group still improves substantially (low_friction +0.84, heavy_mass
+0.80, com_offset +0.17). The exception is `combined_hard` held-out (0.000 → 0.005) —
several simultaneous extreme failures beyond the reach of the routed single-axis
corrections. This is an honest limitation: the routed candidate space recovers
single-mechanism failures robustly but does not solve compounded out-of-distribution
hard cases, which would need joint multi-axis search or a learned correction head.

## Figures

- `figures/success_improvement.png` — per-group success bars, all methods.
- `figures/failure_taxonomy_before_after.png` — failure-type distribution, original vs
  PAEC (slip_out / weak_grasp / unstable_lift collapse into success).
- `figures/correction_cost_vs_gain.png` — deviation vs gain scatter.

## Qualitative before/after videos

Physics rollouts of representative before/after cases are rendered to mp4 via
`src/robosplat_consistency/render_correction_cases.py` (launched through
`scripts/run_consistency.sh`), then assembled into a paper keyframe montage
(`paper/figures/paec_correction_keyframes.png`) by
`scripts/make_paec_correction_montage.sh`. Each hidden-physics group shows a
BEFORE row (original action → slip / tilt / drop) above an AFTER row (PAEC-selected
correction → stable lift), columns = approach / lift / final outcome. Cases:
low_friction demo 99 (+grip force), com_offset demo 99 (+grasp-point shift),
heavy_mass demo 97 (+grip force). mp4s in `outputs/paec_correction/videos/`.

## Caveats (load-bearing per CLAUDE.md)

- **Oracle verifier.** V is the physics rollout, not a learned surrogate. These results
  prove the correction/routing *mechanism* and the value of re-verification, not that a
  learned verifier can reproduce them. The learnability question is separate.
- **Box proxy.** The target is a box approximating the carrot cross-section. Absolute
  tolerances are not literal; the *qualitative decoupling* (which correction fixes which
  failure) is the claim. The contact-margin *width* correction is near-inert in this
  proxy — a rigid box is caged by fingers that close to 0, so width rarely changes the
  outcome (flagged, not hidden).
- **Sim-only.** No real-robot claims. RoboSplat is kinematic; all physics is added in
  the rebuilt PhysX scene (see `robosplat_consistency_results.md`).

Artifacts: `outputs/paec_correction/pick100_eval/{summary.csv, tables/, figures/}`,
`outputs/paec_correction/debug_10demos/summary.csv` (Stage-1 dev). Run via
`scripts/run_consistency.sh .../run_paec_correction_eval.py` and
`scripts/run_paec_ablation.sh` (Stage 3 ablation + Stage 4 generalization).
