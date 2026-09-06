# 模块 07 答案：GPU 与 ML 系统（CSE234）

## A 概念题

**A1** 计算图：
```
a ──┐
    ├─→ c=a+b ──┐
b ──┤           ├─→ y=c*b
    └───────────┘
```
前向：c = a+b = 5，y = c*b = 15。反向（链式法则）：∂y/∂c = b = 3；∂y/∂c·∂c/∂a + ∂y/∂b·0 → ∂y/∂a = b·1 = 3；∂y/∂b = b·1（经 c）+ c·1（直连）= 3+5 = 8。
框架自动化的是：**记录前向的局部导数，按逆拓扑序自动乘链**——你只写前向，反向图自动生成。

**A2** 
- 全连接层：`y = Wx` 本来就是矩阵-向量乘，batch 起来 = GEMM
- 卷积：im2col 把每个滑窗展开成行 → 卷积变成"展开矩阵 × 权重矩阵"的 GEMM（空间换时间的经典）
- attention：softmax(QKᵀ/√d)V = 两次 batched GEMM 夹一个 softmax
所以 GPU FLOPs 大头全是 GEMM，GEMM 快 = 全局快。

**A3** 
| 策略 | 切什么 | 通信 | 频率 |
|---|---|---|---|
| DP | 数据 batch | Allreduce 梯度 | 每步 1 次 |
| TP | 单层权重矩阵 | 每层前向 2 次 + 反向 2 次 | **每层每步都要** |
| PP | 层的归属 | 点对点激活传递 | 每微批 1 次沿流水线 |
TP 通信在最内层循环（每层前后向），需要最高带宽 → 放 NVLink 互联的节点内；PP 只有段间交界处传激活，量小且可与其他计算重叠 → 适合带宽低的跨节点网络。**原则：通信越频繁，越要放带宽高的地方**。

**A4** KV cache 缓存历史 token 的 Key/Value，免去每生成新 token 重算全部历史。没有它：生成第 n 个 token 要对全部 n 个历史重算 attention，总复杂度 O(n²)；有它每步只算新 token，O(n)。
PagedAttention 解决 KV cache **显存碎片**：传统做法按最大长度预分配连续显存，实际利用 ~20-40%；它借 OS **分页（paging）**机制把 cache 切成固定 block 按需分配、逻辑连续物理分散——利用率 90%+，同显存并发数翻数倍。

**A5** 生成 1 个 token = 前向一遍网络，计算 ~2×7 GFLOPs（7B 模型），数据 = 读全部权重 14GB（batch=1 时）。算术强度 ≈ 14G/14G = 1 FLOP/B——远低于 GPU 平衡点（~50 FLOP/B），是 **bandwidth-bound**：GPU 算力大量闲置，权重从显存搬过来的速度决定一切。推论：推理优化优先提高带宽利用率（量化压缩权重、batch 摊薄权重读取、FlashAttention 减少访存），而不是堆 FLOPs。

## B 动手题

**B1** 
```python
a, b = 2.0, 3.0
c = a + b          # 前向，存中间量
y = c * b
dc_da, dc_db = 1.0, 1.0        # 局部导数
dy_dc, dy_db = b, c            # ∂y/∂c=b, ∂y/∂b(直连)=c
dy_da = dy_dc * dc_da          # = 3
dy_db = dy_dc * dc_db + dy_db  # = 3*1 + 5 = 8（两条路径相加！）
print(dy_da, dy_db)            # 3.0 8.0
```
关键点：b 有两条路径到 y（经 c 和直连）——**多路径的导数要相加**，这是链式法则在图上的"多入边求和"规则。

**B2** ① 最大误差 ~ scale 的量级（约 0.1-0.2，因为取整误差均匀分布在 ±scale/2）——由"值的动态范围 ÷ 量化级数"决定。② 范围 [-3,3] 时 scale 变大（|x|max/3），取整粒度变粗，误差约放大 7/4 倍——**位数越少误差越大**。③ scale 相当于这里的 `np.abs(x).max()/7`——量化时算好存下来，解量化时 `q * scale` 恢复数量级。HelloHPC 的 6bit 方案同理：解量化必须知道每组的缩放因子，这正是权重文件里除了位数据还要存元信息的原因。

