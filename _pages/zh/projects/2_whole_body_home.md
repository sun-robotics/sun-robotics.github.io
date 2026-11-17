---
layout: page
title: 面向家庭长程任务的全身 VLA
description: 开冰箱、拿对饮料、关门、递送，在 20 自由度移动人形机器人上完成。
img: assets/img/work/home_unseen_objects_thumb.jpg
importance: 2
zh_project: true
lang: zh
permalink: /zh/projects/whole-body-home/
alt_url: /projects/whole-body-home/
---

**一句话总结。** 我把天轶人形机器人上的一个家庭任务，从第一批示教数据一路推进到接入 LLM Agent 的稳定演示：开冰箱，弯腰，从外观相近的饮品中拿对指定的那一瓶，关门，递给人。一个 π0.5 策略统一控制双臂、腰和腿（20 自由度）。大部分提升来自把数据和调试当作工程问题来做，而不是换更大的模型。

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
  左：移动底盘上的完整任务（2 倍速）。中：训练中从未出现过的饮品。右：未见过品牌的瓶装水（2 倍速）。
</div>

### 难点

- **是全身，不只是双臂。** 机器人在任务中途需要弯腰、转腰：每条手臂 7 维加 1 维夹爪，再加腰 2 维、腿 2 维，共 20 维动作空间。腰部误差会传到手上：腰角偏差 0.3 rad 时，即使手臂关节完全正确也会抓空。
- **目标外观相近。** 四瓶饮料并排摆放，错的那瓶只差几厘米，主要靠颜色区分。
- **自遮挡。** 手臂恰好在策略最需要看清的时候挡住头部相机。
- **长程。** 一条示教约 1000 帧（27 fps），早期的小错误（比如门没关严）要到最后才暴露。

### 训练方案

| 选择     | 设置                                                         | 原因                                                   |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------ |
| 基座     | π0.5（约 3B），含视觉塔全量微调                              | 需要区分的视觉细节是任务特有的                         |
| 动作空间 | 关节相对动作，两个夹爪用绝对量                               | 消除每条数据的起始位姿偏移；夹爪本质上是二值的         |
| 归一化   | 状态与动作都用分位数（q01 / q99）                            | 对腰腿偶发的大幅动作更鲁棒                             |
| 动作块   | 50 步，全部执行                                              | 与[实时执行框架](/zh/projects/realtime-execution/)配合 |
| 规模     | 8× A800 DDP，全局 batch 256，bf16，torch.compile             | 与 OpenPI 常用的微调 batch 对齐                        |
| 调度     | lr 2.5e-5，1k 步 warm-up，30k 步余弦降到 2.5e-6（约 2.8 epoch） | 留出 5% 的 episode 观察何时开始过拟合               |
| 数据增强 | GPU 端仿射、亮度、对比度、轻度饱和度（±15%），**不做色相**   | 默认色相扰动会把橙汁"变成"可乐                         |

数据增强这一行是真实踩过的坑。默认配方（色相 ±0.05、饱和度 0.5–1.5）在多数任务上无害，在这里却抹掉了唯一能区分类别的特征。

### 把数据当作工程系统

我制定了采集 SOP，把数据集扩充到 **2,934 条（295 万帧）**，覆盖 20 种任务指令，变化维度包括饮品种类、货架槽位和底盘位置。每条数据先经过自动质检，再人工复核：

| 检查项       | 规则示例                                                     |
| ------------ | ------------------------------------------------------------ |
| 夹爪二值性   | 半开状态停留超过 2 s 判为失败                                |
| 夹爪抖动     | 5 s 内开合切换 3 次及以上判为失败                            |
| 关节跳变     | 单帧跳变超过 0.1 rad 标记                                    |
| 模式离群     | 开合次数、时刻与轨迹形状偏离众数过远（z-score）标记         |
| 槽位标注     | 按采集日对抓取点聚类，找出目标瓶放错位置的数据               |

