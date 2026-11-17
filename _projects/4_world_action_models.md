---
layout: page
title: World Action Models in the Real World
description: When does imagining the future help a robot act?
img: assets/img/work/wam_pred_vs_gt_thumb.jpg
importance: 4
category: work
permalink: /projects/world-action-models/
alt_url: /zh/projects/world-action-models/
related_publications: false
---

**TL;DR.** World action models (WAMs) pair a video diffusion model with an action expert, so the policy inherits physical priors from video pre-training. I brought Fast-WAM up on a real dual-arm, three-camera folding task, compared it head-to-head with a VLA, and designed a retrieval-conditioned variant that gives the model a compact memory of successful demonstrations. The question behind it: _what is the minimum future a policy needs to see?_

<div class="row mt-3 justify-content-center">
  <div class="col-sm-10">
    <video src="{{ '/assets/video/work/wam_pred_vs_gt.mp4' | relative_url }}" poster="{{ '/assets/img/work/wam_pred_vs_gt.jpg' | relative_url }}" class="img-fluid rounded z-depth-1" autoplay muted loop playsinline controls preload="metadata"></video>
  </div>
</div>
<div class="caption">Imagined vs real. Frames the model predicts (PRED) next to what actually happened (GT), across all three cameras.</div>

### The model in one paragraph

Fast-WAM couples a Wan2.2 Video DiT and an Action DiT through a Mixture-of-Transformers with shared attention. Both are trained with flow matching, one on future video latents and one on action chunks:

$$
\mathcal{L} = \mathcal{L}_{\text{video}} + \lambda\,\mathcal{L}_{\text{action}}
$$

A structured attention mask lets action tokens read only the _current_ frame's video tokens, never ground-truth future frames. At test time the Video DiT runs once on the current frame and caches its keys and values, and the action is denoised against that cache. No future video is generated. Its sibling, IDM, does the opposite: it generates future frames first, then infers actions from them.

### Findings on the real robot

- **No test-time imagination needed for speed.** Fast-WAM ran closed-loop from three cameras at a latency the control rate can absorb. It confirms the paper's claim on our own hardware.
- **Data efficiency.** Starting only from video pre-training, with no embodied pre-training, it matched or beat π0.5 fine-tuned on the same task data.
- **Imagination helps under perturbation.** With IDM, the robot recovered when a person undid a fold: it folded right before left and continued. The VLA treated the same perturbation as a step already done and moved on. Imagining the goal state makes the policy react to the scene, not replay a sequence.
- **The model imagines success, even after failure.** After a missed grasp, the predicted frames still show a successful one. The world model encodes what _should_ happen, which raises the question of how to evaluate world-model quality for control rather than for video fidelity.

<div class="row mt-3">
  <div class="col-sm mt-3 mt-md-0">
    <video src="{{ '/assets/video/work/wam_perturb.mp4' | relative_url }}" poster="{{ '/assets/img/work/wam_perturb.jpg' | relative_url }}" class="img-fluid rounded z-depth-1" autoplay muted loop playsinline controls preload="metadata"></video>
  </div>
  <div class="col-sm mt-3 mt-md-0">
    <video src="{{ '/assets/video/work/vla_perturb.mp4' | relative_url }}" poster="{{ '/assets/img/work/vla_perturb.jpg' | relative_url }}" class="img-fluid rounded z-depth-1" autoplay muted loop playsinline controls preload="metadata"></video>
  </div>
</div>
<div class="caption">Same perturbation, both at 3× speed. Left: WAM-IDM re-does the undone fold. Right: the VLA skips it.</div>

### Demo Memory: a low-bandwidth channel to the past

Long-horizon folding needs to know _where in the task_ it is, and language plus proprioception doesn't say. I added a retrieval path without touching the DiT backbone.

**Memory bank.** Successful demonstrations are scanned at a fixed keyframe stride. Each keyframe $$k$$ stores three quantities aligned with the model's own interfaces: a pooled Wan-VAE latent $$z_k$$ of the first frame, the normalized proprioception $$q_k$$, and the normalized action $$a_k$$. Using the same VAE as training means no extra vision encoder and no domain gap.

**Retrieval.** At deployment, the current observation is encoded to $$z_q$$ and matched by cosine similarity, constrained to move forward within the locked episode:

$$
k^\ast = \arg\max_{k \,\in\, \mathcal{E},\ k \ \ge\ k_{\text{prev}} - \epsilon} \ \frac{z_q^\top z_k}{\lVert z_q\rVert\,\lVert z_k\rVert}
$$

If the constraint excludes everything, it falls back to global search instead of stalling. The target is the keyframe $$k^\ast + \ell$$ for a lookahead $$\ell$$.

**Four tokens, fixed length.** A small encoder maps the hit to four text-dim tokens appended to the T5 context: visual goal $$z_{\text{goal}}$$, target pose $$q_{\text{goal}}$$, action hint $$a_{\text{hint}}$$ (maskable), and tracking error $$e = q_{\text{now}} - q_{\text{goal}}$$. Cross-attention length stays constant. With the switch off, the model is exactly Fast-WAM.

**Train like you'll test.** Training doesn't retrieve. It builds the same four tokens from the ground-truth future inside the current video window, acting as a teacher, so gradients reach the encoder. Deployment retrieves from real demonstrations. Both sides produce the same interface. Memory dropout (p = 0.1) keeps the model working when retrieval misses.

**Making it trainable.** Three H.264 streams per sample starved the GPUs, so I switched decoding to torchcodec (pyav can only seek to keyframes and re-decodes a whole GOP per sample) and made workers persistent with deeper prefetch. I also made batches fixed-size so torch.compile doesn't recompile on the last batch, and logged data-wait time as a first-class metric.

### Where this is going

My working hypothesis is that control needs a **low-dimensional, intervenable future**, not pixels. Concretely, with $$o_t, l, q_t$$ the observation, instruction and state, $$v$$ the future video and $$f$$ the task-relevant object motion flow:

$$
p(a \mid o_t, l, q_t, v) \;\approx\; p(a \mid o_t, l, q_t, f)
$$

If $$f$$ is a true causal bottleneck, then editing $$f$$ should change the action predictably, editing distractor motion should leave it invariant, and adding pixels on top of $$f$$ should not improve success. That gives a falsifiable test of "minimal sufficient future" instead of another architecture. In parallel I'm fine-tuning **FLUX 3 Action (7B)**, which jointly denoises video and actions, on the 20-DoF humanoid data: joint-relative actions, 32-step chunks, a three-camera canvas, FSDP with EMA. Ablations for Demo Memory are in progress, so no gains are claimed here yet.
