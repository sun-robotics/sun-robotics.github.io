---
layout: page
title: Runtime Guardrails for Frozen VLAs
description: Catching wrong-target grasps before contact, without retraining the policy.
img: assets/img/work/harness_pi05_fix_thumb.jpg
importance: 1
category: work
permalink: /projects/vla-guardrail/
alt_url: /zh/projects/vla-guardrail/
related_publications: false
---

**TL;DR.** The costliest VLA failures on our humanoid were not crashes. They were confident, smooth grasps of the _wrong_ object. I built a policy-agnostic guardrail that runs beside the VLA at control rate, predicts where the gripper is about to land, and corrects the arm before contact. It needs no retraining and stays silent when the policy is right.

<div class="row mt-3">
  <div class="col-sm mt-3 mt-md-0">
    <video src="{{ '/assets/video/work/harness_pi05_fix.mp4' | relative_url }}" poster="{{ '/assets/img/work/harness_pi05_fix.jpg' | relative_url }}" class="img-fluid rounded z-depth-1" autoplay muted loop playsinline controls preload="metadata"></video>
  </div>
  <div class="col-sm mt-3 mt-md-0">
    <video src="{{ '/assets/video/work/harness_pelican_fix.mp4' | relative_url }}" poster="{{ '/assets/img/work/harness_pelican_fix.jpg' | relative_url }}" class="img-fluid rounded z-depth-1" autoplay muted loop playsinline controls preload="metadata"></video>
  </div>
  <div class="col-sm mt-3 mt-md-0">
    <video src="{{ '/assets/video/work/harness_no_intervene.mp4' | relative_url }}" poster="{{ '/assets/img/work/harness_no_intervene.jpg' | relative_url }}" class="img-fluid rounded z-depth-1" autoplay muted loop playsinline controls preload="metadata"></video>
  </div>
</div>
<div class="caption">
  Left: asked for orange juice, π0.5 heads for the neighboring bottle. The guardrail steps in before contact, backs off and re-targets. Middle: the same guardrail on a different policy (Pelican-VLA). Right: the policy is correct, so the guardrail does nothing.
</div>

### The failure nobody flags

Long-horizon failures on the fridge task were not spread evenly. They clustered at a few decisive actions, and the hardest one was choosing the wrong object. Failure detectors look for slips, empty grasps, stalled progress or abnormal force. A wrong-target grasp triggers none of them: the hand doesn't slip, the force is normal and the task keeps moving.

The policy also can't see the mistake. At 224×224 input with 14-px patches, one visual token covers about **4.8 cm** of the shelf. Adjacent bottles are **8–10 cm** apart, which is only 1–2 tokens. This is a limit of the input representation, not of model capacity, so any monitor that reuses the policy's own perception inherits the same blind spot.

Two more facts shape the design:

- **Occlusion is structural.** During the reach, the robot's own arm covers the target in the head camera. A monitor that keeps updating its belief from the latest frames gets corrupted exactly when it matters.
- **Some failures are irreversible.** If the fingers land in the gap between two bottles, both get knocked over. Detecting that afterwards is useless. It has to be predicted.

### Architecture

The guardrail is a sidecar process next to the VLA client. It reads camera frames and joint states and can take over the working arm. It never modifies the policy's inputs or weights.

| Component                     | Role                                                                                                               |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| Independent perception        | A full-resolution detector service, separate from the policy's 224×224 pipeline, localizes candidates             |
| Reference built once          | When all candidates are visible, the target's identity and position are written into a reference and then held fixed |
| Landing-point predictor       | Extrapolates gripper motion to where it will stop, and checks it against the reference                            |
| Explicit corrector            | Moves the working arm back to a safe pose, re-aligns to the reference target, hands control back                  |
| Interlocks                    | Phase gating, reachability, body-configuration consistency, step-size sanity                                      |

**Predict, don't detect.** From the last few frames of the gripper's lateral position $$y_t$$ and velocity $$\dot y_t$$, the guardrail estimates the stop point $$\hat y_{\text{stop}} = y_t + \dot y_t\,\tau$$, where $$\tau$$ is the remaining approach time. It compares that point to the target slot $$[y^\ast - \delta,\ y^\ast + \delta]$$ and to the gaps between neighbors. A prediction that lands on a neighbor or in a gap triggers intervention while the arm is still several centimeters away.

**Correct explicitly, hand back cleanly.** The correction is geometric: back off along a known-safe path, shift toward the reference target, resume. It does not ask the policy to re-plan from the current frames, because those frames are the occluded ones. When control returns, in-flight action chunks are dropped and the smoother's state is reset. Without that, the robot rebounds toward the stale plan.

**Know when to abstain.** The guardrail only arms at the trigger phase, waits until the VLA commits to a reach, and releases as soon as the gripper closes. It gives up if the start pose is too far, if the waist angle doesn't match the reference (a 0.3 rad mismatch alone causes a miss), or if any step looks abnormal. Doing nothing is a valid output.

### Bring-up protocol

Trust in an intervention layer has to be earned on hardware. The guardrail shipped in two modes:

1. **Observe.** It logs every judgment (target, predicted landing point, decision) and never commands the robot. Logged judgments are compared with what actually happened, until false alarms are rare enough to accept.
2. **Act.** Only then is it allowed to move the arm, and only the working arm and gripper. All other joints keep the VLA's commands.

The whole layer is exception-safe. Any internal error is logged and swallowed, so it can never break the main control loop.

### Outcome

- Demo success on fridge-to-handoff rose from **84.4%** (92/109 trials) to about **93%**, from this guardrail together with the [data and model fixes](/projects/whole-body-home/).
- The same component worked on **π0.5 and Pelican-VLA** without changes.
- Failure recovery now covers three classes: wrong bottle or empty grasp (back off and re-target), failed grasp (reset posture, retry), and arm not retracted (retract first, then continue).

### Positioning

Most recovery work is triggered by a _failure_: rollback-and-retry, a VLM supervisor that picks a recovery strategy, or an LLM agent that wraps the VLA as a primitive and checks results after each call. Those either act too late for irreversible events or run at seconds per decision, far from a 26 Hz loop. This guardrail targets the earlier regime, **intent mismatch before failure**, at control rate, while the VLA stays in charge.

**Next.** Generalize from shelf picking to a per-phase framework (open, grasp, transport, place, retract), where each phase declares its own reference, predictor and allowed corrections under one handoff contract.