在首批数据上，质检流水线标出了 **229 条中的 54 条**（27 条失败，27 条警告）。复核人员在网页界面里逐条确认或改判，清洗后的数据集带清单重新导出。26 fps 与 27 fps 混采的数据先重新对齐时间轴再合并。Qwen3-VL 自动标注子任务阶段（开门、伸手、抓取、取出、关门），成功率可以按阶段统计，而不是只看一个总数。

### 按根因调试，而不是盲目重训

| 现象             | 根因                                                                                         | 解决                                         |
| ---------------- | -------------------------------------------------------------------------------------------- | -------------------------------------------- |
| 1、4 号位偏低    | 转腰到位的关键决策帧，手臂扫过头部相机视野，遮挡 1–3 号位；4 号位平均集长比 2 号位长 18%，轨迹方差最大 | 调整示教动作；调整训练步数         |
| 搬运途中滑落     | 位置控制的夹爪跟踪误差趋近于 0，等于夹持力趋近于 0                                           | 客户端设阈值，触发主动夹紧                   |
| 拿错品牌         | 背景伪相关，加上破坏颜色的数据增强                                                           | 去掉色相扰动；变换摆放；补采 1200 条         |
| 冰箱门关不严     | 任务末段底盘漂移，示教中从未覆盖                                                             | 补充带底盘变化的示教                         |

### 不止于加数据

有两类失败，单靠加示教数据改善得很慢，我从两个方向处理：

- **从机器人自己的执行中学习。** 每次部署都记录画面、关节和结果，成功和失败都保留。参考 RECAP 的思路，对成功和失败给予不同奖励，让策略从自己的真实表现中学习。
- **精确的权重编辑。** 针对反复出现的失败，把失败状态与"相位匹配"的成功状态做对比（在关节空间取最近邻，避免把任务阶段差异误当成成败差异），按效应量和梯度归因给注意力头与 MLP 神经元排序。然后只拟合 $$K = 4$$ 个标量增益，其余 36 亿参数全部冻结：

$$
\min_{g}\ \mathcal{L}_{\text{fail}}(g) \;+\; \lambda\, \mathcal{L}_{\text{keep}}(g) \;+\; \mu \sum_{k} (g_k - 1)^2
$$

其中 $$\mathcal{L}_{\text{fail}}$$ 是失败状态上的 flow-matching 损失，以最近邻专家动作为伪标签；$$\mathcal{L}_{\text{keep}}$$ 是随机示教帧上的损失，保证其他行为不变。增益可以无损折叠回输出投影矩阵，编辑后的 checkpoint 和普通模型一样部署。这两个方向都在进行中。

### 真机结果

| 测试               | 结果                       |
| ------------------ | -------------------------- |
| 首次完整评测       | 92 / 109（84.4%）          |
| 当前演示           | 约 93%，目标 98%           |
| 未见过的饮品品牌   | 4 / 4                      |
| 颜色指令           | 6 / 6（完全没见过的物品）  |
| 左 / 右方位指令    | 3 / 4                      |
| 准备姿态耗时       | 17 s → 6 s                 |

<div class="row mt-3 justify-content-center">
  <div class="col-sm-8">
    <video src="{{ '/assets/video/work/home_ready_pose.mp4' | relative_url }}" poster="{{ '/assets/img/work/home_ready_pose.jpg' | relative_url }}" class="img-fluid rounded z-depth-1" autoplay muted loop playsinline controls preload="metadata"></video>
  </div>
</div>
<div class="caption">
  重新设计准备姿态，让腰、腿和头部并行动作：17 s → 6 s，晃动更小、视野更清楚。它同时为建图和点位标定提供了固定起点。
</div>

### 从策略到产品

技能按饮品品类封装为 **MCP 工具**：LLM Agent 通过语音澄清用户要哪种饮料，再调用对应工具，整条链路端到端跑通。尚待解决的问题如实列出：失败后复位的稳定性、拿到饮料与递送之间的衔接、导航停靠距离，以及所有服务都跑在机器人上时的 CPU 争用。泛化测试样本较小，因此刻意给出原始计数。