**B3** 
1. 权重 14GB + 单请求 cache 4k×0.5MB = 2GB → **16GB**
2. 32 条：14 + 32×2 = **78GB**——逼近 80GB 卡的极限，且此时 **KV cache（64GB）远大于权重**，是显存压力主体
3. 利用率 30%→90% 意味着同样显存能装 ~3 倍的有效 cache → 并发数 ~3 倍 → 吞吐 ~3 倍。**显存管理的工程优化直接换算成服务吞吐**——这就是 vLLM 论文的核心卖点。

**B4** 参考答案：① 两个卖点：**PagedAttention**（显存）与 **continuous batching**（批处理）。② 三样：模型权重（HF 格式）、`LLM` 推理引擎（vLLM core）、API 服务（OpenAI 兼容 server）。③ 踩坑记录自由发挥（常见：WSL 显卡驱动、CPU 版依赖编译、版本冲突）——记录本身就是要练的能力。

## C 挑战题

**C1** 
1. 算术强度 = 14 GFLOPs / 14 GB = **1 FLOP/B**，远低于平衡点 100T/2T = 50 → **bandwidth 侧**，上限由带宽决定
2. 上限 = min(100T FLOPS ÷ 14 GFLOPs, 2 TB/s ÷ 14 GB) ≈ min(7142, 143) ≈ **143 token/s**
3. batch=32：算 32×14 = 448 GFLOPs，仍只读一次权重 14GB → 强度 32 FLOP/B，逼近平衡点 50 → 新上限 ≈ min(100T/448G≈223, 2T/14G≈143)——**仍卡在带宽 143 token/s，但这是 32 条请求共享的 143**，单请求平均吞吐虽降，总吞吐却 ~32 倍于 batch=1 的 143。continuous batching 的本质：**把"每 token 必读的权重"摊给尽量多的请求**——带宽账本上的团购。

**C2** 
1. 权重 26GB + 优化器状态（FP32 主权重+动量+方差 ≈ 12 字节/参数）156GB → 单卡远不够；80GB×8 = 640GB，**数据并行 8 卡装得下总量**，但纯 DP 要每卡放全套（26+156+梯度 52 ≈ 234GB > 80GB）→ 纯 DP 不行
2. 选 **ZeRO（Zero Redundancy Optimizer）**系列（DP 的显存优化变体）：ZeRO-2/3 把优化器状态/梯度/参数分片到各卡，通信仍是 Allreduce 家族（+分片重聚）。模型本身 26GB 可整份放单卡，无需 TP/PP 的复杂切分——**瓶颈在优化器状态，答案就是 ZeRO-DP**
3. 最贵操作：每步反向结束的**梯度全局归约** → `ncclAllReduce`（ring 算法，04 模块 C3 的同款）。若 ZeRO-3 还要做参数前向重聚（all-gather）。

**C3** 自底向上的排序与关系：
```
warp → GEMM tile → shared memory tiling → Triton 算子 → 数据并行 → Allreduce(NCCL) → KV cache block → PagedAttention
```
- warp 是 SIMD 执行单位 → GEMM tile 切成的微运算由 warp 锁步执行（硬件↔算子）
- shared memory tiling 是 GEMM 快的机制，Triton 让你只写 tile 级运算、编译器生成上述细节（算子↔编译）
- 数据并行是训练层策略，其梯度同步靠 Allreduce(NCCL)（策略↔通信）
- KV cache block 是推理显存的管理单元，PagedAttention 是它的分配器（服务↔OS 机制）
能讲清这 4 对"相邻关系"，CSE234 的系统观就到手了——剩下的深度去模块 05（CUDA 实操）和二刷课程补齐。
