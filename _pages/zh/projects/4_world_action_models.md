---
layout: page
title: 真实世界中的世界动作模型
description: '“想象未来”什么时候能帮机器人做得更好？'
img: assets/img/work/wam_pred_vs_gt_thumb.jpg
importance: 4
zh_project: true
lang: zh
permalink: /zh/projects/world-action-models/
alt_url: /projects/world-action-models/
---

**一句话总结。** 世界动作模型（WAM）把视频扩散模型和动作专家耦合在一起，让策略继承视频预训练中的物理先验。我把 Fast-WAM 部署到真实的双臂三相机叠衣任务上，与 VLA 正面对比；又设计了一个检索条件化的变体，让模型拥有一段紧凑的"成功示教记忆"。背后的问题是：**策略到底需要看到多少"未来"？**

<div class="row mt-3 justify-content-center">
  <div class="col-sm-10">
    <video src="{{ '/assets/video/work/wam_pred_vs_gt.mp4' | relative_url }}" poster="{{ '/assets/img/work/wam_pred_vs_gt.jpg' | relative_url }}" class="img-fluid rounded z-depth-1" autoplay muted loop playsinline controls preload="metadata"></video>
  </div>
</div>
<div class="caption">想象与现实：三路相机下，模型预测的画面（PRED）与真实发生的画面（GT）并排对比。</div>

### 一段话讲清模型

Fast-WAM 用共享注意力的 Mixture-of-Transformers 耦合 Wan2.2 Video DiT 与 Action DiT，两者都用 flow matching 训练，分别作用于未来视频 latent 和动作块：

$$
\mathcal{L} = \mathcal{L}_{\text{video}} + \lambda\,\mathcal{L}_{\text{action}}
$$

结构化注意力掩码只允许动作 token 读取**当前帧**的视频 token，永远看不到真实的未来帧。推理时 Video DiT 只对当前帧跑一次并缓存 K/V，动作在这份缓存上去噪，不生成未来视频。它的兄弟变体 IDM 反其道而行：先生成未来画面，再从画面反推动作。

### 真机上的发现

- **提速不需要测试时想象。** Fast-WAM 用三路相机闭环运行，延迟在控制频率可承受的范围内，在我们的硬件上验证了原论文的结论。
- **数据效率。** 只用视频预训练、没有任何具身预训练，就能达到或超过在同样任务数据上微调的 π0.5。
- **扰动下，想象有用。** 用 IDM 时，有人把叠好的部分拆开，机器人会先叠右边再叠左边，然后继续；VLA 面对同样的扰动，会把这一步当成已经完成而直接跳过。想象目标状态让策略对场景做出反应，而不是回放动作序列。
- **即使失败了，模型仍在"想象成功"。** 一次抓取失败后，预测画面里显示的仍是成功抓取。世界模型编码的是"应该发生什么"。这引出一个问题：评价世界模型时，应该看它对控制的贡献，而不只是视频保真度。

<div class="row mt-3">
  <div class="col-sm mt-3 mt-md-0">
    <video src="{{ '/assets/video/work/wam_perturb.mp4' | relative_url }}" poster="{{ '/assets/img/work/wam_perturb.jpg' | relative_url }}" class="img-fluid rounded z-depth-1" autoplay muted loop playsinline controls preload="metadata"></video>
  </div>
  <div class="col-sm mt-3 mt-md-0">
    <video src="{{ '/assets/video/work/vla_perturb.mp4' | relative_url }}" poster="{{ '/assets/img/work/vla_perturb.jpg' | relative_url }}" class="img-fluid rounded z-depth-1" autoplay muted loop playsinline controls preload="metadata"></video>
  </div>
</div>
<div class="caption">同样的扰动，均为 3 倍速。左：WAM-IDM 重新完成被拆开的那一折；右：VLA 跳过了这一步。</div>

### Demo Memory：通往过去的低带宽通道

长程叠衣需要知道自己**处在任务的哪个阶段**，而语言和本体感觉回答不了这个问题。我在不改动 DiT 主干的前提下加了一条检索通路。

**记忆库。** 以固定关键帧步长扫描成功示教，每个关键帧 $$k$$ 存三个与模型自身接口对齐的量：首帧的池化 Wan-VAE latent $$z_k$$、归一化本体感觉 $$q_k$$、归一化动作 $$a_k$$。用与训练相同的 VAE，就不需要额外的视觉编码器，也没有域偏移。

**检索。** 部署时把当前观测编码为 $$z_q$$，按余弦相似度匹配，并约束在锁定的 episode 内只能向前推进：

$$
k^\ast = \arg\max_{k \,\in\, \mathcal{E},\ k \ \ge\ k_{\text{prev}} - \epsilon} \ \frac{z_q^\top z_k}{\lVert z_q\rVert\,\lVert z_k\rVert}
$$

约束内没有候选时退回全局检索，避免卡死。目标取 $$k^\ast + \ell$$（$$\ell$$ 为前瞻步数）。

**固定长度的四个 token。** 一个小编码器把命中结果映射为四个文本维 token，拼接在 T5 context 之后：视觉目标 $$z_{\text{goal}}$$、目标位姿 $$q_{\text{goal}}$$、动作提示 $$a_{\text{hint}}$$（可屏蔽）、跟踪误差 $$e = q_{\text{now}} - q_{\text{goal}}$$。交叉注意力长度恒定；关掉开关，模型就完全等同于 Fast-WAM。

**训练与推理同构。** 训练时不做检索，而是从当前视频窗口内的真实未来帧构造同样的四个 token 作为 teacher，梯度可以传回编码器；部署时从真实示教中检索。两端产出同一套接口。记忆 dropout（p = 0.1）保证检索失误时模型仍能工作。

**让它训得动。** 每个样本三路 H.264 解码会让 GPU 吃不饱，于是我把解码换成 torchcodec（pyav 只能 seek 到关键帧，每个样本都要重解整个 GOP），并让 worker 常驻、加深预取。另外把 batch 固定为定长，避免 torch.compile 在尾 batch 重新编译，并把数据等待时间作为一级监控指标。

### 下一步方向

我的工作假设是：控制需要的是**低维、可干预的未来**，而不是像素。具体来说，记 $$o_t, l, q_t$$ 为观测、指令和状态，$$v$$ 为未来视频，$$f$$ 为任务相关物体的运动流：

$$
p(a \mid o_t, l, q_t, v) \;\approx\; p(a \mid o_t, l, q_t, f)
$$

如果 $$f$$ 是真正的因果瓶颈，那么修改 $$f$$，动作应按可预测的方式改变；修改干扰物的运动，动作应保持不变；在 $$f$$ 之上再给像素，成功率不应再提升。这给"最小充分未来"提供了一个可证伪的检验，而不是又一个新架构。与此同时，我正在 20 自由度人形数据上微调 **FLUX 3 Action（7B）**，它联合去噪视频与动作：关节相对动作，32 步动作块，三相机拼接画布，FSDP + EMA。Demo Memory 的消融实验仍在进行，因此这里暂不声称提升。
