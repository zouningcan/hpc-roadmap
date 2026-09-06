# 模块 07：GPU 与 ML 系统（CSE234 精要）

> **来源**：[UCSD CSE234 W25](https://haoailab.com/cse234-w25/)（Hao Zhang，Data Systems for ML；前身 DSC 291 S24）；[大纲](https://haoailab.com/cse234-w25/syllabus/)；[UCSD 官方录像](https://podcast.ucsd.edu/watch/wi25/cse234_a00/4)；[B 站中英字幕全集](https://www.bilibili.com/video/BV1YSw4zDEF5/)
> 课程定位：**ML 系统课**——不是 MPI 课（海报 "HPC/Infra 进阶" 之称的出处，但内容是 GPU 计算 + ML 系统）。它的"并行"是模型并行策略（数据/张量/流水线），底层通信走 NCCL
> 本模块是 **15 讲的精要导览**：深水区（CUDA 编程细节）在模块 05，向量汇编优化在 06

## 0. 一句话定位

CSE234 回答一个问题：**一个神经网络从 Python 代码到 GPU 上跑得飞快，中间整条系统栈（框架→编译→算子→通信→服务）每层干了什么？** HelloHPC 第 7 题（vLLM + 6bit 解量化优化）就是这个栈的中段实战。

## 1. 全景图：ML 系统栈（对应课程的三大段）

```
┌─ L4 服务层：vLLM 推理系统（KV cache / PagedAttention / continuous batching）
├─ L3 并行层：数据并行 / 张量并行 / 流水线并行（Megatron / GPipe / DDP）→ NCCL 集合通信
├─ L2 编译层：算子编译（Triton / TVM）、图优化、内存规划
├─ L1 算子层：matmul(GEMM)、attention、conv —— 手写 CUDA / 库(cuBLAS/cuDNN)
└─ L0 硬件层：GPU 架构（SM/warp/shared memory/HBM）—— PMPP 教材领域（模块 05）
```

课程第 1-4 讲打地基（框架/计算图/autodiff/tensor），5-8 讲走 L0-L2（GPU/CUDA/编译），9-15 讲登 L3-L4（并行/LLM 系统）。

## 2. 地基概念（第 1-3 讲精要）

- **计算图（computational graph）**：神经网络 = 一张"数据流图"，节点是算子（matmul/加/激活），边是 tensor。框架（PyTorch）做的事：按拓扑序调度节点到 GPU 上执行
- **autodiff（自动微分）**：前向算一遍记录局部导数，反向沿图**链式法则**自动乘出来——深度学习能训起来的根基。系统视角：反向传播就是"逆拓扑序再调度一遍图"
- **Tensor 格式**：shape + stride（步长）。`transpose` 不搬数据只改 stride——所以转置后 matmul 慢（访存模式变差），这是后面算子优化的常客

## 3. 世界的中心：GEMM（matmul）（第 4 讲）

**为什么一切皆 GEMM**：全连接层是 GEMM，卷积展开（im2col）后是 GEMM，attention 的 QKᵀ 也是 batched GEMM——GPU 上 90% 的 FLOPs 都落在矩阵乘。**优化 GEMM ≈ 优化整个训练**。

GEMM 的性能本质（接 03 模块的 roofline 思维）：

- 计算量 O(n³)、数据量 O(n²)——算术强度 O(n)，**规模越大越靠近计算上限**（对比 sum_array 的 0.25 flop/B，GEMM 大矩阵能到数百 flop/B）
- 优化三板斧：**tiling（分块进 shared memory 复用）→ 寄存器微_tile → 向量化访存**——从朴素 3 重循环到 cuBLAS 级别，差距可达 100 倍，全在这三板斧的排列组合
- **_tensor core**：专门算"小矩阵片乘加"的硬件单元（混合精度 FP16/BF16 输入 FP32 累加），现代 GPU 的算力主要来自它——所以比赛跑分要开混合精度

## 4. GPU 架构与 CUDA（第 5-6 讲，深水区在模块 05）

三层并行映射（必须记住的对应关系）：

```
CUDA 层次        硬件实体           数量级
grid（网格）   → 整块 GPU          1
block（块）    → 一个 SM           几百~几千
thread（线程） → SM 里的调度单位    几万~百万
  └─ 32 线程 = 1 warp → SIMD 锁步执行（和 CPU 向量化同宗！）
```

关键资源约束：**shared memory（每 block 的私有快存，~100KB）和寄存器有限**——占太多反而降占用率（occupancy）。GPU 性能艺术 = 在"并行度拉满"与"访存局部性用好"之间走钢丝。

## 5. 算子编译（第 7-8 讲）

- **手写 CUDA 的痛点**：每个 GPU 型号都要调一遍，专家稀缺 → **算子编译器**（Triton/TVM）让编译器帮你生成
- **Triton**：Python 写"块级"运算（一整块 tile 的运算，不管线程细节），编译器负责切线程+访存+流水——比手写 CUDA 抽象高一层的甜点位
- **图优化**：框架把整张计算图做算子融合（fusion，如 conv+bias+relu 合成一个核，少一次显存读写）、内存复用规划——**少访存是永恒主题**（每次显存往返都是 ~µs 级）

## 6. 并行策略（第 11-14 讲）——L3 层

训练大模型，一个 GPU 装不下算不动，切法三种：

| 策略 | 切什么 | 通信 | 代表系统 |
|------|--------|------|----------|
| **数据并行（DP）** | 切数据（batch），模型人人一份 | 反向结束时 **Allreduce 梯度** | PyTorch DDP |
| **张量并行（TP）** | 切单层的大矩阵（列切/行切） | 前后向各插 2 次集合通信，**最频繁** | Megatron-LM |
| **流水线并行（PP）** | 按层切，前段输出给后段 | 点对点传激活，**微批流水**填泡泡 | GPipe/Megatron |

- **3D 并行 = DP×TP×PP 组合**（千亿模型的标配配置学）
- 底层通信：就是 04 模块的 Allreduce/Broadcast 家族，由 **NCCL**（NVIDIA 集合通信库）在 GPU 拓扑上精调实现——**MPI 思想的 GPU 转世**
- 04 的 ring allreduce 算法在这里直接复用：数据并行梯度同步的流量 = 模型大小 × 每步，N 亿参数模型每步就是 GB 级通信——**通信瓶颈决定扩展效率**

## 7. LLM 推理系统（第 9-10、15 讲）——vLLM 的内脏（HelloHPC 第 7 题主场）

推理为什么是另一个问题：训练是 GEMM 大并行，推理是**逐 token 生成**——一步只算一个词，显存带宽喂权重才是瓶颈。

- **KV cache**：自回归生成时把每层的 Key/Value 缓存下来，避免每个新 token 重算全部历史——**空间换时间的极致**；代价是显存被吃（长上下文 = cache 爆炸）
- **PagedAttention（vLLM 核心）**：借鉴操作系统**分页**思想，把 KV cache 切成固定大小的 block 按需分配——解决"预分配连续显存 → 碎片浪费 60-80%"的问题，吞吐翻倍级提升（**OS 思想反哺 ML 系统，CSE234 的招牌案例**）
- **continuous batching**：动态批——生成完的请求立刻退出、新请求立刻插入，GPU 永不空转（对比静态批：等最慢的）
- **FlashAttention**：重写 attention 计算，分块进 SRAM、永不落显存——数学结果不变，显存读写从 O(n²) 降到 O(n)，又是一次"访存决定生死"
- **量化（quantization）**：FP16 → INT8/INT4/6bit 压权重，省显存+提带宽利用率；**解量化（dequant）**是推理加载时的必经之路——HelloHPC 第 7 题让你优化的 `recover_from_quant` 函数正是它（baseline 是 PyTorch 逐元素算子，慢；正确姿势是写 C++/CUDA 自定义算子批量位操作——压缩的位.unpack 成浮点）

## 8. 常见误区清单

1. "CSE234 是 HPC 课"：它是 ML 系统课——MPI/OpenMP 不在课内（本仓库 03/04 补），但 Allreduce 概念同源
2. "推理和训练是一回事"：训练 compute-bound、推理 bandwidth-bound——优化方向完全不同
3. "KV cache 是优化项可关掉"：没有它每生成一个 token 都重算全部历史，复杂度从 O(n) 退化回 O(n²)——它是架构的一部分
4. "量化就是把位数截断"：6bit 量化有专门的分组/缩放/编码方案，解量化要按协议逆向——HelloHPC 第 7 题的复杂度所在
5. "并行策略随便选"：TP 通信最频繁只能放节点内（NVLink 带宽高），PP 适合跨节点——**拓扑决定切法**
6. "GPU 快是因为核多"：核多只是表象，本质是**吞吐导向设计**（warp 锁步+大规模线程藏访存延迟）——对分支发散的串行代码 GPU 反而慢

## 9. 与超算比赛的联系

- **HelloHPC 第 7 题（Amazing LLM，150 分）**：编译部署 vLLM（30 分）+ 优化 6bit 解量化（70 分，对数插值）——本模块第 7 节就是它的知识地图；解量化优化的正确打开方式 = C++ 位运算自定义算子（PyTorch 的 torch extension 接口题目已给好）
- **ASC 决赛 AI 赛题**（LLM 微调/推理/AIGC）：并行策略选型（第 6 节）+ 推理系统调优（第 7 节）直接对应
- **与模块 05 的分工**：本模块给"系统观"，05 给"手写 CUDA 的肌肉"——两手都要有
- **学习路线建议**：先 03/04（并行地基）→ 本模块建立系统图 → 05 CUDA 实操 → 回头二刷 CSE234 的 GPU 编译章节，收获翻倍
