# Start-Over Handoff Summary

Generated on 2026-06-19 after restoring the workspace to the original cloned-repo state.

This file is the one retained summary of the experiment work. The experiment scripts, result CSVs, model ZIPs, logs, videos, pycache files, and generated notebooks were removed from the working tree by resetting the top-level repo to the pre-artifact commit and restoring the nested repos to the commits recorded there.

## Final Workspace State

- Top-level repo restored to `6b132cb` (`PhysicsInformed`), the commit before the bulk artifact save.
- `Creeping` restored to `ddb5332` (`Removed the cap`).
- `HighwayEnv` restored to `01f5ffff` (`Update README.md`).
- `rl-agents` restored to `84df15e` (`Update build badge`).
- The top-level branch is intentionally behind `origin/master` by one commit because `origin/master` still points at the artifact-heavy commit `cf5ae68` (`chore: save notebook artifacts and submodule updates`). Do not pull or merge that commit if the goal is to keep the clean restart state.
- Empty leftover `notebooks/results` and `notebooks/__pycache__` directories were removed.
- This `START_OVER_SUMMARY.md` file is the only new file intentionally left behind.

## Repo Map

### `Creeping`

This is the SUMO/CARLA creeping-negotiation repo.

- `Creeping_Sumo/Scenario1.*`, `Scenario2.*`, `Scenario3.*`: SUMO networks, route files, and configs for intersection scenarios.
- `Creeping_Sumo/simulation/environment.py`: main SUMO/TraCI environment. It models a T-intersection creeping task, TTC features, creep-zone logic, target speed near the intersection, and reward components such as safety, approach, creep speed, yield response, progress, and gap.
- `Creeping_Sumo/driver.py`: PPO training driver with curriculum-style route switching and logging.
- `Creeping_Sumo/test_creeping.py`: behavioral inspection script for trained creeping policies. It reports creep-zone action counts, mean creep speed, TTC, and qualitative pass/fail signals.
- `autoencoder/`: VAE encoder/decoder utilities and reconstructed image outputs in the previous state.

### `HighwayEnv`

Upstream `highway-env`, used as the simulator for the later intersection experiments. The restored clone is back at the recorded upstream-ish commit. The previous experiment work did not need to keep local modifications here.

### `rl-agents`

Upstream `rl-agents`, used as a reference RL agents package. The restored clone is back at the recorded upstream-ish commit. The previous artifact commit had included pycache and desktop conflict files; those are gone from the current state.

### `notebooks`

After cleanup, only the original tracked notebook remains:

- `notebooks/intersection_rl_agents_training.ipynb`

The removed work had added scripts, reports, trained models, logs, videos, and evaluation CSVs under `notebooks/`.

## What Was Built During The Removed Work

### 1. First HighwayEnv Creeping Formulation

A helper module called `highway_ppo_creeping_utils.py` was created for a basic `intersection-v0` PPO setup:

- Kinematics observation normalization.
- TTC estimation from relative vehicle motion.
- Hand-coded and learned policies.
- Reward shaping for creeping near the intersection.
- Evaluation summaries with success, collision, timeout, TTC, creep-zone speed, and action rates.
- Behavior cloning from scripted creeping teachers, followed by PPO fine-tuning.

This version was useful for proving that reward shaping can create creeping-like behavior, but it was not route-specific enough for the straight versus left-turn benchmark.

### 2. Route-Aware Attention Formulation

The main later work moved to a route-aware module called `highway_route_attention_creeping_utils.py`.

Core ideas:

- `RouteIntersectionConfig` encoded maneuver-specific destination:
  - `straight -> o2`
  - `left -> o1`
- The target action space was discrete longitudinal speed commands:
  - `0.0`, `2.0`, `4.5`, `7.0`, `9.0` m/s
- The target benchmark floor was:
  - `initial_vehicle_count = 12`
  - `spawn_probability = 0.5`
  - `duration = 22`
  - `negotiating_traffic = False`
- The attention observation wrapper created near-vehicle tokens plus route-level auxiliary cues:
  - ego speed
  - distance to center
  - conflict TTC
  - approach/creep-zone flags
  - reward-only target speed
- The `NearVehicleAttentionExtractor` used ego and nearby-vehicle embeddings with distance-biased attention before the PPO policy/value heads.
- Reward shaping encouraged:
  - approach progress
  - controlled creep-zone speed
  - low high-speed-zone rate
  - avoiding low TTC
  - eventual arrival
