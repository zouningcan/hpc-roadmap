# CSE234 / Deep Learning Systems 上课资料包

> 背景：学习者正在修读 Hao Zhang 的 MLSys 课（Spring 2026 为 [DSC 291/CSE 291: Deep Learning Systems](https://haoailab.com/cse291-s26/)，其前身即 Winter 2025 [CSE234](https://haoailab.com/cse234-w25/)——W25 版有全公开 slides+录像，是最完整的自学底本）。
> 本文档 = 官方资源导航 + 20 讲必读论文对照 + 本仓库配套模块映射。查证 2026-09-06。

## 1. 官方资源

| 资源 | 链接 | 说明 |
|------|------|------|
| CSE234 W25 课程主页 | [haoailab.com/cse234-w25](https://haoailab.com/cse234-w25/) | 全部 slides PDF 公开 |
| 同上 · 大纲 | [syllabus](https://haoailab.com/cse234-w25/syllabus/) | 计分：Reading Summary 10% + Scribe 8% 等——**阅读论文是这门课的正式作业** |
| 同上 · W25 官方录像 | [podcast.ucsd.edu](https://podcast.ucsd.edu/watch/wi25/cse234_a00/1) | 20 讲全 |
| 同上 · B 站中英字幕 | [BV1YSw4zDEF5](https://www.bilibili.com/video/BV1YSw4zDEF5/) | 中文学习者的直达通道 |
| Sp26 新课（Deep Learning Systems） | [haoailab.com/cse291-s26](https://haoailab.com/cse291-s26/) | 主题同源升级；slides/录像部分公开 |
| 前身（DSC291 S24） | [dsc291-s24](https://github.com/hao-ai-lab/dsc291-s24) | 往期大纲/syllabus |
| **官方教材《MLSys book》** | [mlsysbook.ai](https://mlsysbook.ai/) | 每周必读的主教材（readings 编号 1.1/2.1… 即其章节） |
| 教师教学页 | [haozhang.ai/teaching](https://haozhang.ai/teaching/) | 全部学期版本入口 |
| 自学社区收录 | [PKUFlyingPig/cs-self-learning](https://github.com/PKUFlyingPig/cs-self-learning) | 中文自学路线含此课 |

## 2. W25 二十讲 × 必读论文对照（黄金表）

> readings 编号 = MLSys book 章节 / arXiv 论文；★ = 强烈建议精读（对应本仓库模块 07 重点）

| 讲次 | 主题 | 必读 Readings |
|------|------|---------------|
| L1-2 | ML 系统总览 / 现代深度学习 | MLSys 书 1.1（Intro）、1.2（DNN） |
| L3-4 | **autodiff / 计算图**、tensor 格式与 matmul | TF 论文（2.1）、**PyTorch 论文（2.2）** |
| L5-6 | GPU/CUDA、GPU matmul | GPU Performance（3.1）、MI300X vs H100（3.2） |
| L7-8 | **Triton/算子编译**、内存 | TVM（4.1）、**Triton 论文（4.2）** |
| L9 | 量化 | Deep Compression（5.1）、Quantization Survey（5.2） |
| L10 | 客座：陈天奇（TVM 作者） | — |
| L11-12 | **并行化**、集合通信 | **ML Parallelism Blog（6.1）、Megatron（6.2）** |
| L13 | 并行化：数据/算子间/算子内 | **GPipe（7.1）、Alpa（7.2）**；选读 Megatron v3、PipeDream、GShard |
| L15-17 | Scaling law、Transformer/attention | GPT-3（8.1）、Chinchilla Scaling Law（8.2） |
| L17-19 | **LLM serving** | ★ **PagedAttention/vLLM（9.1）**、★ **FlashAttention（9.2）**；选读 Orca、Speculative Decoding、DistServe、EAGLE |
| L20 | FlashAttention、DeepSeek-V3 复盘 | 选读 DeepSeek-V3 |

> PA 节奏（W25）：PA2 于 2/11 发布、3/4 截止——**GPU/算子优化类作业**，尽早开工。

## 3. 配套工具与练习（听课之外的双手）

- **Triton**：[官方 tutorials](https://triton-lang.org/main/getting-started/tutorials.html)（vector-add → fused softmax → matmul 三部曲，正是 L7-8 的作业形态）
- **GPU Puzzles**（Sp26 readings 官方推荐）：[srush/GPU-Puzzles](https://github.com/srush/GPU-Puzzles)——用填空题学 CUDA，无 GPU 可做
- **GPU Glossary**（Sp26 readings 官方推荐）：[modal.com/gpu-glossary](https://modal.com/gpu-glossary/)——术语速查
- **vLLM**：[docs.vllm.ai](https://docs.vllm.ai)（L17-19 的活教材，HelloHPC 第 7 题同款）
- **FlashAttention**：[arXiv 2205.14135](https://arxiv.org/abs/2205.14135)；**vLLM/PagedAttention**：[arXiv 2309.06180](https://arxiv.org/abs/2309.06180)

## 4. 本仓库配套模块映射（听课 ↔ 仓库复习）

| CSE234 讲次 | 对应本仓库模块/小节 | 用法 |
|-------------|---------------------|------|
| L3 autodiff/计算图 | [07 notes §2](notes.md) | A1 练习手推 (a+b)*b 图 ↔ 听课前预习 |
| L4-6 matmul/GPU | 07 §3-4、[05-cuda](../05-cuda/) | 07 建观、05 动手（B1-B4 纸上执行） |
| L7-8 Triton/内存 | 07 §5 | Triton tutorials 三部曲当 PA 热身 |
| L9 量化 | 07 §7、exercises B2 | 解量化体感实验（HelloHPC 第 7 题内核） |
| L11-15 并行化 | [03-openmp](../03-openmp/) + [04-mpi](../04-mpi/) + 07 §6 | **先修 03/04 再听，事半功倍**——DP/TP/PP 的通信原语就是 Allreduce 家族 |
| L16-20 LLM serving | 07 §7 | KV cache/PagedAttention/continuous batching——HelloHPC 第 7 题主场 |

> 学习顺序建议：**仓库 03/04（并行地基）→ 07 模块（系统观预习）→ 跟课（重点听 L5-8 GPU 编译 + L11-19 并行与 serving）→ PA 实战**。课堂没听懂的回到对应模块小节补。
