---
layout: page
title: Whole-Body VLA for Long-Horizon Home Tasks
description: Fridge, right drink, close, hand over, on a 20-DoF mobile humanoid.
img: assets/img/work/home_unseen_objects_thumb.jpg
importance: 2
category: work
permalink: /projects/whole-body-home/
alt_url: /zh/projects/whole-body-home/
related_publications: false
---

**TL;DR.** I took a household task on the Tien Yi humanoid from first demonstrations to a stable demo behind an LLM agent: open the fridge, bend, pick the requested drink from look-alikes, close the door, hand it over. One π0.5 policy controls the dual arms, waist and legs (20 DoF). Most of the gains came from treating data and debugging as engineering problems, not from a bigger model.

<div class="row mt-3">
  <div class="col-sm mt-3 mt-md-0">
    <video src="{{ '/assets/video/work/home_full_task.mp4' | relative_url }}" poster="{{ '/assets/img/work/home_full_task.jpg' | relative_url }}" class="img-fluid rounded z-depth-1" autoplay muted loop playsinline controls preload="metadata"></video>
  </div>
  <div class="col-sm mt-3 mt-md-0">
    <video src="{{ '/assets/video/work/home_unseen_objects.mp4' | relative_url }}" poster="{{ '/assets/img/work/home_unseen_objects.jpg' | relative_url }}" class="img-fluid rounded z-depth-1" autoplay muted loop playsinline controls preload="metadata"></video>
  </div>
  <div class="col-sm mt-3 mt-md-0">
    <video src="{{ '/assets/video/work/home_unseen_category.mp4' | relative_url }}" poster="{{ '/assets/img/work/home_unseen_category.jpg' | relative_url }}" class="img-fluid rounded z-depth-1" autoplay muted loop playsinline controls preload="metadata"></video>
  </div>
</div>
<div class="caption">
  Left: full task on the mobile base (2× speed). Middle: drinks never seen in training. Right: an unseen brand of bottled water (2× speed).
</div>

### What made it hard

- **Whole body, not just arms.** The robot bends and turns at the waist mid-task: 7+1 dims per arm and gripper, plus 2 waist and 2 leg dims, in one 20-D action space. An error in the waist propagates to the hand: at a 0.3 rad waist error the arm joints can be exactly right and the grasp still misses.
- **Look-alike targets.** Four drinks sit side by side, and the wrong one is only a few centimeters away. Color is the main cue that separates them.
- **Self-occlusion.** The arm repeatedly blocks the head camera exactly when the policy needs to see.
- **Long horizon.** An episode runs about 1,000 frames at 27 fps. A small mistake early, such as a door left ajar, only shows up at the end.

### Training recipe

| Choice                      | Setting                                                         | Why                                                                                  |
| --------------------------- | --------------------------------------------------------------- | ------------------------------------------------------------------------------------ |
| Base                        | π0.5 (~3B), full fine-tune including the vision tower           | The visual distinctions are task-specific                                            |
| Action space                | Relative joint actions, absolute for the two grippers           | Removes per-episode pose offsets; grippers are effectively binary                    |
| Normalization               | Quantile (q01 / q99) for state and action                       | Robust to rare large waist and leg excursions                                        |
| Chunk                       | 50 steps, all executed                                          | Pairs with the [real-time execution stack](/projects/realtime-execution/)            |
| Scale                       | 8× A800 DDP, global batch 256, bf16, torch.compile              | Matches common OpenPI fine-tuning batch                                              |
| Schedule                    | lr 2.5e-5, 1k warm-up, cosine to 2.5e-6 over 30k steps (~2.8 epochs) | Held-out 5% of episodes shows where overfitting starts                          |
| Augmentation                | GPU-side affine, brightness, contrast, mild saturation (±15%), **no hue** | Default hue jitter turns orange juice into cola                             |

The augmentation row was a real lesson. A stock recipe of hue ±0.05 and saturation 0.5–1.5 is harmless on most tasks. Here it erased the one feature that defines the class.

### Data as an engineered system

I wrote the collection SOP and grew the dataset to **2,934 episodes (2.95M frames)** across 20 task instructions, varying drink type, shelf slot and base position. Every episode passes an automatic QA pass, then human review:

