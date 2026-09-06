# 模块 05 答案：CUDA

## A 概念题

**A1** 1 线程：算 500 周期、等 500 周期 → 带宽/算力利用率 ~50% 还算乐观，真实访存密集代码利用率个位数。64 warp 轮转：一个 warp 等内存的瞬间硬件立刻切下一个 warp——**只要线程数足够，SM 永远有活干**，延迟被"藏"进其他线程的计算里。GPU 的答案不是"让等待变短"（CPU 的缓存哲学）而是"让等待时别人干活"（吞吐哲学）。

**A2** grid=一次 kernel 启动（对应整卡）；block=SM 上的驻留单位（共享 shared memory + `__syncthreads()`；不同 block 间**不能**直接同步）；thread=warp 内的锁步单元。block 间同步：**没有直接手段**——靠 kernel 结束（隐式全局 barrier）或 cooperative groups（进阶），常规写法是拆成两次 kernel 启动。

**A3** 100×256 = **25600 线程启动**；`i = blockIdx.x*256 + threadIdx.x`，i < 25599 的线程干活（最后一个 block 的 1 个线程守卫刹车）。守卫：kernel 首行 `if (i < n)`。

**A4** 条件：warp 内相邻线程访问相邻地址（32×4B 落在同一 128B 段）。连续 `a[i]`：1 次事务（128B 一把抓）；`a[i*32]`（步长 128B）：每个线程各撞一条缓存行 → **32 次事务**，差 32 倍——带宽利用率 1/32。

**A5** 两笔账：① 搬运：主机↔显存走 PCIe ~16-32 GB/s，n MB 数据要 n/16GB s；② 计算：GPU 算力虽高但小任务塞不满。判断式：**算术强度 × GPU 峰值算力 ≫ 搬运时间占比** 才值得上 GPU——即"数据搬过去 → 算很久 → 搬回来"里"算"必须占大头（模块 07 roofline：强度不足的活 GPU 反而输 CPU）。

## B 动手题

**B1** `<<<3,8>>>` 表格（前 12 行）：
| blockIdx | threadIdx | i |
|---|---|---|
| 0 | 0-7 | 0-7 |
| 1 | 0-7 | 8-15 |
| 2 | 0-7 | 16-23 |
`<<<2,12>>>`：blockIdx 0 的 i=0-11、blockIdx 1 的 i=12-23——**总线程数不变（24），i 分布变**（i 公式不变，块内人数变）。要点：同一组线程，启动配置不同全局编号完全不同。

**B2** 搬运：每 block 256 线程各搬 1 次（256 次 global 读装满 tile）；之后 block 内每线程要用的数据（比如一行卷积窗口跨 16 次输出）全部命中 shared。以每个输入元素被本 tile 内 16 次输出复用计：global 读从 16 次/元素 → **1 次/元素，省 16 倍**；代价 = 1 次 shared 写 + N 次 shared 读（快 ~100×）。tiling 的账：**搬运次数 ÷ 复用次数 = 加速上限**。

**B3** 接近 **16×heavy + 16×light**（总时间≈两路之和）。SIMT 锁步：warp 只有一个程序计数器，if/else 两路**串行各走一遍**（不走的路被掩码禁用但占时）。改写：按奇偶**重排数据/线程**，让同一 warp 全走同一路（例如偶偶一组、奇奇一组分两个 kernel/两个 warp 区域）——发散按 warp 粒度消除。

**B4** 输入 [3,1,4,1,5,9,2,6]，相邻配对步长 1→2→4：
```
步1（8线程）：3,1,4,1,5,9,2,6 → 4,1,5,1,14,9,8,6      (a[i]+=a[i+1], i=0,2,4,6 偶位)
  标准树形：i += stride，active = 8,4,2,1
步1 后：[4,1,5,1,14,9,8,6]
步2（4线程）：[4,6,5,15,14,9,8,6]（i=0,2,4,6 处 a[i]+=a[i+2] → [4,1,5,1,14,9,8,6]→a[0]+=a[2]=5? 视实现而定——交叉配对版）
步3（2线程）：… 最终 a[0]=31
```
（注：树形归约有两种配对习惯——相邻配对与交叉配对，手推时先声明用哪种；答案是 active 线程数 8→4→2→1。）
性能问题：**后半程每步一半以上线程闲置**（步长越大闲得越多），且 shared memory bank conflict 会出现——warp shuffle（warp 内寄存器直接交换）就是为收割这半程而生的。

## C 挑战题

**C1** 参考流程（骨架）：
```c
cudaMalloc(&da, n*sizeof(float)); …
cudaMemcpy(da, ha, bytes, cudaMemcpyHostToDevice);
vecAdd<<<(n+255)/256, 256>>>(da, db, dc, n);
cudaMemcpy(hc, dc, bytes, cudaMemcpyDeviceToHost);
cudaDeviceSynchronize();
```
n=1e8：搬运 ~2×400MB/16GB/s ≈ 50ms，kernel ~1-3ms——**搬运是计算的 ~20 倍**。结论：GPU 的甜蜜点 = "数据上去一次、算得够久"（强度高、复用多），vecAdd 这类流式任务其实在吃搬运的亏（真实场景里应与后续计算融合成 pipeline）。

**C2** 朴素版每线程 C[i][j] 读整行 A 与整列 B（2n 次 global）；tiling 版：
```c
for (int t = 0; t < n/16; t++) {
    As[ty][tx] = A[row][t*16+tx];          // 协作搬运 tile
    Bs[ty][tx] = B[t*16+ty][col];
    __syncthreads();                        // ① 等 tile 装满
    for (int k = 0; k < 16; k++) sum += As[ty][k] * Bs[k][tx];
    __syncthreads();                        // ② 等算完才能覆盖 tile
}
C[row][col] = sum;
```
两个 `__syncthreads()` 缺一：① 缺=读半成品 tile；② 缺=下一轮搬运覆盖了别人还在读的数据。⚡ 实测 tiling 版 GFLOPS 常是朴素版 5-10×。

**C3** 
1. block 负责 old[256*b .. 256*b+255]，计算需要 old[256b-1] 与 old[256b+256]——左右各 **1 个** halo 元素
2. `__shared__ float tile[258];` 搬运：`tile[tx+1] = old[start+tx]`，首尾两个特殊线程补 `tile[0]=old[start-1]`、`tile[257]=old[start+256]`；`__syncthreads()` 位置：搬运后（算之前）一次 + 循环内写回 global 后（覆盖 old 前）一次
3. 同构：MPI 进程的 halo 交换 = 把"邻居的边界数据"搬进自己的影子单元，GPU block 的 halo = 把"邻居块的数据"搬进 shared——**都是"计算单元只看局部 + 边界补一圈"**，只不过 MPI 用网络传、block 用 shared 读。分形结构：同一模式在进程/线程两个尺度重复出现——系统观的证据。