- Evaluation tracked both safety and efficiency:
  - success rate
  - collision rate
  - timeout rate
  - mean survival time
  - min TTC and conflict TTC
  - creep-zone mean speed
  - high-speed-zone rate
  - creep-speed rate
  - ego delay proxy
  - traffic delay proxy
  - system delay proxy
  - stopped time
  - action rates

### 3. Behavior Cloning Plus PPO

A scripted creeping teacher was used to behavior-clone the PPO actor before fine-tuning. The best route-aware policies in the removed results were the calibrated attention behavior-cloned policies:

- `straight_ppo_attention_bc_creeping_calibrated_v2.zip`
- `left_ppo_attention_bc_creeping_calibrated_v2.zip`

Those model artifacts are now deleted, but the result numbers are summarized below.

### 4. Safety-Ellipse Shield

A hard safety wrapper was added around the ego action:

- It predicted future ego and neighbor positions.
- It inflated vehicle footprints as ellipses.
- It rejected actions whose predicted clearance went below a margin.
- It tried preferred fallback actions and optional emergency braking.
- Later probes used a reduced margin and shorter horizon because the first shield was too conservative.

Important lesson: the shield reduced some collisions, but it often converted crashes into deadlocks/timeouts and removed the desired creeping signature unless the policy was trained inside the shielded dynamics.

### 5. Crash-Tolerant Benchmark Runner

The benchmark was moved into per-episode subprocess workers:

- `run_benchmark12_05_subprocess.py`
- `route_episode_worker.py`

Purpose:

- Isolate simulator crashes.
- Record crash rows instead of losing the whole run.
- Evaluate policies at 12/0.5 density across straight and left maneuvers.

### 6. Forced-Zero Reset Audit

`audit_forced_zero_resets.py` checked whether dense resets were impossible for an ego-only controller. It forced ego speed to zero immediately after reset and stepped the slowest action.

Result: in the 12/0.5 audit slice, there were no forced-zero collisions over 100 resets for either straight or left. That means the remaining failures should be treated as policy/shield failures unless a larger audit finds impossible resets.

### 7. Native HighwayEnv PPO Baseline

`run_native_intersection_ppo_baseline.py` trained/evaluated PPO directly on native `intersection-v0` at 12/0.5.

It included two local patches during experimentation:

- A safer `RegulatedRoad.is_conflict_possible` approximation to avoid a Windows crash in dense left-turn conflict prediction.
- A safer closest-lane lookup that avoided failures from some lane geometry cases.

This native baseline was useful as a sanity check, but the available run was short and mostly learned to drive fast into collisions.

## Key Results From The Removed Artifacts

### Very Dense 6/0.35 Final Reward Comparison, 120 Episodes

This was the strongest result for the route-aware calibrated attention policy, but it was below the final 12/0.5 target density.

| Maneuver | Policy | Episodes | Success | Collision | Mean Creep-Zone Speed | High-Speed Zone | Creep-Speed Rate |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: |
| straight | fast/non-suited | 120 | 54.2% | 45.8% | 8.91 | 99.2% | 0.0% |
| straight | calibrated attention creeping | 120 | 90.8% | 9.2% | 4.11 | 1.6% | 88.9% |
| left | fast/non-suited | 120 | 51.7% | 48.3% | 8.90 | 99.2% | 0.0% |
| left | calibrated attention creeping | 120 | 94.2% | 5.8% | 5.27 | 40.1% | 54.0% |

Main takeaway: the learned creeping policy clearly beat fast driving at 6 vehicles / 0.35 spawn, especially on success and collision rate. But left-turn creeping still had too much high-speed-zone behavior.

### Clean Dense 6/0.35 Efficiency Comparison, 100 Episodes

| Maneuver | Policy | Success | Collision | System Delay Proxy | Creep-Speed Rate | Safe-Creeping Efficiency Index |
| --- | --- | ---: | ---: | ---: | ---: | ---: |
| straight | fast non-creeping | 76% | 24% | 5.28 | 0.0% | 0.066 |
| straight | learned creeping | 82% | 18% | 27.72 | 98.0% | 0.342 |
| left | fast non-creeping | 58% | 42% | 5.43 | 0.0% | 0.039 |
| left | learned creeping | 68% | 32% | 25.33 | 97.9% | 0.245 |

