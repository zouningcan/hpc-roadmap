# 模块 05：CUDA GPU 编程基础

> **来源**：PMPP 4th ed（*Programming Massively Parallel Processors*, Hwu/Kirk/El Hajj，前 10 章骨架）；[NVIDIA CUDA C++ Programming Guide](https://docs.nvidia.com/cuda/cuda-c-programming-guide/)；[NVIDIA DLI 免费课](https://www.nvidia.com/en-us/training/)
> 衔接：模块 07 已建 GPU 系统观（warp/SM/内存层级），本模块动手写 kernel
> 环境提示：无 GPU 也能学——本模块 B 卷用"纸上执行 + 概念验证"为主，装了 NVIDIA 卡（或云 GPU/WSL2 CUDA）的题标注 ⚡

## 0. 一句话定位

CUDA = **把 C 函数（kernel）撒到几万个线程上并行执行**的编程模型。GPU 的工作方式：**用海量线程把访存延迟藏起来**（一个线程等内存时，调度器无缝切到另一个算）——与 CPU"大缓存+分支预测"是两种哲学。

## 1. 编程模型三层结构

```c
// kernel：一个"线程的程序"（每个线程都执行这段）
__global__ void vecAdd(float* a, float* b, float* c, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;   // 我的全局编号
    if (i < n) c[i] = a[i] + b[i];
}

int main() {
    // ... 显存分配 cudaMemcpy ...
    vecAdd<<<numBlocks, threadsPerBlock>>>(da, db, dc, n);   // 启动配置：网格×块
}
```

| 层 | 含义 | 量级 |
|----|------|------|
| **grid**（网格） | 一次 kernel 启动的全部线程 | 1 |
| **block**（块） | 一组共享 shared memory、可同步的线程 | 数百~数千个 |
| **thread**（线程） | 最小执行单元，有唯一 `threadIdx` | 总计可达百万 |
| **warp**（硬件） | 32 个线程锁步（SIMT）执行 | 调度/发散的单位 |

启动配置的算术：`numBlocks = ceil(n / threadsPerBlock)`（常取 256），`i < n` 守卫是**越界防线**——最后一个块的多余线程靠它刹车。

## 2. 内存层级：性能的全部秘密

| 层 | 谁可见 | 速度 | 用途 |
|----|--------|------|------|
| 寄存器 | 线程私有 | 最快 | 局部变量 |
| **shared memory** | block 内共享 | ~100× 显存带宽 | **手工缓存/协作**（tiling 的舞台） |
| L2 | 全 GPU | 中 | 硬件自动 |
| **global（显存 HBM）** | 全体 | 慢但大 | 输入输出 |

**CUDA 优化的第一性原理：把数据从 global 搬进 shared，复用足够多次，摊薄搬运成本**。GEMM 的 tiling（模块 07）就是这句话的极致版。

## 3. 性能四守则（PMPP 第 5-6 章）

1. **合并访存（coalescing）**：一个 warp 的 32 个线程访问**连续** 32×4B = 128B → 硬件合成 1 次事务；跨步/随机访问 → 32 次独立事务，带宽碎成渣——**CPU 的"连续访存"在 GPU 是生存底线**
2. **避免发散（divergence）**：warp 锁步，`if/else` 两路都走（串行执行两遍）——分支按 warp 对齐写
3. **占用率（occupancy）**：每个 SM 的寄存器/shared 用量决定驻留线程数——留够线程才能藏延迟，但不盲目拉满（shared 用太多反而降占用）
4. **少同步**：`__syncthreads()` 是 block 内 barrier，多用一次少一分并行；global 同步要靠下一次 kernel 启动（天然 barrier）

## 4. 标准模式三连（PMPP 后半本的骨架）

- **卷积/stencil**：邻域计算——`__constant__` 存卷积核，shared memory 缓存输入 tile + halo 边界（模块 04 的 halo 交换的 GPU 版）
- **归约（reduction）**：树形两两合并——shared memory 折半 + `__syncthreads()`；相邻配对→交叉配对→warp shuffle 逐步优化（OpenMP reduction 的 GPU 全功率版）
- **前缀和（scan/prefix sum）**：看似串行依赖，Hillis-Steele（双缓冲）与 Blelloch（工作高效）两套并行算法——**"看似必须串行"的算法也能并行**是并行思维的分水岭

## 5. 三类函数与 API 速查

| 前缀 | 在哪跑 | 谁调 |
|------|--------|------|
| `__global__` | GPU（kernel 入口） | CPU 调，`<<<>>>` 启动 |
| `__device__` | GPU | 只能 GPU 调（kernel 的函数） |
| `__host__` | CPU | CPU（默认） |

```c
cudaMalloc(&d_ptr, bytes);              // 显存分配
cudaMemcpy(d, h, bytes, cudaMemcpyHostToDevice);  // 主机↔设备搬运（贵！）
cudaDeviceSynchronize();                // 等 kernel 完（异步启动）
```

🕷 **两大坑**：① kernel 启动是**异步**的——忘了 `cudaDeviceSynchronize` 就读结果，读到的是没算完的；② 搬运（PCIe ~16GB/s）比算还慢——**小任务别上 GPU**（搬运开销 > 计算收益），GPU 吃"大块、规则、重计算"的活。

## 6. 编译与调试

```bash
nvcc -O3 -arch=sm_80 vec.cu -o vec    # .cu 文件；-arch 对齐目标卡（A800=sm_80）
nvcc -G …                              # 可调试版（禁优化）
⚡ nvidia-smi                          # 显卡在不在/显存占用
⚡ compute-sanitizer ./vec             # 越界/竞态检测（CUDA 的 valgrind）
⚡ ncu ./vec                           # Nsight Compute：kernel 级剖析（模块 02 的 GPU 版）
```

错误检查纪律：每个 CUDA API 返回值都该查（`cudaGetLastError()`）——GPU 出错是异步爆雷，不查栈早就丢了。

## 7. 常见误区清单

1. `blockIdx.x * blockDim.x + threadIdx.x` 写错顺序/忘加守卫——越界或漏算，入门第一大坑
2. 忘 `cudaMemcpy` 回来就打印：读到旧内存——异步性坑
3. 把 CPU 指针直接传 kernel：主机指针与设备指针是**两个地址空间**——段错误
4. shared memory 用过头：occupancy 反而掉（每 SM 的 shared 有上限）——性能调优不是堆快存
5. warp 发散当免费：`if` 里两路都执行——别在 kernel 里写重分支
6. 期望小矩阵乘在 GPU 上赢：搬运吃掉一切——**算术强度不够的活 CPU 更快**（07 模块 roofline 结论）

## 8. 与超算比赛的联系

- **ASC/SC 赛题 GPU 化**：AI 赛题全在 GPU 上——本模块是读懂赛题代码（CUDA/PyTorch 混合）的门票
- **A800 节点（交我算 a800 队列）**：80GB HBM、sm_80 架构——nvcc `-arch=sm_80` 与 nvidia-smi 看到的就是这个
- **模块 07 的衔接**：vLLM/量化算子优化 = 写高效 kernel；HelloHPC 第 7 题的 C++ 算子是 CPU 版先行，GPU 版是进阶
- **学习路径**：无卡先过概念+纸上执行（B 卷）→ 有卡/云 GPU 跑 vectorAdd→tiling matmul（C 卷）→ PMPP 习题
