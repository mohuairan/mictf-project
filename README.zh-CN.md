<p align="center">
  <img src="assets/banner.png" alt="MICTF Banner" width="100%">
</p>

<h1 align="center">MICTF</h1>

<p align="center">
  <b>Morphology-Informed Contact Template Framework</b><br>
  一种面向异构灵巧手的在线接触维持重定向方法
</p>

<p align="center">
  <a href="https://mohuairan.github.io/mictf-project/"><img src="https://img.shields.io/badge/项目主页-Page-2ec4b6?logo=github&logoColor=white" alt="项目主页"></a>
  <a href="https://youtu.be/rv6dafFKCII"><img src="https://img.shields.io/badge/演示-YouTube-red?logo=youtube" alt="YouTube Demo"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/许可证-MIT-blue.svg" alt="License: MIT"></a>
  <img src="https://img.shields.io/badge/Python-3.10+-3776AB?logo=python&logoColor=white" alt="Python 3.10+">
  <img src="https://img.shields.io/badge/MuJoCo-3.x-orange" alt="MuJoCo 3.8">
  <img src="https://img.shields.io/badge/状态-论文准备中-yellow" alt="Status">
</p>

<p align="center">
  <b>语言：</b> <a href="README.md">English</a> | <b>简体中文</b>
</p>

---

## 概述

MICTF 是一种在线、无需训练数据的重定向方法，在精确接触建立*之后*维持指腹（pad）级接触。它引入形态学导出的**接触模板**作为操作者意图与机器人执行之间的中间表示：每个模板将机器人手型模型映射为接触角色、局部指腹坐标系，以及接触后的运动学约束。

框架的长期愿景是通过接触原语实现零样本跨具身遥操作。本页面反映当前进展，聚焦于位控型灵巧手与在线接触后维持。

<p align="center">
  <img src="assets\system_overview.png" alt="System Overview" width="100%">
</p>

## 为什么需要 MICTF

绝大多数在线重定向方法 —— DexPilot 式向量跟踪、关节复制、指尖 IK、AnyTeleop 式管线 —— 回答的都是"指尖该去哪里？"它们在自由空间里表现良好。可一旦两个指腹已经接触、操作者还在继续移动，这些方法都没有显式机制来维持接触面的持续贴合。结果就是接触在不知不觉中被撑开。

（接触丰富的优化方法 —— 接触不变控制、带接触约束的 relaxed-IK —— *确实*是接触感知的，但它们是离线规划器或离线轨迹优化器，并非在操作者实时运动下维持接触的在线重定向层。MICTF 针对的是后者这一工况。）

这并非实现问题，而是结构性的。一个仅被 3D 指尖向量约束的 4-DOF 机器人拇指，会留下一个未被约束的自由度 —— 指腹绕"拇指-指尖轴"的旋转。DexPilot 的平滑正则项把这个自由度约束在初始解附近，但不提供任何接触几何引导。MICTF 转而用任务几何吸收这个自由度：一个指腹法向对齐残差强制拇指指腹朝向对侧表面。原本需要被正则化掉的"运动学冗余"，反而成了*维持*接触的机制。

## 架构

| 层 | 职责 |
|---|---|
| **形态学导入** | 解析 MJCF 手部模型，推断指链，从网格面片自动提取指腹接触坐标系，导出每只手的运行时容差。 |
| **接触模板** | 离散可执行对象（张开 / 精确捏合 / 多点捏合 / 闭合），携带接触角色、局部指腹坐标系与求解器容差。 |
| **贝叶斯意图滤波** | 一个隐马尔可夫前向更新从 Quest 3/WebXR 观测中选择当前活跃模板，转移代价由机器人自身工作空间中的几何邻近度决定。 |
| **闭链投影** | 一个有界非线性最小二乘求解器在操作者持续移动的输入下维持所选接触；指腹法向残差通过接触几何消解拇指冗余。 |

当前执行后端假设关节可独立位控。面向耦合/欠驱动传动（如肌腱驱动、协同耦合手型）的执行器空间间接求解路径**正在开发中**，不包含在本轮评估范围内。

## 实验结果

使用真实 Quest 3/WebXR 日志在 MuJoCo 中回放评估。每只手的所有方法变体回放同一条 JSONL 输入，因此方法差异纯粹来自算法。

**接触后保持间隙（主指标）：**

