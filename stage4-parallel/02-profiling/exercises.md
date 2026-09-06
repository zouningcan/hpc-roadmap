# 模块 02 练习：性能剖析

> 环境准备：`sudo apt install linux-tools-generic linux-tools-$(uname -r)` 或 `sudo apt install perf`（WSL 注意：内核 perf_event 支持有限，若不可用改用 gprof/valgrind 或时间戳插桩完成 B 卷，概念题不受影响）。
> 全部实验遵守三纪律：留 baseline、单变量、重验正确性。

## A 概念题

**A1** 采样与插桩两门派的原理、优缺点各是什么？为什么现代默认用采样派？
*考察点：profiling 两大范式。*

**A2** `perf stat` 输出 `IPC = 0.35, cache-misses = 28%`——判断这个程序是 compute-bound 还是 memory-bound，推理链是什么？如果换成 `IPC = 2.1, branch-misses = 15%` 呢？
*考察点：硬件计数器读法——roofline 的计数器版。*

**A3** 火焰图的 x 轴为什么是"字母序"而不是时间轴？宽度、y 轴、最宽的塔各代表什么？
*考察点：火焰图读法——最常见的读错是把 x 当时间。*

**A4** "先测量再优化"的完整纪律是三条（剖析三纪律），每条各防什么事故？
*考察点：baseline/单变量/正确性——writeup 的骨架。*

## B 动手题

**B1（造一个热点）**：编译运行下面的"故意病"程序，用剖析工具找出热点行并报告占比：
```c
// slow.c
#include <stdio.h>
#include <math.h>
int main() {
    double s = 0;
    for (long i = 0; i < 100000000L; i++) {
        s += sin(i) * cos(i) * sqrt(i);   // 全是重复可预计算的东西
    }
    printf("%f\n", s);
}
// gcc -O2 slow.c -o slow -lm
```
任务：① `time ./slow` 记 baseline ② 剖析找热点函数 ③ 把三角函数积提取成查表/化简（sin·cos = sin(2x)/2，sqrt(i) 可增量更新），再 time——报提速比。写一份 5 行 mini-writeup（仿 OceanSim 满分骨架）。
*考察点：剖析→优化→复测的完整闭环。*

**B2（perf stat 判型）**：分别对下面两个程序跑 `perf stat`：
```c
// A: 计算密集——n² 次 fma，数据全在寄存器/缓存
// B: 访存密集——大数组随机下标访问 a[rand()%N]++
```
报告两者的 IPC 与 cache-misses，验证"数值型 vs 随机访存"的判型预期。
*考察点：两类 roofline 病人的计数器指纹。*

**B3（并行没生效的侦查）**：给某程序加了 `#pragma omp parallel for`，4 核机器还是跑不快：
```bash
htop          # 观察命令一
time ./prog   # 观察命令二
```
列出三种可能病因（忘 -fopenmp/线程数=1/循环太短不值得并行）与各自的确认方法——哪种用 htop 看？哪种用 echo $OMP_NUM_THREADS？哪种是剖析里"parallel 区域占比太小"？
*考察点：03 模块联动；"静默串行"的取证学。*

**B4（gprof/valgrind 备用路）**：若 perf 不可用（WSL 常见），用插桩派完成 B1：`gcc -pg`（gprof）或 `valgrind --tool=callgrind ./slow` + `callgrind_annotate`。对比插桩派与采样派的输出差异（函数被调次数 vs 采样占比）。
*考察点：两门派的动手体感。*

## C 挑战题

**C1（OceanSim 全流程模拟）**：写一个双层循环热传导 stencil（j 内层、i 外层，但按 `a[j*W+i]` 存储刻意造成跨步访存）：
1. perf stat：确认 memory-bound
2. perf report：确认热点在内层循环
3. 循环交换（i 内层化）恢复连续访存：报 cache-misses 前后对比与耗时提速比
4. mini-writeup：三行（症状/处方/疗效）
*考察点：跨步访存→循环交换——剖析驱动优化的最经典处方。*

**C2（伪共享取证，进阶）**：拿 03 模块 C2 的伪共享实验代码，用 `perf c2c record/report`（可用则）或对比实验（padding 前后耗时）取证，解释"两变量无逻辑共享为何打架"的证据链。
*考察点：性能问题的法庭级证明——没有证据的优化理由都是猜。*
