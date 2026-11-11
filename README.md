# 具身智能作品集 · Embodied AI Portfolio

机器人仿真模仿学习的可复现实验集，覆盖 **VLA（视觉-语言-动作）推方块** 与 **行为克隆抓取-放置（pick-and-place）** 两条主线，每条主线都在 **PyBullet 与 MuJoCo** 两个物理引擎上各实现一遍。所有项目均为自包含、CPU 可跑、脚本端到端可复现，不含闭源 API 包装。

## 项目一览

| 子目录 | 任务 | 仿真器 | 机械臂 | 方法 | 闭环结果 |
|---|---|---|---|---|---|
| [`vla-lang-push-pybullet/`](vla-lang-push-pybullet) | 语言指令推方块到四区域 | PyBullet + panda-gym | PandaPush-v3（7-DoF） | VLA（CNN+Transformer），BC → REINFORCE 微调 | 见子目录 README |
| [`vla-lang-push-mujoco/`](vla-lang-push-mujoco) | 同上，MuJoCo 跨平台复刻 | MuJoCo + menagerie | Franka Emika Panda（7-DoF，OS-DLS 伺服） | 同一 VLA 模型与训练管线 | 见子目录 README |
| [`pickplace-pybullet/`](pickplace-pybullet) | 抓取-放置（标准 benchmark） | PyBullet + panda-gym | PandaPickAndPlace-v3（7-DoF + IK） | goal-conditioned MLP 行为克隆 | BC 35%，oracle 92% |
| [`pickplace-mujoco/`](pickplace-mujoco) | 抓取-放置（自建场景） | MuJoCo | 3 轴 Cartesian slide + 平行双指 | goal-conditioned MLP 行为克隆 | BC 95%，oracle 100% |

## 两条主线

### 1. VLA 语言推方块（`vla-lang-push-*`）

回答一个问题：**一个端到端的小型 VLA——输入相机图像 + 自然语言指令、输出电机指令——能否只用几千条演示训练出来，并泛化到训练中没见过的指令措辞？**

- 观测：64×64 俯视 RGB 图像 + 英文指令 token（23 词词表）
- 策略：CNN（视觉）+ Transformer（语言）→ MLP → 2/3 维末端速度
- 训练：脚本 oracle 采集成功演示 → 行为克隆（MSE）→ 闭环 REINFORCE 微调
- 泛化评测：留出指令措辞（同一四区域，不同句子模板），而非留出区域语义
- MuJoCo 版是 PyBullet 版的跨平台移植：语义层逐字对齐，物理层换成 MuJoCo 软接触与 OS-DLS 逆运动学闭环，并记录了从 0–1% 调到 69% oracle 的真实踩坑过程

### 2. 抓取-放置 BC（`pickplace-*`）

在标准/自建环境上跑通 **脚本演示采集 → BC 训练 → 闭环评估 → 视频** 全链路：

- 观测：EE 位姿/速度 + 物体位姿 + 目标位置（MuJoCo 版补了夹爪开度与任务进度）
- 策略：MLP（128-128），观测标准化，MSE 回归 4 维动作（xyz 位移 + 夹爪开合）
- 对照结论：PyBullet 真实 7-DoF + IK 环境更"标准"但接触阶段单步 BC 仅 35%；MuJoCo 简化 Cartesian 夹爪环境 BC 达 95%。**环境简化能抬高 BC 数字，但泛化回 7-DoF + 视觉才是真实方向。**

## 复现环境

本机存在两个包不互通的 Python 解释器，按项目选用：

- **PyBullet / panda-gym 项目**（`vla-lang-push-pybullet`、`pickplace-pybullet`）：Python 3.11，依赖 `pybullet`、`panda-gym`、`gymnasium`、`torch`
- **MuJoCo 项目**（`vla-lang-push-mujoco`、`pickplace-mujoco`）：Python 3.13，依赖 `mujoco`、`torch`、`opencv-python`

各子目录 `requirements.txt` 列出具体依赖。数据（npz）、权重（pt）、视频（mp4）不入库，按各目录 README 命令可重新生成。

## 目录结构

```
embodied-portfolio/
├── vla-lang-push-pybullet/    # VLA 推方块（PyBullet）
├── vla-lang-push-mujoco/      # VLA 推方块（MuJoCo 复刻）
├── pickplace-pybullet/        # 抓取-放置 BC（panda-gym 标准环境）
└── pickplace-mujoco/          # 抓取-放置 BC（MuJoCo 自建夹爪）
```

## 方法栈与局限

- **已覆盖**：行为克隆、语言条件策略、视觉观测、闭环 RL 微调（REINFORCE）、双仿真器对照、脚本 oracle 数据采集。
- **已知局限**：单步 BC 存在分布漂移，长程接触任务成功率受限；MuJoCo 抓取版使用任务进度（时间索引）作为输入，泛化到不同时长会退化；VLA 为小规模 Transformer，未接入预训练视觉/语言 backbone。
- **后续方向**：动作 chunk 与观测历史堆叠、Diffusion Policy 动作头、HER、视觉策略（替换状态输入）。
