# 模块 07 练习：GPU 与 ML 系统（CSE234）

> 分层：A 概念 / B 动手 / C 挑战。
> 本模块以概念为主（手写 CUDA 在模块 05），B 卷 CPU 即可完成；先预测再验证。

## A 概念题

**A1** 计算图 + autodiff：画出 `y = (a+b)*b` 的计算图，标出前向求值顺序；再用链式法则手推 ∂y/∂a。框架的"自动"微分到底自动化了哪一步？
*考察点：前向/反向两遍图遍历；autodiff 的系统本质。*

**A2** 为什么说"现代 GPU 上优化 GEMM 等于优化一切"？从三层展开：全连接层、卷积、attention 各自怎么变成 GEMM 的？
*考察点：GEMM 中心地位；im2col 与 batched matmul。*

**A3** 数据并行/张量并行/流水线并行各切什么？各自的通信原语与通信频率？为什么张量并行只能放在节点内（NVLink），流水线并行适合跨节点？
*考察点：3D 并行选型；通信量与拓扑匹配。*

**A4** KV cache 解决什么问题？没有它，生成第 n 个 token 的计算复杂度是多少（相对上下文长度 n）？PagedAttention 又解决了 KV cache 自身的什么问题？借鉴了 OS 的哪个机制？
*考察点：自回归推理的显存经济学；vLLM 双核。*

**A5** 训练是 compute-bound，推理（逐 token 生成）是 bandwidth-bound——用"每生成一个 token 要做什么"解释这句话，并推论：推理优化的重点应放在提高 FLOPs 还是提高显存带宽利用率？
*考察点：roofline 在 ML 系统的应用。*

## B 动手题

**B1（亲手 autodiff：不上框架）**：用 Python 手写一个小计算图的前向与反向（不用 PyTorch）：
```python
# y = (a+b)*b，在 a=2, b=3 处求 ∂y/∂a 和 ∂y/∂b
# 提示：引入中间变量 c = a+b；前向存 c；反向 ∂y/∂b = ∂y/∂c*∂c/∂b + b 的直连项
```
要求：手推结果先写在注释里，再让代码算出来对答案（答案：∂y/∂a=3, ∂y/∂b=8）。
*考察点：反向传播的链式法则内核——框架自动化的是"乘一遍局部导数"。*

**B2（量化/解量化体感）**：实现最朴素的 per-tensor 线性量化：
```python
import numpy as np
x = np.random.randn(10000).astype(np.float32)
scale = np.abs(x).max() / 7          # 映射到 [-7,7] 的 INT8 范围
q = np.clip(np.round(x / scale), -7, 7).astype(np.int8)
deq = q * scale                       # 解量化
print("最大误差:", np.abs(deq - x).max(), " 压缩比: 4x")
```
跑通后回答：① 误差大约多大量级？为什么？② 若把范围压到 [-3,3]（INT4 风格），误差怎么变？③ 这解释了 HelloHPC 第 7 题的 6bit 方案为什么需要 scale 因子存下来才能解量化——scale 相当于这里的什么？
*考察点：量化的数学本质（缩放+取整）；解量化协议的必要性。*

**B3（KV cache 效果测算）**：某 7B 模型（FP16 权重 14GB），上下文 4k tokens，KV cache 约每 token 0.5MB：
1. 显存里权重 + 一条请求的 KV cache 共多少 GB？
2. 同时服务 32 条请求呢？此时显存压力主要来自谁？
3. PagedAttention 号称把可用显存利用率从 ~30% 提到 ~90%+，对并发数意味着什么？
*考察点：推理显存账本；服务层优化的价值量化。*

**B4（读文档实战：vLLM 速览——HelloHPC 第 7 题热身）**：浏览 vLLM 官方文档（docs.vllm.ai）的 Quickstart 与 PagedAttention 介绍页，回答：
1. vLLM 相对 HuggingFace transformers 推理的卖点（两个词，提示：一个关于显存、一个关于批处理）
2. 部署一个模型需要哪三样（模型、什么引擎、什么 API 服务）？
3. （选做）WSL CPU 上装 vLLM 跑通一个小模型（`pip install vllm`，CPU 版受限但足以见流程），记录安装中踩的坑
*考察点：信息素养——比赛第 7 题第一步"编译运行 vLLM"就是读文档干活。*

## C 挑战题

**C1（roofline 算账）**：某 GPU 算力 100 TFLOPS（FP16）、显存带宽 2 TB/s。一个 7B 模型推理，每 token 需读全部权重一次（14GB）、算 ~14 GFLOPs：
1. 算术强度多少 FLOP/B？落在 roofline 哪一侧？
2. 理论上限每秒生成多少 token？（两条上限取 min）
3. batch=32 时（权重读一次服务 32 个 token，算力 ×32），算术强度变多少？新上限？——解释 continuous batching 为什么"白捡"吞吐
*考察点：roofline 实战；batch 摊薄权重的带宽账。*

**C2（设计题：给比赛写选型）**：ASC 决赛要在一个 8 卡 A800 节点上微调一个 13B 模型（FP16 权重 26GB，单卡 80GB），训练时还有优化器状态（约权重 3 倍）。回答：
1. 单卡装得下吗？（26+78=?）至少需要几张卡？
2. 你会选 DP/TP/PP/Zero 哪种组合？（提示：模型 26GB < 80GB 单卡，瓶颈在优化器状态与梯度）
3. 通信上最贵的操作是什么？对应 NCCL 的哪个原语？（04 模块呼应）
*考察点：显存账本 + 并行策略选型——ASC 集群设计题的标准题型。*

**C3（贯通题：一句话地图）**：把以下概念按"系统栈层"排序，并用一句话说明相邻两层的关系：
`warp` / `PagedAttention` / `Triton 算子` / `Allreduce(NCCL)` / `GEMM tile` / `数据并行` / `KV cache block` / `shared memory tiling`
*考察点：整条栈在脑中的空间感——CSE234 结业的样子。*
