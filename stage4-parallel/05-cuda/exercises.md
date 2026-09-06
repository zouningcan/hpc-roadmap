# 模块 05 练习：CUDA

> B 卷为"纸上执行"（无 GPU 可做）：手推线程编号/内存行为；⚡ = 有 NVIDIA GPU（本机/云/WSL2-CUDA）才做。
> 手推时画 grid→block→thread 的编号图，别心算。

## A 概念题

**A1** 用"藏延迟"解释 GPU 为什么需要几万线程：单线程等一次显存 ~500 周期，若只有 1 个线程，带宽利用率多少？SM 里有 64 个 warp 可切换呢？
*考察点：吞吐导向设计——线程=延迟的填充物。*

**A2** grid/block/thread 三层各对应什么硬件资源？block 内线程能共享什么、不能共享什么？两个 block 间如何同步？
*考察点：层次模型；shared memory 边界；grid 级同步=kernel 边界。*

**A3** `vecAdd<<<100, 256>>>(..., n=25599)`：总启动线程数多少？哪些线程真正干活？"守卫"写在哪里？
*考察点：启动配置算术；越界守卫。*

**A4** 合并访存（coalescing）的条件是什么？warp 的 32 个线程分别读 `a[i]`（i 连续）和 `a[i*stride]`（stride=32）时，显存事务数差多少倍？
*考察点：合并访存——GPU 性能第一守则。*

**A5** 为什么说"小任务别上 GPU"？给出定量判断依据（两类开销的对比）。
*考察点：PCIe 搬运 vs 计算收益；算术强度门槛。*

## B 动手题（纸上执行为主）

**B1（编号手推）**：`kernel<<<3, 8>>>(...)`（grid 3 块 × 8 线程），写出全部 24 个线程的 `blockIdx.x`、`threadIdx.x`、全局编号 `i` 的表格（前 12 行即可）。改 `<<<2, 12>>>` 呢？i 的计算公式不变时，哪些值变了？
*考察点：三维编号的平面版；启动配置对编号的影响。*

**B2（shared memory tiling 手推）**：把 16×16 的输入 tile 装进 shared memory：每个 block 256 线程（16×16），写出每个线程负责搬运的元素公式 `tile[ty][tx] = in[row0+ty][col0+tx]`，并回答：相比每个线程直接从 global 读 16 次，tiling 后 global 读多少次/复用多少次？
*考察点：tiling 的搬运-复用账——GEMM 优化的种子。*

**B3（warp 发散手推）**：一个 warp（32 线程）执行：
```cuda
if (threadIdx.x % 2 == 0) heavy();   // 16 线程走
else light();                          // 16 线程走
```
实际耗时接近 16×heavy + 16×light 还是 max(16×heavy, 16×light)？为什么？怎么改写让两路不发散？
*考察点：SIMT 锁步；分支重排。*

**B4（归约手推）**：8 个线程在 shared memory 里做树形归约（相邻配对，步长 1,2,4）：画出每步后 shared memory 的内容变化（输入 [3,1,4,1,5,9,2,6]）。第 k 步有几个线程在干活？这暴露什么性能问题？
*考察点：归约的树形结构；后半程闲置（warp shuffle 动机的由来）。*

## C 挑战题（⚡ 有 GPU 做 / 无 GPU 写伪代码+审查）

**C1（⚡ vectorAdd 全流程）**：实现 vecAdd：cudaMalloc/cudaMemcpy/kernel/同步/copy 回/check。用 `compute-sanitizer` 验证无越界，n=1e8 计时并与 CPU 版对比。回答：搬运时间 vs 计算时间的比值——这个比说明 GPU 的甜蜜点在哪里？
*考察点：全流程肌肉；搬运开销的定量感知。*

**C2（共享内存矩阵乘）**：朴素 GPU matmul（每线程算一个 C 元素，global 直读）→ shared memory tiling 版（16×16 tile）。⚡ 跑 n=1024 对比两版 GFLOPS；无 GPU 则写两版完整代码并互相 review：tile 循环边界、`__syncthreads()` 位置、守卫。
*考察点：模块 07 "GEMM 三板斧" 的动手落地。*

**C3（设计题：stencil 的 halo）**：一维热传导 `new[i]=(old[i-1]+old[i+1])/2`，block 尺寸 256：
1. 每个 block 除自身 256 个元素外，还需读哪些"邻居块"的元素？（几个？）
2. 设计 shared memory 数组的尺寸（含 halo），写出搬运公式与 `__syncthreads()` 的两个必要位置
3. （口答）这与 MPI 的 halo 交换（04 模块 C2）在思想上有何同构？
*考察点：halo 模式——CPU 多进程与 GPU 多 block 的同构，系统观连点。*