Main takeaway: learned creeping improved safety and negotiation discipline, but it increased total delay. Delay alone is misleading because the fast baseline often "wins" by failing early.

### Target 12/0.5 Benchmark, 20 Episodes

This is the benchmark that matters most for restart.

| Maneuver | Policy | Success | Collision | Timeout | Survival s | System Delay Proxy | Creep-Speed Rate | Shield Intervention |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| straight | learned creeping | 60% | 40% | 0% | 13.44 | 33.69 | 99.0% | 0.0% |
| straight | fast non-creeping | 35% | 65% | 0% | 6.36 | 5.49 | 0.0% | 0.0% |
| straight | learned safety shield | 40% | 25% | 35% | 17.02 | 71.17 | 60.9% | 35.1% |
| straight | safety progress assist | 5% | 25% | 70% | 20.07 | 92.57 | 48.7% | 67.2% |
| left | learned creeping | 50% | 50% | 0% | 12.13 | 39.03 | 90.5% | 0.0% |
| left | fast non-creeping | 40% | 60% | 0% | 6.34 | 4.39 | 0.0% | 0.0% |
| left | learned safety shield | 60% | 20% | 20% | 16.80 | 36.48 | 69.6% | 22.8% |
| left | safety progress assist | 5% | 10% | 85% | 21.47 | 115.28 | 38.8% | 69.0% |

Main takeaways:

- At the target 12/0.5 density, no current policy was close to solved.
- Unshielded learned creeping preserved the creeping signature but still collided too often.
- The safety shield reduced collisions in some cases but introduced timeouts and heavy delay.
- The progress-assist variant was worse because it mostly froze or timed out.
- The fast baseline had low delay only because it crashed quickly.

### Negotiation-Efficiency Diagnostic At 12/0.5

The diagnostic score rewarded successful crossing, penalized collisions/timeouts and system delay, penalized high-speed movement in the conflict zone, and rewarded actual creep-speed behavior.

| Maneuver | Policy | Safe-Crossing Efficiency | Conflict-Discipline Efficiency | Negotiation Efficiency Index |
| --- | --- | ---: | ---: | ---: |
| straight | learned creeping | 0.2151 | 0.2134 | 0.2112 |
| straight | learned safety shield | 0.0476 | 0.0443 | 0.0270 |
| straight | safety progress assist | 0.0006 | 0.0006 | 0.0003 |
| left | learned creeping | 0.1404 | 0.1340 | 0.1213 |
| left | learned safety shield | 0.1493 | 0.1386 | 0.0965 |
| left | safety progress assist | 0.0005 | 0.0005 | 0.0002 |

Main takeaway: for straight, unshielded learned creeping was best by this diagnostic. For left, the shield improved raw safety enough to slightly improve safe-crossing efficiency, but it reduced actual creeping behavior.

### Forced-Zero Reset Audit, 12/0.5, 100 Resets Each

| Maneuver | Tested Resets | Forced-Zero Collisions | Collision Rate | Stationary Steps |
| --- | ---: | ---: | ---: | ---: |
| straight | 100 | 0 | 0.0% | 3 |
| left | 100 | 0 | 0.0% | 3 |

Main takeaway: the 12/0.5 failures cannot be dismissed as obviously unavoidable initial collisions for this seed slice.

### Shielded Interaction Fine-Tune v3, 12/0.5, 5k PPO Steps

| Maneuver | Success | Collision | Timeout | System Delay Proxy | Creep-Speed Rate | Shield Intervention |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| straight | 65% | 15% | 20% | 49.39 | 10.4% | 24.2% |
| left | 20% | 15% | 65% | 96.04 | 46.9% | 49.1% |

Main takeaway: short shield-aware fine-tuning helped straight collision rate but damaged the creeping behavior; left mostly became timeout/deadlock. A deeper redesign is needed.

### Native HighwayEnv PPO Baseline, 12/0.5, 100 Episodes

| Maneuver | Success | Collision | Timeout | Mean Survival s | High-Speed Zone | Faster Action Rate |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| straight | 44% | 56% | 0% | 6.85 | 99% | 100% |
| left | 44% | 56% | 0% | 6.58 | 98% | 100% |

Main takeaway: the native PPO baseline mostly learned or retained a fast-driving behavior under the short available run. It is not a solved baseline and should be rerun cleanly with enough training if used.

