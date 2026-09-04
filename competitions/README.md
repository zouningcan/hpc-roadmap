# 比赛资料

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
