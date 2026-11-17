---
layout: about
title: 关于
permalink: /zh/
lang: zh
alt_url: /
nav: false
subtitle: X-Humanoid 机器人学习工程师 · 让通用机器人策略在真实人形机器人上可靠落地

profile:
  align: right
  image: work/profile_tienyi.jpg
  image_circular: false
  more_info:

selected_papers: false
social: true

announcements:
  enabled: false

latest_posts:
  enabled: false
---

我做的是机器人基础模型与"人可以放心交给它干活的机器人"之间的那一层。

在 **X-Humanoid**，我负责天轶人形机器人在家庭场景的全身 VLA 落地，完成长程任务：打开冰箱，在外观相近的饮品中拿对那一瓶，关门，递给人。我负责完整闭环：**数据引擎 → 策略训练（π0.5、世界动作模型）→ 实时执行 → 运行时护栏 → Agent 接口**。本站所有结果都来自真机。

> Benchmark 奖励"平均成功"的策略，真实部署惩罚"无声失败"的策略。

这道鸿沟引出我关注的三个问题：

- **策略之外的可靠性。** VLA 不知道"我马上要抓错了"。能否用一层轻量、与策略无关的运行时监督，以控制频率补上这个判断，而且不重训模型？
- **把延迟当作控制问题。** 3B 参数的策略部署在云端，往返约 480 ms，机器人却以 26 Hz 运动。怎样让延迟不可见，而不是靠停顿来掩盖？
- **控制到底需要怎样的"未来"？** 世界动作模型从视频中学到物理规律。什么时候"想象未来"胜过"对当下反应"？值得计算的最小未来表示是什么？

<div class="row mt-4">
  <div class="col-sm-4 mt-3 mt-md-0">
    <a href="{{ '/zh/projects/vla-guardrail/' | relative_url }}">
      <video src="{{ '/assets/video/work/harness_pi05_fix.mp4' | relative_url }}" poster="{{ '/assets/img/work/harness_pi05_fix.jpg' | relative_url }}" class="rounded z-depth-1" style="width: 100%; aspect-ratio: 4 / 5; object-fit: cover;" autoplay muted loop playsinline preload="metadata"></video>
    </a>
    <p class="mt-2 mb-0"><strong>运行时护栏</strong></p>
    <p class="small text-muted">在接触前预判"要拿错瓶"，并把手臂纠正到正确目标。与策略无关，无需重训。</p>
  </div>
  <div class="col-sm-4 mt-3 mt-md-0">
    <a href="{{ '/zh/projects/whole-body-home/' | relative_url }}">
      <video src="{{ '/assets/video/work/home_full_task.mp4' | relative_url }}" poster="{{ '/assets/img/work/home_full_task.jpg' | relative_url }}" class="rounded z-depth-1" style="width: 100%; aspect-ratio: 4 / 5; object-fit: cover;" autoplay muted loop playsinline preload="metadata"></video>
    </a>
    <p class="mt-2 mb-0"><strong>全身家庭操作</strong></p>
    <p class="small text-muted">单个 20 自由度 π0.5 策略控制双臂、腰与腿，成功率 84% → 93%。</p>
  </div>
  <div class="col-sm-4 mt-3 mt-md-0">
    <a href="{{ '/zh/projects/realtime-execution/' | relative_url }}">
      <video src="{{ '/assets/video/work/rtc_fold_52_vs_30.mp4' | relative_url }}" poster="{{ '/assets/img/work/rtc_fold_52_vs_30.jpg' | relative_url }}" class="rounded z-depth-1" style="width: 100%; aspect-ratio: 4 / 5; object-fit: cover;" autoplay muted loop playsinline preload="metadata"></video>
    </a>
    <p class="mt-2 mb-0"><strong>实时执行</strong></p>
    <p class="small text-muted">前缀引导的异步分块执行吸收 480 ms 延迟，叠衣服 52 s → 30 s。</p>
  </div>
</div>

<div class="row mt-4 text-center">
  <div class="col-6 col-md-3"><h4 class="mb-0">93%</h4><p class="small text-muted">冰箱取饮到递送演示成功率（基线 84.4%）</p></div>
  <div class="col-6 col-md-3"><h4 class="mb-0">−42%</h4><p class="small text-muted">异步分块执行带来的任务耗时下降</p></div>
  <div class="col-6 col-md-3"><h4 class="mb-0">2.9k</h4><p class="small text-muted">经质检的全身示教数据（295 万帧）</p></div>
  <div class="col-6 col-md-3"><h4 class="mb-0">20-DoF</h4><p class="small text-muted">一个策略统一控制双臂、腰和腿</p></div>
</div>

### 系统全景

下面每一层都由我设计并在真机上交付，项目页会再深入一层。

| 层级       | 做什么                                                                         | 核心思想                       |
| ---------- | ------------------------------------------------------------------------------ | ------------------------------ |
| 接口层     | 冰箱取饮技能封装为 MCP 工具，LLM Agent 通过语音澄清意图后调用                  | 技能是有名字的工具，而不是脚本 |
| 护栏层     | 旁路监督预测夹爪落点，在接触前纠正抓错目标                                     | 判定"意图偏离"，而不是"失败"   |
| 执行层     | 异步前缀引导分块、接缝融合、平滑、关节与速率限制、26 Hz 遥测                   | 延迟是控制器可以支配的预算     |
| 策略层     | π0.5（3B）相对动作全量微调；Fast-WAM、FLUX 3 Action 作为世界模型路线          | 先选对表示，再选模型           |
| 数据引擎   | 采集 SOP、规则 + 统计质检、人工复核、阶段标注、真机 rollout 回收与失败挖掘     | 数据质量要被度量，而不是假设   |

### 动态

| 日期       | 进展                                                                                                       |
| ---------- | ---------------------------------------------------------------------------------------------------------- |
| 2026.11.13 | 冰箱取饮技能以 MCP 工具形式接入 LLM Agent 与语音链路，端到端跑通，演示成功率约 93%                         |
| 2026.11.06 | [运行时护栏](/zh/projects/vla-guardrail/)在接触前纠正抓错目标，同一组件在 π0.5 与 Pelican-VLA 上均有效     |
| 2026.10.23 | [全身 π0.5](/zh/projects/whole-body-home/) 能抓取未见过的饮品品牌，并理解颜色与左右方位指令              |
| 2026.09.24 | [异步分块执行](/zh/projects/realtime-execution/)让真机叠衣服从 52 s 缩短到 30 s，且无接缝抖动             |
| 2026.09.17 | [世界动作模型](/zh/projects/world-action-models/)真机闭环：被人为打乱后会重排叠衣步骤，而不是照搬示教      |

更多内容见[项目](/zh/projects/)页。