### Rendered 12/0.5 Video Metrics, 10 Episodes Each

| Policy | Maneuver | Success | Collision | Ego Delay | Traffic Delay | System Delay |
| --- | --- | ---: | ---: | ---: | ---: | ---: |
| before learned safety | straight | 50% | 50% | 5.27 | 24.28 | 29.55 |
| before learned safety | left | 50% | 50% | 5.64 | 29.72 | 35.36 |
| after learned safety | straight | 50% | 30% | 8.74 | 46.05 | 54.79 |
| after learned safety | left | 40% | 20% | 13.08 | 74.57 | 87.65 |

Main takeaway: visual/video runs matched the CSV story. The shield made some episodes safer but often more hesitant and delayed.

## Main Lessons

1. The real target is 12 vehicles and 0.5 spawn probability. Results at 6/0.35 are encouraging but not enough.

2. Creeping behavior was achieved in the lower-density route-aware setting. The calibrated attention behavior-cloned policy had strong success/collision performance at 6/0.35 and preserved a clear low-speed creep-zone signature.

3. The 12/0.5 setting is not solved. Current best unshielded policies collide too often. Current shielded policies reduce some collisions but can freeze, timeout, or erase creeping behavior.

4. Do not optimize only for raw delay. Fast policies look efficient because they crash early. Use success, collision, timeout, conflict-zone speed discipline, stopped time, and system delay together.

5. Treat safety as part of training, not just a post-hoc filter. A hard shield changes the dynamics seen by the policy. The best next version should train inside the shielded/action-projected environment from the beginning or use a softer differentiable/penalty-style safety mechanism.

6. Keep reset audits in the loop. The forced-zero audit showed no unavoidable stationary-ego collisions in the tested 12/0.5 seeds, so failures should drive policy and shield improvements.

7. Keep generated artifacts out of git. The removed commit tracked hundreds of CSVs, MP4s, ZIP models, logs, and pycache files. On restart, add a `.gitignore` before any training.

## Suggested Restart Plan

1. Pick one primary simulator path before coding:
   - SUMO/Creeping if the project needs TraCI and the custom T-intersection setup.
   - HighwayEnv route-aware intersection if the project needs fast PPO iteration and simple Python-only experiments.

2. Rebuild the experiment harness minimally:
   - one environment wrapper
   - one metrics module
   - one training script
   - one evaluation script
   - one results-summary script

3. Start with deterministic reference policies:
   - fast non-creeping
   - hold/idle
   - scripted reward creeping
   - optionally native HighwayEnv PPO

4. Lock the benchmark:
   - straight and left
   - `initial_vehicle_count=12`
   - `spawn_probability=0.5`
   - `duration=22`
   - no negotiating traffic unless explicitly testing that ablation
   - separate training and evaluation seeds

5. Add tests before long runs:
   - reset does not crash
   - one episode terminates
   - metrics are finite
   - forced-zero audit runs
   - straight and left route destinations are correct

6. Train in stages:
   - behavior clone a scripted creeping teacher
   - PPO fine-tune at moderate density
   - PPO fine-tune at 12/0.5
   - only then introduce safety constraints

7. If using a safety shield:
   - train with the shield active from the start of fine-tuning
   - log intervention rate, emergency brake rate, violation rate, and timeout rate
   - reject configurations that improve collision rate by mostly creating timeouts

8. Save only reproducible essentials:
   - config JSON
   - compact summary CSV
   - final markdown report
   - optionally a few curated videos outside git
   - never commit models, large videos, logs, or pycache

## Commands And Files That Existed Before Cleanup

The removed files included:

- `notebooks/highway_ppo_creeping_utils.py`
- `notebooks/highway_route_attention_creeping_utils.py`
- `notebooks/run_route_attention_creeping_experiments.py`
- `notebooks/run_benchmark12_05_subprocess.py`
- `notebooks/route_episode_worker.py`
- `notebooks/run_shielded_interaction_finetune_12_05.py`
- `notebooks/run_native_intersection_ppo_baseline.py`
- `notebooks/audit_forced_zero_resets.py`
- `notebooks/build_12_05_efficiency_report.py`
- `notebooks/results/...`
- generated `.ipynb` reports
- trained `.zip` model artifacts
- `.mp4` videos
- `.log` files
- Python `__pycache__` files

Use this summary to recreate only the useful pieces, deliberately and cleanly.
