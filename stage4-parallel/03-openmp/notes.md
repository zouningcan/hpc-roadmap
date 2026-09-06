# 模块 03：OpenMP 共享内存并行

> **来源**：[LLNL OpenMP 教程](https://hpc-tutorials.llnl.gov/openmp/)（Blaise Barney，渐进式经典）；[openmp.org 官方资源](https://www.openmp.org/resources/)
> 配套案例：华科 HPC-roadmap 的 sum_array 归约题（2026-09-05 已预讲，本讲正式收编）；HelloHPC OceanSim 题（`-fopenmp` 编译）
> 环境提示：全部示例可在 WSL 上运行（`sudo apt install gcc` 即含 OpenMP 支持）

## 0. 一句话定位

OpenMP = **在 C/C++/Fortran 代码里插"编译器指令"，让一排线程分吃你的循环**。单节点多核（4~128 核）并行的第一武器，HelloHPC 4 核题（OceanSim）到 128 核题（Graph500/RopeNet）的节点内主力。

## 1. 编程模型：Fork-Join 与共享内存

程序从**单线程**启动，遇到并行区域就 **fork** 出一队线程（含主线程），区域结束 **join** 回单线程：

```
主线程 ──┬── 并行区域 ──┬──
          │  (线程0-3    │
          │   分吃循环)   │
          └──────────────┘
```

两个前提要焊死：

- **共享内存**：所有线程看得见同一块地址空间——这是 OpenMP 的世界观（MPI 是反过来的，各进程私有内存靠发消息）
- **线程 ≠ 进程**：线程轻（共享地址空间，创建快，通信零拷贝），线程间抢数据要靠同步

## 2. API 三支柱

| 支柱 | 是什么 | 例子 |
|------|--------|------|
| 编译指令（directives） | `#pragma omp ...`，指挥编译器 | `#pragma omp parallel for` |
| 运行时库（libgomp 等） | 程序里调用的函数 | `omp_get_thread_num()` |
| 环境变量 | 运行前控制行为 | `OMP_NUM_THREADS=8` |

编译：**`gcc -fopenmp sum.c -o sum`**——`-fopenmp` 一开，编译器才认识那些 pragma（不开也不报错，pragma 被当注释忽略——这是 OpenMP 的"可串行性"设计：指令拿掉程序仍是对的串行版）。

## 3. 并行区域：`parallel`

```c
#include <omp.h>
#include <stdio.h>
int main() {
    #pragma omp parallel
    {
        int id = omp_get_thread_num();       // 我是几号线程（0 起）
        int n  = omp_get_num_threads();      // 一共几个线程
        printf("hello from thread %d of %d\n", id, n);
    }
    return 0;
}
```

```bash
gcc -fopenmp hello.c -o hello
OMP_NUM_THREADS=4 ./hello     # 运行时定线程数（默认通常是核数）
```

注意：`#pragma omp parallel` 会让**每个线程把大括号里的代码整个跑一遍**（4 线程 = 4 份 hello）——它是"复制执行"，不是"分吃"。分吃靠下一节。

## 4. 工作共享：`parallel for` 与伙伴

**`for`**——把循环的迭代分给线程们，最常用构造：

```c
#pragma omp parallel for
for (int i = 0; i < N; i++) c[i] = a[i] + b[i];   // N 次迭代被均匀切块
```

组合写法 `parallel for` = parallel + for 两个指令的合并简写（等价于 parallel 区域里包一个 for）。

工作共享家族：

| 构造 | 分什么 |
|------|--------|
| `for` | 循环迭代 |
| `sections` | 几段互不相干的代码块 |
| `single` | 只让一个线程执行（如打印头信息） |
| `task` | 动态任务（递归/不规则并行，进阶） |

**`for` 可并行化的铁律：迭代之间相互独立**（无循环携带依赖，loop-carried dependency）。反例：

```c
for (int i = 1; i < N; i++) a[i] += a[i-1];   // 前缀和：i 依赖 i-1 —— 不能直接并行！
```

## 5. 数据作用域：谁看得见谁

并行区域里的变量，默认行为：**并行区外声明的 → 共享（shared）；循环变量 → 自动私有**。显式控制用子句：

| 子句 | 语义 | 典型场景 |
|------|------|----------|
| `shared(x)` | 全体线程共用同一份 x | 只读大数组 |
| `private(x)` | 每线程一份**未初始化**副本 | 循环内的临时变量 |
| `firstprivate(x)` | private + **用原值初始化** | 临时变量需要初值 |
| `lastprivate(x)` | private + 退出时把"最后一次迭代"的值写回 | 罕用 |
| `reduction(op: x)` | 各线程私有累加 → 结束时按 op 合并 | **求和/求最值（下节重讲）** |

🕷 **私有化的经典 bug**：循环体内的临时变量忘标 private，多线程写同一份 → 结果随机错。症状是"1 线程对、多线程错、每次错得还不一样"——听到这个描述先查作用域子句。

## 6. 归约（reduction）：本模块的皇冠（预讲案例归位）

**问题**（华科 sum_array 原型）：`sum += a[i]*b[i]` 想并行，但所有线程抢着更新同一个 sum——写冲突。

**reduction 子句**：

```c
double dot = 0.0;
#pragma omp parallel for reduction(+:dot)
for (int i = 0; i < N; i++) dot += a[i] * b[i];
```

它默默干了三件事（**归约三件事**，预讲背过）：

1. **私有化**：每线程领一个 `dot` 私有副本（初始化为 op 的单位元，`+` → 0）
2. **分工**：迭代切块，各线程累加自己的私有副本——互不干扰
3. **合并**：并行区结束时，把所有私有副本按 `+` 归并回原变量

**优化阶梯**（从 baseline 到满分，爬梯实录）：

```c
// L0 baseline：朴素串行
for (i) dot += a[i]*b[i];
// L1 加 OpenMP：reduction 一行起飞
#pragma omp parallel for reduction(+:dot)
// L2 SIMD 向量化：让每核的每次迭代也并行（AVX/NEON 一条指令算多个）
#pragma omp parallel for simd reduction(+:dot)
// L3 memory-bound 认知：a[i]*b[i] 每次迭代只有 2 次读 1 次乘 1 次加，
//    算术强度 ~0.25 flop/B —— 内存带宽先饱和，加速比封顶在带宽倍数而非核数
```

🕷 **浮点归约不满足结合律**：`+` 对浮点数严格说不可结合（(0.1+0.2)+0.3 ≠ 0.1+(0.2+0.3)），多线程归约的合并顺序不定 → **结果与串行版末位几位不一致**。这是特性不是 bug；若题目校验按位一致，可开 `-ffast-math` 换编译器重排（⚠️ 它还会破坏 NaN/Inf 语义，比赛用前先确认正确性校验方式——HelloHPC OceanSim 给了 1e-6/1e-4 容差就是为此留的余地）。

## 7. 同步构造：不打架的四种手段

| 构造 | 干什么 | 开销/备注 |
|------|--------|-----------|
| `#pragma omp critical` | 临界区：同一时刻只许一个线程进 | 最重；保护 printf/非原子更新 |
| `#pragma omp atomic` | 原子更新单条内存语句（`x+=y`） | 轻于 critical，能用就用 |
| `#pragma omp barrier` | 栅栏：全员到齐才放行 | 隐式存在于多数构造末尾 |
| `#pragma omp master` / `single` | 只由 0 号/某一线程执行 | 打印进度用 |

启发式：**能用 reduction 就不用锁，能用 atomic 就不用 critical**——开销逐级递增。

## 8. 调度：`schedule` 子句

迭代不均匀时（如 `if (a[i]>0) 大计算 else 小计算`），默认切块会饿死快线程：

```c
#pragma omp parallel for schedule(static)    // 均匀预切：迭代均匀时最快（零调度开销）
#pragma omp parallel for schedule(dynamic, 64)   // 抢任务：先到先得，块 64；负载不均时用
#pragma omp parallel for schedule(guided)    // 大块起步渐小：兼顾两者，不均匀首选
```

选择口诀：**均匀用 static，不均匀用 guided，不可预测用 dynamic**。代价：调度越动态，管理开销越大。

## 9. 环境变量与常用运行时函数

```bash
OMP_NUM_THREADS=8 ./a.out        # 线程数（最常用）
OMP_PROC_BIND=true ./a.out       # 线程绑核（NUMA 亲和——跨 NUMA 访存慢，绑核是 HPL 级调优第一步）
OMP_SCHEDULE="dynamic,64"        # schedule(runtime) 时从这里读
```

```c
omp_get_thread_num()   // 我的编号
omp_get_num_threads()  // 队伍人数
omp_get_wtime()        // 壁钟计时（计性能用它，别用 clock()——多线程 clock 会把所有线程时间加总）
```

## 10. 常见误区清单

1. 忘 `-fopenmp`：pragma 被当注释，**静默变串行**——不报错、只是慢（速度疑云第一排查项）
2. 循环携带依赖硬并行：前缀和、迭代蒙特卡洛这类，直接加 pragma 得到的是**错误结果**
3. 循环内临时变量忘 private：多线程随机错（1 线程对多线程错=作用域问题）
4. 以为加速比 = 核数：memory-bound 任务带宽先饱和（sum_array 实测见 exercises B3）
5. 浮点归约结果与串行不一致就当 bug：先想结合律与合并顺序
6. 在循环里 critical 打日志：IO 锁把并行变串行——性能剖析时先摘日志

## 11. 与超算比赛的联系

- **OceanSim（HelloHPC 4 核）**：编译命令 `g++ ... -fopenmp -O3` 已写明答案方向——热点循环 `parallel for`，误差约束决定能不能 `-ffast-math`
- **WRF（128 核）**：真实气象模式 = **MPI（跨节点）+ OpenMP（节点内）混合**，正是模块 04 的预告
- **ARM 鲲鹏**：NEON 向量化的 `parallel for simd` 在 aarch64 上由编译器自动映射 `vaddq_f32` 一族——模块 06 展开
- **Slurm 交接**：sbatch 脚本里 `-c 128` 申请的核数，就是 OpenMP 线程该设的数量（`OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK`）——模块 01 与本模块的接头暗号
