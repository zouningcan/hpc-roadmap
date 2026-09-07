# HPC 超算学习路线仓库

> 目标：从零基础到能打超算比赛（ASC / SC / ISC 校内选拔）
> 建立时间：2026-09-04 ｜ 维护方式：每个模块 = 官方教材搜索 → 浓缩讲义 → 教育性练习

## 这是什么

一个自包含的学习仓库：四个阶段，每个模块（一讲/一章）一个目录，内含两样东西——

- `notes.md`：基于**官方讲义/教材**（联网搜索后浓缩）的讲义，注明来源链接
- `exercises.md`：分层练习（概念题 → 动手题 → 挑战题），每题标注**考察点**与**常见误区**

配套机制：`drills/`（学长突袭题库，跨模块随机复习）、`progress/progress.md`（进度与错题本）。

## 阶段导览

> 表中"生产"= 讲义+练习+答案三件套是否齐备；个人学习进度见 [progress/progress.md](progress/progress.md)。

| 阶段 | 内容 | 模块数 | 生产状态 |
|------|------|--------|----------|
| [stage1-tools](stage1-tools/) | MIT《The Missing Semester》 | 11 讲 | ✅ **完成（11/11）** 2026-09-05 |
| [stage2-cpp](stage2-cpp/) | Stanford CS106B（C++ 编程抽象） | 14 模块（按官方28讲聚类） | ✅ **完成（14/14）** 2026-09-05 |
| [stage3-dsa](stage3-dsa/) | 清华邓俊辉《数据结构》 | 11 模块 | ✅ **完成（11/11）** 2026-09-06 |
| [stage4-parallel](stage4-parallel/) | Slurm/perf/OpenMP/MPI/CUDA/CSE234 | 8 模块 | ✅ **完成（8/8）** 2026-09-06 |

> **🏁 仓库生产全部完工（45/45 模块，2026-09-06）**：每模块三件套（讲义+分层练习+答案）。学习进度见 [progress/progress.md](progress/progress.md)。

另有：[competitions/](competitions/)（三大竞赛赛制 + 评分规则 + HelloHPC 真题分析）、[docs/00-路线总览.md](docs/00-路线总览.md)（原始路线图展开）、[docs/01-课程溯源.md](docs/01-课程溯源.md)（**每个模块的官方教材源头图谱**，进什么料一目了然）。

## 怎么用

0. **选课看 [学习菜单.md](学习菜单.md)**——全部目录×学习状态一页扫（结课后由学长维护）
1. 按阶段顺序学；每模块先读 `notes.md`，再做 `exercises.md`
2. 做错的题记进 `progress/progress.md` 的错题本，`drills/` 会从中抽题
3. WSL 侧路径：`/mnt/d/znc/智谱/hpc-roadmap`（远端：https://github.com/zouningcan/hpc-roadmap）

## 行尾约定

本仓库所有文本文件 **LF** 行尾（`.gitattributes` 强制）。Windows 记事本编辑会引入 CRLF，请用 VSCode（右下角确认 LF）。
