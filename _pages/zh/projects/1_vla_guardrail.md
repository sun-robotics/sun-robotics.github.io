---
layout: page
title: 冻结 VLA 的运行时护栏
description: 在接触之前识别并纠正"抓错目标"，无需重训策略。
img: assets/img/work/harness_pi05_fix_thumb.jpg
importance: 1
zh_project: true
lang: zh
permalink: /zh/projects/vla-guardrail/
alt_url: /projects/vla-guardrail/
---

**一句话总结。** 在我们的人形机器人上，代价最高的 VLA 失败不是崩溃，而是**自信、流畅地抓错了对象**。我做了一层与策略无关的护栏，它以控制频率在 VLA 旁边运行，预测夹爪即将落在哪里，在接触前把手臂纠正过来。无需重训，策略正确时完全不干预。

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
  左：指令是拿橙汁，π0.5 却伸向相邻的瓶子；护栏在接触前介入，后撤并重新对准。中：同一护栏用在另一个策略（Pelican-VLA）上。右：策略判断正确，护栏不做任何动作。
</div>

### 没有任何报警的失败

冰箱任务的长程失败并不平均分布，而是集中在几个关键动作上，其中最难的是**选错对象**。常见的失败检测看的是打滑、空抓、进度停滞或力异常；抓错目标时这些信号一个都不会触发：手没打滑，力正常，任务还在推进，只是对象错了。

策略自己也看不出来。输入 224×224、patch 14 像素时，一个视觉 token 在货架上约对应 **4.8 cm**，而相邻饮料中心间距只有 **8–10 cm**，也就是只差 1–2 个 token。这是输入表示的信息容量问题，而不是模型能力问题。任何复用策略同一套感知的监督器，都会继承同样的盲区。

另外两个事实决定了设计方向：

- **遮挡是机理，不是偶发。** 伸手过程中，机器人自己的手臂会挡住头部相机里的目标。持续用最新画面更新判断的监督器，恰恰会在最关键的时刻被污染。
- **有些失败不可逆。** 夹爪一旦落进两瓶之间的缝隙，两瓶会一起被推倒，事后检测毫无意义，只能事前预测。

### 架构

护栏是 VLA 客户端旁边的一个旁路进程：读取相机画面和关节状态，必要时接管作业臂；从不修改策略的输入或权重。

| 组件         | 作用                                                                 |
| ------------ | -------------------------------------------------------------------- |
| 独立感知通道 | 独立于策略 224×224 链路的全分辨率检测服务，定位所有候选              |
| 一次性参照   | 在所有候选同时可见的时刻，把目标身份与位置写入参照，随后保持不变     |
| 落点预测     | 外推夹爪运动的停止位置，并与参照比对                                 |
| 显式修正     | 作业臂回到安全位姿，对准参照目标，再交还控制权                       |
| 安全联锁     | 阶段门控、可达性、身体构型一致性、步进合理性检查                     |

**预测，而不是检测。** 根据最近几帧夹爪的横向位置 $$y_t$$ 和速度 $$\dot y_t$$，估计停止点 $$\hat y_{\text{stop}} = y_t + \dot y_t\,\tau$$（$$\tau$$ 为剩余接近时间），再与目标槽位 $$[y^\ast - \delta,\ y^\ast + \delta]$$ 以及相邻候选之间的缝隙比较。只要预测落在邻瓶或缝隙上，手臂还在几厘米之外时就会触发介入。

**显式修正，平滑交还。** 修正是几何式的：沿已知安全路径后撤，向参照目标平移，再继续执行。它不会让策略基于当前画面重新规划，因为当前画面正是被遮挡的那些。交还控制权时，会丢弃在途的动作块并重置平滑器状态，否则机器人会朝过时的计划回弹。

**知道何时放弃。** 护栏只在触发阶段启用，等 VLA 真正出手后才介入，夹爪闭合即解除。起点过远、腰角与参照不一致（仅 0.3 rad 的腰角偏差就会抓空）或步进异常时，直接放弃。"什么都不做"也是合法输出。

### 上线流程

介入层的可信度必须在真机上一点点挣来。护栏分两种模式上线：

1. **观察模式。** 只记录每次判定（目标、预测落点、决策），不向机器人发任何指令。把记录的判定和实际结果对照，直到误报率低到可以接受。
2. **执行模式。** 这之后才允许它动手臂，而且只动作业臂和夹爪，其余关节保持 VLA 的指令。

整层逻辑做了异常隔离：任何内部错误只记录、不抛出，绝不会打断主控制循环。

### 结果

- 冰箱取饮到递送的演示成功率从 **84.4%**（92/109 次）提升到约 **93%**，这是护栏与[数据和模型修复](/zh/projects/whole-body-home/)共同作用的结果。
- 同一组件在 **π0.5 与 Pelican-VLA** 上无需改动即可工作。
- 失败恢复覆盖三类情况：拿错瓶或空抓（后撤并重新对准）、抓取失败（姿态复位后重试）、手臂未收回（先回收再继续）。

### 与现有工作的区别

现有的恢复方法大多由"失败"触发：回退重试、由 VLM 监督者选择恢复策略，或者由 LLM Agent 把 VLA 封装成原语、每次调用后检查结果。这些方法要么对不可逆事件来得太晚，要么单次决策就要秒级，远跟不上 26 Hz 的控制回路。这层护栏针对的是更早的阶段：**在失败发生之前识别意图偏离**，以控制频率运行，并且 VLA 始终保持主控。

**下一步。** 从货架取物推广为按阶段配置的统一框架（开门、抓取、搬运、放置、回撤），每个阶段声明自己的参照、预测器和允许的修正方式，在同一套交接约定下运行。
