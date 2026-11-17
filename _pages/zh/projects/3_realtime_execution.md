---
layout: page
title: 云端策略的实时执行
description: 把 480 ms 网络延迟变成可支配的预算，而不是停顿。
img: assets/img/work/rtc_fold_52_vs_30_thumb.jpg
importance: 3
zh_project: true
lang: zh
permalink: /zh/projects/realtime-execution/
alt_url: /projects/realtime-execution/
---

**一句话总结。** 策略跑在云端，机器人以 26 Hz 运动。朴素的客户端每次请求都停下来等，新动作块替换旧块时还会抖一下。我围绕一个观点重建了执行层：**延迟本身就是控制问题的一部分**。结果是任务提速 42%、接缝不可见，这套执行框架也已在多个 VLA 部署中复用。

<div class="row mt-3 justify-content-center">
  <div class="col-sm-8">
    <video src="{{ '/assets/video/work/rtc_fold_52_vs_30.mp4' | relative_url }}" poster="{{ '/assets/img/work/rtc_fold_52_vs_30.jpg' | relative_url }}" class="img-fluid rounded z-depth-1" autoplay muted loop playsinline controls preload="metadata"></video>
  </div>
</div>
<div class="caption">同一策略、同一任务，均为 2 倍速。左：阻塞式执行；右：异步实时分块执行。</div>

### 先看问题的量级

策略每次返回一个动作块 $$A = (a_0, \dots, a_{H-1})$$，包含 $$H = 50$$ 步、每步 $$D = 20$$ 维关节目标。一次往返约 480 ms，在 $$f = 26$$ Hz 下，新块到达时机器人已经执行了旧块的

$$
d = \lceil \text{RTT} \cdot f \rceil \approx 13\text{–}14 \ \text{步}
$$

两种朴素做法都会失败。每次都等响应，延迟就变成纯粹的停顿（任务耗时 52 s）。新块一到就直接切换，接缝处会出现最大 0.051 rad 的跳变，因为新块是从机器人已经离开的状态规划出来的。

### 前缀引导的异步分块

客户端从不阻塞。执行块 $$A^{(n)}$$ 的同时就请求 $$A^{(n+1)}$$，并把服务端保持一致所需的信息一起发过去：

- 接下来 $$d$$ 步已经提交、不可更改的动作；
- 对 $$d$$ 的估计，根据每次实测的往返时间在线更新；
- 执行窗口 $$s = 8$$，即下一次请求前客户端至少消费的新步数。

服务端的 flow-matching 采样器把已提交前缀当作 inpainting 约束，每个去噪步都按如下形式的指数权重向前缀引导：

$$
w_i = \begin{cases} 1 & i < d \\ \exp\!\big(-\alpha\,(i-d)\big) & d \le i < H - s \\ 0 & i \ge H - s \end{cases}
$$

引导强度上限 $$\beta = 5$$。前 $$d$$ 步被钉死，中段平滑衰减，尾段完全自由。这样新块既与机器人正在执行的动作一致，又能对新观测做出反应。

**在约束区内交接。** 接管点始终落在被引导的前缀区内，而不是自由尾段。真正下发给电机的是一条拼接好的时间线，而不是一个个完整的动作块。

### 让接缝不可见

引导让接缝变小，融合让接缝归零。设接管步 $$k_0$$ 处的残差为 $$r = a^{\text{old}}_{k_0} - a^{\text{new}}_{k_0}$$，对新块的前 $$m = 8$$ 步做升余弦淡出修正：

$$
\tilde a_{k_0 + j} = a^{\text{new}}_{k_0 + j} + r \cdot \tfrac{1}{2}\big(1 + \cos(\pi j / m)\big), \quad j = 0, \dots, m
$$

只改动 8 步，就把 0.051 rad 的跳变压到 0。块内按关节做 Savitzky–Golay 滤波（窗长 7、二阶），去掉高频噪声，同时保留真实的大幅运动（滑动平均会把这些运动抹平）。夹爪不参与滤波，因为它的阶跃是有意为之。

### 执行层的其他部分

| 环节     | 作用                                                         |
| -------- | ------------------------------------------------------------ |
| 插值     | 从策略频率到伺服频率做 C1 连续的 3 倍上采样                  |
| 安全     | 双臂、腰、腿逐关节限位，并做速率限幅                         |
| 重新锚定 | 指令与实测关节偏差超过 0.08 rad 时，把限幅基准拉回实测值     |
| 遥测     | 后台线程以 26 Hz 独立采样关节状态，不依赖控制循环            |

遥测让调试从"凭感觉"变成"看测量"。本地网页按关节叠加显示三样东西：实测轨迹、每一拍真正下发的指令、按到达时刻贴到时间轴上的原始块与平滑块，并给出 RMS 与最大跟踪误差。它以零改动的方式包装客户端，所有硬件读取共用一把锁，采样线程不会与厂商 SDK 并发冲突。

<div class="row mt-3">
  <div class="col-sm mt-3 mt-md-0">
    <video src="{{ '/assets/video/work/rtc_jitter_before.mp4' | relative_url }}" poster="{{ '/assets/img/work/rtc_jitter_before.jpg' | relative_url }}" class="img-fluid rounded z-depth-1" autoplay muted loop playsinline controls preload="metadata"></video>
  </div>
  <div class="col-sm mt-3 mt-md-0">
    <video src="{{ '/assets/video/work/rtc_jitter_after.mp4' | relative_url }}" poster="{{ '/assets/img/work/rtc_jitter_after.jpg' | relative_url }}" class="img-fluid rounded z-depth-1" autoplay muted loop playsinline controls preload="metadata"></video>
  </div>
</div>
<div class="caption">回放真机记录的动作块。左：复现的抖动；右：加入接缝融合与块内平滑之后。</div>

### 结果

- 叠衣服 **52 s → 30 s（−42%）**；接缝跳变从 0.051 rad 降到 0，真机抖动明显减轻。
- 同一套执行层已用于 20 自由度的冰箱任务和其他 VLA 部署，是默认部署路径，而不是一次性的 demo。
- 相关修复：开机自检以 0.3 rad/s 推动腰腿，腿还没停稳，站立程序就把它反向拉回，激起约 3.5 Hz、持续 1.5 s 的弱阻尼躯干晃动。调整两者的执行时序后，晃动消失。

**待解决。** 当相邻动作块差异很大时（比如在决策点），引导需要在一致性和反应性之间取舍。下一步是让衰减率 $$\alpha$$ 与执行窗口 $$s$$ 根据实测的块间差异自适应调整。
