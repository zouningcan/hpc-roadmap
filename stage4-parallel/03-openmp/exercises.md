# 模块 03 练习：OpenMP

> 分层：A 概念 / B 动手 / C 挑战。🕷 = 实战易错点。
> B 卷全部可在 WSL 完成（`sudo apt install build-essential` 即有 gcc+OpenMP）。先预测再执行。

## A 概念题

**A1** 用 Fork-Join 模型解释：`#pragma omp parallel` 区域里的代码，4 线程会执行几遍？`#pragma omp parallel for` 呢？两者语义差在哪？
*考察点：复制执行 vs 分吃迭代——parallel 与工作共享的本质区别。*

**A2** 下面代码有错（多线程结果随机不对），指出病灶并给两种修法：
```c
int count = 0;
#pragma omp parallel for
for (int i = 0; i < N; i++)
    if (a[i] > 0) count++;          // 统计正数个数
```
*考察点：写冲突；atomic 与 reduction 两个修法的取舍。*

**A3** 默写 **归约三件事**（reduction 子句背后发生的三步），并回答：浮点数归约的结果和串行版为什么可能末位不一致？这算 bug 吗？
*考察点：私有化/分工/合并；浮点结合律。*

**A4** 循环体：`if (a[i] > 100) s += heavy(a[i]); else s += 1;`（heavy 耗时远大于 1），想并行化该选 `schedule(static/dynamic/guided)` 哪个？为什么默认切法会慢？
*考察点：负载不均与调度策略。*

**A5** 一句话对比 OpenMP 和 MPI 的世界观差异，并据此判断：①求两个大向量的内积 ②128 核跨 2 台机器跑 WRF，各自该用哪个？
*考察点：共享内存 vs 消息传递；任务与机器规模的匹配。*

## B 动手题

**B1（首次点火）**：
```bash
mkdir -p ~/ex/omp && cd ~/ex/omp
cat > hello.c <<'EOF'
#include <omp.h>
#include <stdio.h>
int main() {
    #pragma omp parallel
    printf("thread %d of %d\n", omp_get_thread_num(), omp_get_num_threads());
    return 0;
}
EOF
gcc -fopenmp hello.c -o hello
./hello                     # 预测：打印几行？
OMP_NUM_THREADS=2 ./hello   # 预测：现在几行？
gcc hello.c -o hello_noomp && ./hello_noomp   # 预测：几行？为什么？
```
最后回答：第三种编译（忘 -fopenmp）为什么不报错也不警告？这埋着什么隐患？
*考察点：三支柱之指令与开关；静默串行陷阱。*

**B2（作用域手术）**：下面代码"1 线程跑结果对，4 线程跑每次都不一样"：
```c
double total = 0;
#pragma omp parallel for
for (int i = 0; i < 1000; i++) {
    double w = sqrt((double)i);   // 临时变量
    total += w * data[i];
}
```
指出 `total` 和 `w` 各自的正确作用域，写出修正版（提示：一个用子句显式声明，一个靠 reduction 顺带解决）。
*考察点：shared/private 判定；🕷 临时变量私有化。*

**B3（加速比实测——预讲作业正式收编）**：华科 sum_array 简化版：
```bash
cat > sum.c <<'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#define N 100000000
double a[N], b[N];
int main() {
    for (int i = 0; i < N; i++) { a[i] = 1.0; b[i] = 2.0; }
    double dot = 0;
    clock_t t0 = clock();
    for (long i = 0; i < N; i++) dot += a[i]*b[i];
    clock_t t1 = clock();
    printf("serial: %ld ms  dot=%f\n", (t1-t0)*1000/CLOCKS_PER_SEC, dot);

    dot = 0;
    t0 = clock();
    #pragma omp parallel for reduction(+:dot)
    for (long i = 0; i < N; i++) dot += a[i]*b[i];
    t1 = clock();
    printf("openmp: %ld ms  dot=%f\n", (t1-t0)*1000/CLOCKS_PER_SEC, dot);
    return 0;
}
EOF
gcc -O2 -fopenmp sum.c -o sum && OMP_NUM_THREADS=1 ./sum && OMP_NUM_THREADS=8 ./sum
```
任务：① 贴 1 线程和 8 线程两个耗时 ② 算加速比 ③ 回答：为什么不是 8 倍？（提示：N=1e8 双数组共 1.6GB，每次迭代 2 读 1 加——算术强度多低？瓶颈在 CPU 还是内存带宽？）
*考察点：memory-bound 认知——加速比封顶在带宽倍数；crawler 之前先想 roofline。*

**B4（调度实验）**：把 B3 的循环改成不均匀负载（如 `if (i % 100 == 0) w = heavy(); else w = 1;`），分别用 `schedule(static)`、`schedule(dynamic,1000)`、`schedule(guided)` 跑，记录三种耗时并解释排序。
*考察点：schedule 三兄弟的手感。*

## C 挑战题

**C1（π 积分）**：用梯形/矩形积分算 π：`π ≈ 4·Σ 1/(1+x_i²)·Δx`（x_i 取 [0,1] 均匀网格）。串行版写好后再 OpenMP 化，要求：① 1e-9 精度下与 math.h 的 M_PI 比较 ② 4 线程加速比报告 ③ 解释为什么这题比 B3 的加速比更接近核数（提示：算术强度）。
*考察点：并行数值积分标准范式；计算密集 vs 访存密集的加速比差异。*

**C2（伪共享，🕷 进阶）**：两个线程分别疯狂累加**相邻**的两个 long 变量：
```c
long c[2];   // c[0] 线程0 加 1 亿次，c[1] 线程1 加 1 亿次
#pragma omp parallel sections
{ c[0] 加加加;  c[1] 加加加; }   // 各自 critical 或 atomic
```
与"两变量相距很远（各开一个 64B 对齐的缓冲）"的版本比耗时，解释 **false sharing（伪共享）**：两变量无逻辑共享，却在同一条 64B 缓存行上，缓存一致性协议让两个核为它打架。
*考察点：缓存行粒度；性能问题可以毫无逻辑原因。*

**C3（考试预演：OceanSim 风格）**：给你一段热传导 stencil 计算（`new[i] = (old[i-1]+old[i+1])/2` 迭代 1000 轮）：
1. 哪一层循环能安全 `parallel for`？（提示：外层 i 有邻居依赖吗？内层时间步呢？）
2. 每轮迭代都开 parallel for 合算吗？（线程创建开销 vs 计算量）——给出两种改进思路
3. 若题目要求与串行版误差 <1e-6，`-ffast-math` 敢不敢开？为什么这题（纯加除二）比带 sin/cos 的题更敢？
*考察点：stencil 并行化位置选择；并行区域开销意识；-ffast-math 与正确性容差的关系。*
