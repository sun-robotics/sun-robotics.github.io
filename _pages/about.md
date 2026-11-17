---
layout: about
title: about
permalink: /
alt_url: /zh/
subtitle: Robot Learning Engineer at X-Humanoid · Making generalist robot policies dependable on real humanoids

profile:
  align: right
  image: work/profile_tienyi.jpg
  image_circular: false # crops the image to make it circular
  more_info:

selected_papers: false # includes a list of papers marked as "selected={true}"
social: true # includes social icons at the bottom of the page

announcements:
  enabled: true # includes a list of news items
  scrollable: true # adds a vertical scroll bar if there are more than 3 news items
  limit: 5 # leave blank to include all the news in the `_news` folder

latest_posts:
  enabled: false
  scrollable: true # adds a vertical scroll bar if there are more than 3 new posts items
  limit: 3 # leave blank to include all the blog posts
---

I build the layer between a robot foundation model and a robot people can rely on.

At **X-Humanoid**, I lead whole-body VLA deployment on the Tien Yi humanoid for long-horizon household tasks: open the fridge, pick the right drink from look-alikes, close the door, hand it over. I own the full loop — **data engine → policy training (π0.5, world action models) → real-time execution → runtime guardrails → agent interface** — and every number on this site was measured on the real robot.

> Benchmarks reward policies that succeed on average. Deployment punishes the ones that fail silently.

That gap drives three questions I work on:

- **Reliability outside the policy.** A VLA has no notion of "I am about to grab the wrong thing". Can a lightweight, policy-agnostic runtime layer supply that judgment at control rate, without retraining?
- **Latency as a control problem.** A 3B-parameter policy in the cloud answers in about 480 ms, and the robot moves at 26 Hz. How do you make the delay invisible instead of hiding it with stops?
- **What future does control need?** World action models learn physics from video. When does imagining the future beat reacting to the present, and what is the smallest representation of the future worth computing?

<div class="row mt-4">
  <div class="col-sm-4 mt-3 mt-md-0">
    <a href="{{ '/projects/vla-guardrail/' | relative_url }}">
      <video src="{{ '/assets/video/work/harness_pi05_fix.mp4' | relative_url }}" poster="{{ '/assets/img/work/harness_pi05_fix.jpg' | relative_url }}" class="rounded z-depth-1" style="width: 100%; aspect-ratio: 4 / 5; object-fit: cover;" autoplay muted loop playsinline preload="metadata"></video>
    </a>
    <p class="mt-2 mb-0"><strong>Runtime guardrail</strong></p>
    <p class="small text-muted">Predicts a wrong-bottle grasp before contact and re-targets it. Policy-agnostic, no retraining.</p>
  </div>
  <div class="col-sm-4 mt-3 mt-md-0">
    <a href="{{ '/projects/whole-body-home/' | relative_url }}">
      <video src="{{ '/assets/video/work/home_full_task.mp4' | relative_url }}" poster="{{ '/assets/img/work/home_full_task.jpg' | relative_url }}" class="rounded z-depth-1" style="width: 100%; aspect-ratio: 4 / 5; object-fit: cover;" autoplay muted loop playsinline preload="metadata"></video>
    </a>
    <p class="mt-2 mb-0"><strong>Whole-body home manipulation</strong></p>
    <p class="small text-muted">One 20-DoF π0.5 policy over arms, waist and legs. 84% → 93% success.</p>
  </div>
  <div class="col-sm-4 mt-3 mt-md-0">
    <a href="{{ '/projects/realtime-execution/' | relative_url }}">
      <video src="{{ '/assets/video/work/rtc_fold_52_vs_30.mp4' | relative_url }}" poster="{{ '/assets/img/work/rtc_fold_52_vs_30.jpg' | relative_url }}" class="rounded z-depth-1" style="width: 100%; aspect-ratio: 4 / 5; object-fit: cover;" autoplay muted loop playsinline preload="metadata"></video>
    </a>
    <p class="mt-2 mb-0"><strong>Real-time execution</strong></p>
    <p class="small text-muted">Prefix-guided async chunking hides 480 ms of latency. Cloth folding 52 s → 30 s.</p>
  </div>
</div>

<div class="row mt-4 text-center">
  <div class="col-6 col-md-3"><h4 class="mb-0">93%</h4><p class="small text-muted">fridge-to-handoff demo success, up from 84.4%</p></div>
  <div class="col-6 col-md-3"><h4 class="mb-0">−42%</h4><p class="small text-muted">task time with asynchronous chunking</p></div>
  <div class="col-6 col-md-3"><h4 class="mb-0">2.9k</h4><p class="small text-muted">curated whole-body episodes, 2.95M frames</p></div>
  <div class="col-6 col-md-3"><h4 class="mb-0">20-DoF</h4><p class="small text-muted">one policy over dual arms, waist and legs</p></div>
</div>

### System at a glance

Each layer below is something I designed and shipped on the robot. The project pages go one level deeper.

| Layer         | What it does                                                                                            | Key idea                                                       |
| ------------- | ------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| Interface     | The fridge skill is an MCP tool. An LLM agent clarifies intent by voice and calls it                   | Skills are tools with names, not scripts                       |
| Guardrail     | Sidecar monitor that predicts the grasp landing point and corrects wrong-target reaches before contact | Judge intent mismatch, not failure                             |
| Execution     | Async prefix-guided chunking, seam blending, smoothing, joint and slew limits, 26 Hz telemetry         | Latency is a budget the controller spends                      |
| Policy        | π0.5 (3B) full fine-tune with relative actions; Fast-WAM and FLUX 3 Action as world-model alternatives | Pick the representation, then the model                        |
| Data engine   | SOP, rule and statistics QA, human review, stage annotation, rollout logging for failure mining        | Data quality is measured, not assumed                          |

Selected work is on the [projects]({{ '/projects/' | relative_url }}) page.
