# 阶段 4：并行编程与 HPC 实战

> 进入比赛硬技能区。参考：HPC-roadmap（华科）、ZJUSCT 指南、PMPP 教材、UCSD CSE234

## 🎯 跟学营：WMHPC Training Camp × LCPU AI Infra Seminars（2026 暑期，正在跟）

北大未名超算队 × 北大 Linux 俱乐部（LCPU）联办、**vLLM 官方合作**的 AI Infra 系列（官网 [infra.seminars.lcpu.dev](https://infra.seminars.lcpu.dev)）。主题四块与本阶段目录一一对应：

| 营的主题 | 内容关键词 | 对应本仓库 |
|----------|-----------|------------|
| Kernel & ML Compiler | CUDA、Triton、TileLang、Tensor Core、Warp Specialization、软件流水 | 05-cuda、06-advanced-opt |
| Interconnect & Communication | RDMA、NCCL、Scale-Up/Out、网络 Fabric | 04-mpi（集合通信）、08-benchmarks |
| LLM Serving & Inference | KV Cache Centric、Batching、PD 分离、Spec Decode、vLLM | 07-gpu-ml-systems §7 |
| Distributed RL Systems | PPO、veRL、Rollout、Agentic RL、训推一致性 | （扩展区，07 模块延伸） |

- 首讲录播：初识 CUDA 编程模型（郑熠，SIMD/SIMT → CUDA 编程模型 → Triton/TileLang）
- 作业仓库：[lcpu-club/wmhpc-training-camp-x-lcpu-ai-infra-seminars](https://github.com/lcpu-club/wmhpc-training-camp-x-lcpu-ai-infra-seminars)（`assignment01` = CUDA/GPU 编程练习，232★）
- ⚠️ 仓库有 AI 使用政策（CLAUDE.md）：**"AI 可以帮你理解，但不能替你实现"**——作业必须自己写，AI 只当答疑器
- 跟学方法：营的每讲先过本仓库对应模块 notes 做预习，作业按"先自己写 → 卡住再对照讲义 → 最后才问 AI"三段式

## 参考资源（2026-09-05 查证）

- **总路线（同类社团视角）**：[HPC-roadmap 华科](https://github.com/heptagonhust/HPC-roadmap)（[在线版](https://heptagonhust.github.io/HPC-roadmap/)，"HPC=高性能算法+软件+硬件"）；[ZJUSCT HPC101 浙大超算队](https://github.com/ZJUSCT/HPC101)；[l0ngc/hpc-learning](https://github.com/l0ngc/hpc-learning)
- **02 profiling**：[perf 官方 tutorial](https://perf.wiki.kernel.org/index.php/Tutorial)；[Brendan Gregg perf 示例集](https://www.brendangregg.com/perf.html)；[NVIDIA Nsight Systems](https://developer.nvidia.com/nsight-systems)
- **03 OpenMP**：[LLNL OpenMP 教程](https://hpc-tutorials.llnl.gov/openmp/)（渐进式经典）；[openmp.org 官方资源](https://www.openmp.org/resources/)
- **04 MPI**：[LLNL MPI 教程](https://hpc-tutorials.llnl.gov/mpi/)；[mpitutorial.com](https://mpitutorial.com/tutorials/)（有中文版入口）
- **05 CUDA / PMPP**：教材《Programming Massively Parallel Processors》4th ed（Hwu/Kirk/El Hajj）；[NVIDIA DLI 免费课](https://www.nvidia.com/en-us/training/)
- **经典公开课（整体并行观）**：CMU 15-418（cs.cmu.edu/~418，YouTube 全套录像）；Berkeley CS267 Applications of Parallel Computers（sites.google.com/lbl.gov 搜最新学期，YouTube 全套）
- **07 CSE234**：[UCSD CSE234 W25 主页](https://haoailab.com/cse234-w25/)（Hao Zhang，Data Systems for ML；**前身为 DSC 291 S24，故常写作 234/291**）；[教学大纲](https://haoailab.com/cse234-w25/syllabus/)；[UCSD 公开录像](https://podcast.ucsd.edu/watch/wi25/cse234_a00/1)；[B 站中英字幕全集](https://www.bilibili.com/video/BV1YSw4zDEF5/)；[课程网站源仓库](https://github.com/hao-ai-lab/cse234-w25)（fork 自 dsc291-s24）；[往期 wi23](https://cseweb.ucsd.edu/classes/wi23/cse234-a/)
- **课号勘误（2026-09-06 查证）**：CSE 234 **不教 MPI/OpenMP**——它的"并行"是 ML 并行策略（数据/张量/流水线并行，底层 NCCL）。UCSD 真正的传统并行编程课是 CSE 160（本科）/ CSE 260（研究生，覆盖 MPI/OpenMP）；SDSC 超算中心另有 [MPI/OpenMP 实训](https://hpc-training.sdsc.edu/hpc-training-docs/sdsc-summer-institute-2023/6.1a_parallel_computing_mpi_openmp/)。本仓库 MPI/OpenMP 学习仍以 LLNL 两门教程为主
- **08 基准**：[HPL](https://www.netlib.org/benchmark/hpl/)；[HPCG](https://www.hpcg-benchmark.org/)

| # | 模块 | 目录 | 讲义 | 练习 |
|---|------|------|------|------|
| 1 | 集群与 Slurm 调度 | [01-slurm-cluster](01-slurm-cluster/) | ✅ | ✅ |
| 2 | 性能剖析：perf / Nsight / VTune | [02-profiling](02-profiling/) | ✅ | ✅ |
| 3 | OpenMP 共享内存并行 | [03-openmp](03-openmp/) | ✅ | ✅ |
| 4 | MPI 消息传递并行 | [04-mpi](04-mpi/) | ✅ | ✅ |
| 5 | CUDA GPU 编程基础 | [05-cuda](05-cuda/) | ✅ | ✅ |
| 6 | 进阶优化：SIMD/NEON、访存、编译器 | [06-advanced-opt](06-advanced-opt/) | ✅ | ✅ |
| 7 | GPU 与 ML 系统（CSE234 精要） | [07-gpu-ml-systems](07-gpu-ml-systems/) | ✅ | ✅（另附[上课资料包](07-gpu-ml-systems/resources.md)：20 讲必读论文对照+官方资源） |
| 8 | HPL/HPCG 基准实战 | [08-benchmarks](08-benchmarks/) | ✅ | ✅ |

> **🎉 stage4 生产完成 2026-09-06**：8/8 模块三件套齐备。结业标准 = 08-benchmarks/exercises.md 全卷通过。
