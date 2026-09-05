# 阶段 4：并行编程与 HPC 实战

> 进入比赛硬技能区。参考：HPC-roadmap（华科）、ZJUSCT 指南、PMPP 教材、UCSD CSE234

## 参考资源（2026-09-05 查证）

- **总路线（同类社团视角）**：[HPC-roadmap 华科](https://github.com/heptagonhust/HPC-roadmap)（[在线版](https://heptagonhust.github.io/HPC-roadmap/)，"HPC=高性能算法+软件+硬件"）；[ZJUSCT HPC101 浙大超算队](https://github.com/ZJUSCT/HPC101)；[l0ngc/hpc-learning](https://github.com/l0ngc/hpc-learning)
- **02 profiling**：[perf 官方 tutorial](https://perf.wiki.kernel.org/index.php/Tutorial)；[Brendan Gregg perf 示例集](https://www.brendangregg.com/perf.html)；[NVIDIA Nsight Systems](https://developer.nvidia.com/nsight-systems)
- **03 OpenMP**：[LLNL OpenMP 教程](https://hpc-tutorials.llnl.gov/openmp/)（渐进式经典）；[openmp.org 官方资源](https://www.openmp.org/resources/)
- **04 MPI**：[LLNL MPI 教程](https://hpc-tutorials.llnl.gov/mpi/)；[mpitutorial.com](https://mpitutorial.com/tutorials/)（有中文版入口）
- **05 CUDA / PMPP**：教材《Programming Massively Parallel Processors》4th ed（Hwu/Kirk/El Hajj）；[NVIDIA DLI 免费课](https://www.nvidia.com/en-us/training/)
- **经典公开课（整体并行观）**：CMU 15-418（cs.cmu.edu/~418，YouTube 全套录像）；Berkeley CS267 Applications of Parallel Computers（sites.google.com/lbl.gov 搜最新学期，YouTube 全套）
- **07 CSE234**：[UCSD CSE234 W25 主页](https://haoailab.com/cse234-w25/)（Hao Zhang，Data Systems for ML）；[教学大纲](https://haoailab.com/cse234-w25/syllabus/)；[UCSD 公开录像](https://podcast.ucsd.edu/watch/wi25/cse234_a00/1)；[往期 wi23](https://cseweb.ucsd.edu/classes/wi23/cse234-a/)
- **08 基准**：[HPL](https://www.netlib.org/benchmark/hpl/)；[HPCG](https://www.hpcg-benchmark.org/)

| # | 模块 | 目录 | 讲义 | 练习 |
|---|------|------|------|------|
| 1 | 集群与 Slurm 调度 | 01-slurm-cluster | ✅ | ✅ |
| 2 | 性能剖析：perf / Nsight / VTune | 02-profiling | ⬜ | ⬜ |
| 3 | OpenMP 共享内存并行 | 03-openmp | ⬜ | ⬜ |
| 4 | MPI 消息传递并行 | 04-mpi | ⬜ | ⬜ |
| 5 | CUDA GPU 编程基础 | 05-cuda | ⬜ | ⬜ |
| 6 | 进阶优化：SIMD/NEON、访存、编译器 | 06-advanced-opt | ⬜ | ⬜ |
| 7 | GPU 与 ML 系统（CSE234 精要） | 07-gpu-ml-systems | ⬜ | ⬜ |
| 8 | HPL/HPCG 基准实战 | 08-benchmarks | ⬜ | ⬜ |