| Check                     | Rule (examples)                                                              |
| ------------------------- | ---------------------------------------------------------------------------- |
| Gripper binarity          | Fail if the gripper dwells in a half-open state for more than 2 s             |
| Gripper chatter           | Fail on 3 or more open/close switches within 5 s                              |
| Joint jumps               | Flag single-frame jumps above 0.1 rad/frame                                   |
| Pattern outliers          | Flag open/close counts, timing and trajectory shape far from the mode (z-score) |
| Slot labelling            | Cluster grasp points per collection day to catch misplaced target bottles     |

On the first batch the pipeline flagged **54 of 229 episodes** (27 fail, 27 warn). Reviewers confirm or overturn each flag in a web UI, and the cleaned dataset is re-exported with a manifest. Recordings mixed at 26 and 27 fps are re-timed before merging. Qwen3-VL labels sub-task stages (open, reach, grasp, retract, close), so success can be reported per phase rather than as one number.

### Debug by root cause, not by retraining

| Symptom                       | Root cause                                                                                                              | Fix                                                              |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- |
| Slots 1 and 4 underperform    | At the waist-turn decision frame, the arm sweeps across the head camera and hides slots 1–3. Slot 4 episodes are 18% longer than slot 2, with the largest trajectory variance | Adjust demonstration motion; retune the schedule |
| Bottle slips during transport | Position-controlled gripper converges to near-zero tracking error, which means near-zero squeeze force                  | Threshold-triggered tightening on the client                     |
| Picks the wrong brand         | Background correlation plus color-destroying augmentation                                                               | Remove hue jitter; vary layouts; re-collect 1,200 episodes       |
| Door not fully closed         | Base drift at the end of the task was never covered in demonstrations                                                   | Add demonstrations with base variation                           |

### Beyond more data

Two lines of work target failures that more demonstrations fix only slowly.

- **Learning from the robot's own rollouts.** Every deployment run is logged with frames, joints and outcome, both successes and failures. Following a RECAP-style recipe, successes and failures get different rewards, so the policy learns from how it actually behaves.
- **Surgical weight editing.** For a recurring failure, I compare failing states with phase-matched successful states (nearest neighbors in joint space, so task phase isn't confused with outcome) and rank attention heads and MLP neurons by effect size and gradient attribution. Then only $$K = 4$$ scalar gains are fitted, with the other 3.6B parameters frozen:

$$
\min_{g}\ \mathcal{L}_{\text{fail}}(g) \;+\; \lambda\, \mathcal{L}_{\text{keep}}(g) \;+\; \mu \sum_{k} (g_k - 1)^2
$$

$$\mathcal{L}_{\text{fail}}$$ is the flow-matching loss on failing states with the nearest expert action as pseudo-label, and $$\mathcal{L}_{\text{keep}}$$ is the loss on random demonstrations, so behavior elsewhere doesn't move. The gains fold losslessly back into the output projections, so the edited checkpoint deploys like any other. Both lines are in progress.

### Results (real robot)

| Test                   | Result                        |
| ---------------------- | ----------------------------- |
| First full evaluation  | 92 / 109 (84.4%)              |
| Current demo           | ~93%, target 98%              |
| Unseen drink brands    | 4 / 4                         |
| Color instructions     | 6 / 6 (never-seen objects)    |
| Left / right wording   | 3 / 4                         |
| Ready posture          | 17 s → 6 s                    |

<div class="row mt-3 justify-content-center">
  <div class="col-sm-8">
    <video src="{{ '/assets/video/work/home_ready_pose.mp4' | relative_url }}" poster="{{ '/assets/img/work/home_ready_pose.jpg' | relative_url }}" class="img-fluid rounded z-depth-1" autoplay muted loop playsinline controls preload="metadata"></video>
  </div>
</div>
<div class="caption">
  Ready posture redesigned so the waist, legs and head move in parallel: 17 s → 6 s, with less sway and a clearer view. It also gives mapping and calibration a fixed starting pose.
</div>

### From policy to product

The skill is exposed as an **MCP tool** per drink category. An LLM agent clarifies which drink the user wants by voice and calls the matching tool, and the chain runs end to end. Open items are honest ones: reset stability after a failure, the transition from grasp to handoff, navigation docking distance, and CPU contention when every service runs on the robot. Small generalization samples are reported as raw counts on purpose.
