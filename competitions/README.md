# 比赛资料

> **真题库**：[真题库.md](真题库.md) —— HelloHPC 之外的公开题源（交大 XFLOPS 招新考核、北大 HPCGame、ASC/SC/ISC 历届赛题）+ 模块映射 + 刷题建议

## 三大国际大学生超算竞赛

| 竞赛 | 主办 | 赛制要点 |
|------|------|----------|
| ASC | 中国发起 | 预赛交方案 → 决赛 3000W 功耗墙现场搭集群；赛题近年偏 AI/LLM/AI for Science |
| SC SCC | 美国 | 6 人队，48 小时连续挑战，功耗封顶 |
| ISC SCC | 德国 | 线上+线下，26 队规模 |

共同内容：HPL/HPCG 基准优化 + 应用赛题（MPI/OpenMP/CUDA 优化）+ 集群搭建（Linux/Slurm）+ 答辩。

## HelloHPC 真题分析（交大第一届，2025.09）

完整题目存于：`C:\Users\znc20\.zcode\workspace\default\hello-hpc\`（zip 解包）。

10 题速览（按核心数 1→128 递增）：

| # | 题 | 核 | 考什么 |
|---|---|---|---|
| 1 | HelloHPC 签到 | – | ssh + ED25519 指纹 |
| 2 | 小交问答 | – | 信息素养：Slurm/网络/NUMA/module |
| 3 | 位矩阵密度 | 1 | 位运算 popcount + ARM NEON 向量化 |
| 4 | 海洋模拟 | 4 | perf 找热点 + OpenMP |
| 5 | Calculation | 8 | 算法强度削减 + 编译器选项（毕昇 clang++） |
| 6 | 魔咒筛选器 | 8 | 数据流水线，Python→C++ 重写 |
| 7 | Amazing LLM | 32 | 编译 vLLM + 6bit 解量化 C++ 自定义算子 |
| 8 | Graph500 | 128 | 不改码，MPI 运行时调优冲 TEPS |
| 9 | 绳网模拟 | 128 | 物理仿真并行化+向量化，热点自找 |
| 10 | WRF | 128 | 从源码编译 WRF+4 依赖（ARM 鲲鹏），踩文档坑 |

关键情报：
- 评分全部是"速度对数插值 + 正确性门槛"——先对再快
- 不能在登录节点跑重负载（封号），长任务用 sbatch 交 kp_run 队列
- 行尾必须 LF（Windows 记事本会引入 CRLF 导致脚本失效）
- 平台是 ARM 鲲鹏：向量化是 NEON 不是 x86 AVX
- 必交 writeup.md，赛后人工 Code Review

## 评分规则总结（2026-09-05 网上查证 + 真题提取）

### 一、OJ 式校内/区域赛（HelloHPC 模式，也是复旦选拔的参照系）

每题独立公式化评分，两种模式：

**模式 A：运行时间型**（WRF 题）
- 正确性门槛分 + 时间插值分各占一半
- `score = 50 × (ln(t_zero) − ln(t_your)) / (ln(t_zero) − ln(t_full))`
- 对数插值 → **离满分远时每分钟优化都值钱，逼近上限收益递减**
- WRF 实例：编译+正确 = 50 分；运行 15min = 0 分、10.5min = 50 分

**模式 B：吞吐率型**（Graph500 题）
- `score = clip(((实际 − 基础) / (满分 − 基础))², 0, 1) × 权重`（bfs 60% + sssp 40% 调和平均）
- 平方插值 → **起步阶段不值钱，越接近满分每一分 TEPS 越贵**（与模式 A 相反）

**共同纪律**：先对再快（正确性门槛 + 严禁重定向/篡改输出）；编译题必须真从源码编译（用集群自带版本 = 作弊）；超时强杀；writeup 人工复审。

**算例（WRF，t_zero=15min，t_full=10.5min）**：
| 运行时间 | 性能分 | 总分（+50 门槛） | 备注 |
|---------|--------|------------------|------|
| 14:00（裸跑） | 9.7 | 59.7 | 编译跑通≈拿 60% |
| 13:00 | 20.1 | 70.1 | 提速 7% ≈ +10 分 |
| 11:30 | 37.3 | 87.3 | |
| 10:30 | 50.0 | 100 | 封顶 |

对数插值的正确读法：**等比例提速换等分**——任何位置再快 15% 都值约 20 分（不触顶时）；绝对省时相同时，越接近满分每分钟越值钱。
Graph500 平方插值对照：走到满分路段 5%→10% 只多 0.75 分；90%→95% 多 9 分——**末段溢价，要么不做要么做到底**。

### 二、三大国际赛（查证：2026-09）

| 赛 | 赛制 | 功耗墙（核心变量） | 评分构成 |
|----|------|------|------|
| ASC | 初赛线上（赛题作业+报告）→ 决赛现场 ~5 天自建集群 | 3000W 传统；ASC25 4000W；ASC26 5000W+单机 2000W | HPL+HPCG 基准 + 应用题（近年 LLM/AI4S：引力波、AlphaFold 等）+ 答辩 |
| SC SCC | 6 人队，赞助硬件，赛前基准上分 + 现场 48h 连轴 | 固定上限（SC25 = 4500W），全程监控 | HPL/HPCG/IO500 基准 + 真实应用 + 复现类任务（reproducibility）+ 突发神秘题 + 面试，综合积分 |
| ISC SCC | 欧洲场，线上+线下 | 功耗封顶 | 微基准 + HPC 应用 + 现场任务 |

**共同哲学**：①**每瓦性能 > 峰值性能**——功耗墙是硬约束，超了直接判负，降频省电反而总分高；②基准分只是门槛，应用优化和现场应变才是拉分项；③先正确后速度一以贯之。

来源：[SC25 SCC 官方](https://sc25.supercomputing.org/students/student-cluster-competition/)、[HPCwire SC25](https://www.hpcwire.com/off-the-wire/scc25-students-power-up-for-the-ultimate-hpc-challenge-at-sc25/)、[MGHPCC 2022 报告](https://mghpcc.org/2022-sc-student-cluster-competition/)、[ISC SCC](https://isc-hpc.com/program/student-cluster-competition/)、[UESTC ASC26 报道](https://info.uestc.edu.cn/info/1014/4384.htm)、[百度百科第十届 ASC](https://baike.baidu.com/item/%E7%AC%AC%E5%8D%81%E5%B1%8AASC%E4%B8%96%E7%95%8C%E5%A4%A7%E5%AD%A6%E7%94%9F%E8%B6%85%E7%BA%A7%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%AB%9E%E8%B5%9B/62915942)
