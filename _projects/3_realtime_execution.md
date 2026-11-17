---
layout: page
title: Real-Time Execution of a Cloud-Served Policy
description: Turning 480 ms of network latency into a budget instead of a stall.
img: assets/img/work/rtc_fold_52_vs_30_thumb.jpg
importance: 3
category: work
permalink: /projects/realtime-execution/
alt_url: /zh/projects/realtime-execution/
related_publications: false
---

**TL;DR.** The policy runs in the cloud and the robot runs at 26 Hz. A naive client stops and waits on every request, and jerks whenever a new action chunk replaces the old one. I rebuilt execution around one idea: **latency is part of the control problem**. The result was a 42% faster task with no visible seams, and an execution stack now reused across VLA deployments.

<div class="row mt-3 justify-content-center">
  <div class="col-sm-8">
    <video src="{{ '/assets/video/work/rtc_fold_52_vs_30.mp4' | relative_url }}" poster="{{ '/assets/img/work/rtc_fold_52_vs_30.jpg' | relative_url }}" class="img-fluid rounded z-depth-1" autoplay muted loop playsinline controls preload="metadata"></video>
  </div>
</div>
<div class="caption">Same policy, same task, both at 2× speed. Blocking execution (left) vs asynchronous real-time chunking (right).</div>

### The numbers that frame the problem

The policy returns a chunk $$A = (a_0, \dots, a_{H-1})$$ with $$H = 50$$ steps of $$D = 20$$ joint targets. A round trip takes about 480 ms, so at $$f = 26$$ Hz the robot has executed

$$
d = \lceil \text{RTT} \cdot f \rceil \approx 13\text{–}14 \ \text{steps}
$$

of the old chunk by the time the new one arrives. There are two naive options, and both fail. Waiting for each response turns the delay into dead time (52 s for the task). Switching to the new chunk as soon as it lands causes a jump of up to 0.051 rad at the seam, because the new chunk was planned from a state the robot has already left.

### Prefix-guided asynchronous chunking

The client never blocks. While chunk $$A^{(n)}$$ executes, it requests $$A^{(n+1)}$$ and sends along what the server needs to stay consistent with the robot:

- the actions already committed for the next $$d$$ steps, which cannot change;
- an estimate of $$d$$, updated online from measured round trips;
- an execution horizon $$s = 8$$, the minimum number of new steps the client will consume before the following request.

On the server, the flow-matching sampler treats the committed prefix as an inpainting constraint. Each denoising step is guided toward the prefix with an exponential per-step weight schedule of the form

$$
w_i = \begin{cases} 1 & i < d \\ \exp\!\big(-\alpha\,(i-d)\big) & d \le i < H - s \\ 0 & i \ge H - s \end{cases}
$$

with the guidance strength capped at $$\beta = 5$$. The first $$d$$ steps are pinned, the middle decays smoothly, and the tail is free. The new chunk agrees with what the robot is already doing and stays free to react to the new observation.

**Hand off inside the constrained region.** The takeover index always falls inside the guided prefix, never in the free tail. What reaches the motors is a stitched timeline, not a sequence of whole chunks.

### Making the seam invisible

Guidance makes the seam small. Blending makes it zero. Let $$r = a^{\text{old}}_{k_0} - a^{\text{new}}_{k_0}$$ be the residual at the takeover step $$k_0$$. The first $$m = 8$$ steps of the new chunk are corrected with a raised-cosine fade:

$$
\tilde a_{k_0 + j} = a^{\text{new}}_{k_0 + j} + r \cdot \tfrac{1}{2}\big(1 + \cos(\pi j / m)\big), \quad j = 0, \dots, m
$$

This brings the 0.051 rad jump to 0 while touching only 8 steps. Inside each chunk, a per-joint Savitzky–Golay filter (window 7, order 2) removes high-frequency noise and keeps real large motions, which a moving average would flatten. Grippers are excluded, since their steps are intentional.

### The rest of the execution stack

| Stage                  | What it does                                                                                       |
| ---------------------- | -------------------------------------------------------------------------------------------------- |
| Interpolation          | C1-continuous 3× upsampling from policy rate to servo rate                                         |
| Safety                 | Per-joint limits for both arms, waist and legs; slew-rate limiting                                 |
| Re-anchoring           | If commanded and measured joints drift apart by more than 0.08 rad, the slew base snaps back to the measurement |
| Telemetry              | A background thread samples joints at 26 Hz, independent of the control loop                      |

The telemetry turned debugging from opinion into measurement. A local web view overlays, per joint: measured trajectory, every command actually sent, and every raw and smoothed chunk pinned to its arrival time, with RMS and max tracking error. It runs as a zero-change wrapper around the client, and all hardware reads share one lock, so sampling never races the vendor SDK.

<div class="row mt-3">
  <div class="col-sm mt-3 mt-md-0">
    <video src="{{ '/assets/video/work/rtc_jitter_before.mp4' | relative_url }}" poster="{{ '/assets/img/work/rtc_jitter_before.jpg' | relative_url }}" class="img-fluid rounded z-depth-1" autoplay muted loop playsinline controls preload="metadata"></video>
  </div>
  <div class="col-sm mt-3 mt-md-0">
    <video src="{{ '/assets/video/work/rtc_jitter_after.mp4' | relative_url }}" poster="{{ '/assets/img/work/rtc_jitter_after.jpg' | relative_url }}" class="img-fluid rounded z-depth-1" autoplay muted loop playsinline controls preload="metadata"></video>
  </div>
</div>
<div class="caption">Replaying logged chunks from the real robot. Left: the jitter reproduced. Right: after seam blending and in-chunk smoothing.</div>

### Outcome

- Cloth folding **52 s → 30 s (−42%)**. The seam jump went from 0.051 rad to 0, with clearly less jitter on the real robot.
- The same stack serves the 20-DoF fridge task and other VLA deployments. It is the default deployment path, not a one-off demo.
- A related fix: the startup self-test drove the waist and legs at 0.3 rad/s, and the stand routine pulled them back before they settled. That excited a lightly damped 3.5 Hz torso sway for 1.5 s. Sequencing the two removed it.

**Open issue.** When consecutive chunks disagree strongly, for example at a decision point, guidance has to choose between consistency and reactivity. Next is making the decay rate $$\alpha$$ and the horizon $$s$$ adaptive to measured chunk disagreement.