| 手型 | MICTF | DexPilot 式 | 关节复制 | 指尖 IK |
|---|---|---|---|---|
| Wuji Right | **8.3 mm** | 40.3 mm | 42.4 mm | 45.8 mm |
| Allegro Right | **15.4 mm** | 51.0 mm | 48.2 mm | 55.1 mm |
| LEAP Right | **19.2 mm** | 44.4 mm | 47.3 mm | 49.0 mm |
| ORCA Right v1 | **8.9 mm** | 23.6 mm | 26.1 mm | 28.4 mm |
| Sharpa Wave | **13.5 mm** | 39.7 mm | 41.2 mm | 43.6 mm |

5 只手的平均任务帧间隙：**MICTF 8.3–19.2 mm**，DexPilot 式 **27.7–76.6 mm**，关节复制 **23.1–55.7 mm**。所有基线均无法建立稳定接触（所有手型上 Acquire = 0.000）；MICTF 在每只手上都能过渡到保持接触（Acquire 0.222–0.778）。DexPilot 在向量误差上更低（33.9–47.6 mm，MICTF 为 54.7–106.8 mm），这是预期内的 —— MICTF 以端点向量保真度换取指腹对齐。关闭闭链求解器后任务距离回落到基线水平，证实起作用的是投影层本身，而非仅仅是模板选择。

完整的数字、消融与每只手的求解器诊断将随论文一同公布。

## 支持的手型

同一套导入流水线覆盖异构的位控型灵巧手 —— 无需针对每只手硬编码：

| 手型 | DoF | 手指 | 模板数 | 验证 |
|---|---|---|---|---|
| Wuji Right | 20 | 5 | 12 | Quest 3 回放（完整基线） |
| Allegro Right | 16 | 4 | 9 | Quest 3 回放（完整基线） |
| LEAP Right | 16 | 4 | 9 | Quest 3 回放（完整基线） |
| ORCA Right v1 | 16 | 5 | 12 | Quest 3 回放（完整基线） |
| Sharpa Wave | 22 | 5 | 12 | Quest 3 回放（完整基线） |
| Shadow Right | 22 | 5 | 12 | 肌腱拓扑诊断 |
| Robotiq 2F-85 | 1 | 2 | — | 非对掌诊断 |

> **ORCA Right v1** 与 **Sharpa Wave** 在开发过程中**未经任何适配**即成功导入 —— 对这两款手型实现了真正的零样本接入。

## 演示

**四手型对比**（Wuji · Allegro · ORCA · Sharpa）—— 回放操作者运动下的闭链接触维持，1.5 倍速：

<p align="center">
  <img src="assets/multihand_demo.gif" alt="MICTF 四手型闭链接触对比" width="90%">
</p>
<p align="center"><em>每个面板：同一条回放的 Quest 3 输入，不同的手型形态。接触建立后，四只手均维持了指腹级接触。</em></p>

**一条轨迹，五只手** —— 同一条 Quest 3 手部追踪回放映射到五种不同形态（Wuji · Allegro · LEAP · ORCA · Sharpa），展示 MICTF 无需逐手调参即可在所有手型上建立稳定接触：

<p align="center">
  <a href="https://youtu.be/rv6dafFKCII">
    <img src="https://img.youtube.com/vi/rv6dafFKCII/maxresdefault.jpg" alt="MICTF 一轨迹五手型演示" width="70%">
  </a>
  <br>
  <a href="https://youtu.be/rv6dafFKCII">在 YouTube 上观看</a>
</p>

## 范围与局限

- **不是抓取规划器。** 维持小的指腹间隙与坐标系对齐，本身并不构成力旋量闭合或稳定的物体操作。
- **仅几何接触。** 当前闭环中没有触觉反馈或接触力估计。
- **位控型手型。** 执行后端假设关节可独立指令；面向耦合/欠驱动传动的执行器空间路径正在开发中。
- **仅仿真验证。** 真机实验与用户研究已规划，尚未完成。
- **无需训练数据。** 运行时完全依赖形态学元数据与几何约束。

## 路线图

- [x] 形态学导入 + 自动接触坐标系提取
- [x] 从手型拓扑枚举接触模板
- [x] 贝叶斯意图滤波 + 正运动学转移代价
- [x] 闭链接触后投影 + 表面 UV 联合优化
- [x] 五手型 Quest 3 回放基线
- [ ] 面向耦合/欠驱动手型的执行器空间间接求解路径
- [ ] 真机闭环验证
- [ ] 用户研究（NASA-TLX、任务完成时间）
- [ ] 触觉接触置信度与更丰富的接触模型

## 代码

源代码与实验流水线位于私有仓库中，将随论文一并开源。本页面用于公开跟踪项目状态。

## 引用

论文发表后将提供 BibTeX 条目。在此之前如需引用本项工作进行中的版本，请引用本页面或发起一个 Discussion。

## 许可

文档与素材：[MIT License](LICENSE)。源代码的开源条款将随发布一同公布。
