# 模块 08 练习：HPL/HPCG 基准

> B 卷可 WSL 完成：装 OpenMPI + 从源码编一个单机 HPL（gcc+OpenBLAS），跑小规模 HPL.dat 扫参——完整体验"编译→配参→跑分→读结果"。
> 先预测再动手；单变量纪律全程生效。

## A 概念题

**A1** HPL 与 HPCG 各测什么？为什么 HPCG 成绩通常只有 HPL 的 1-3%？从"算术强度/负载规则性"两个维度对比。
*考察点：双基准的设计哲学。*

**A2** HPL 的 `效率 = Rmax/Rpeak`。某机器 Rpeak=100 TFLOPS，实测 Rmax=75 TFLOPS：效率多少？列出三个"效率上不去"的常见嫌犯。
*考察点：效率概念；瓶颈候选清单。*

**A3** HPL.dat 四大参数 N/NB/P×Q 各控制什么？给出一条"调参顺序"并说明为什么这个顺序（哪个先固定、哪个后扫）。
*考察点：参数语义；单变量调参法。*

**A4** 为什么 N 要"内存占 80-90%"而不是 100% 或 50%？写出 N 的估算式（总内存 M 字节、双精度）。
*考察点：规模-内存权衡；N 估算公式。*

**A5** 换 BLAS 库为什么常常是 HPL 最大的单笔优化？Top500 前三名各用自己的数学库（oneMKL/BLIS）说明了什么行业事实？
*考察点：软件栈意识；厂商库与硬件的绑定关系。*

## B 动手题

**B1（单机 HPL 全流程，WSL）**：
```bash
sudo apt install openmpi-bin libopenmpi-dev libopenblas-dev
wget https://www.netlib.org/benchmark/hpl/hpl-2.3.tar.gz && tar xf hpl-2.3.tar.gz
# 按 hpl-2.3/INSTALL 复制 setup/Make.Linux_PII_FBLAS 为 Make.wsl，
# 改 ARCH=wsl、TOPdir、MPI_HOME、LAdir=/usr/lib/x86_64-linux-gnu、LA=-lopenblas
make arch=wsl
cd bin/wsl && mpirun --allow-run-as-root -np 4 ./xhpl   # root 下需该 flag
```
任务：① 跑通默认 HPL.dat（小 N）确认 PASS（看输出里 PASSED 校验行）② 读输出表的 Gflops 列 ③ 记录本机 Rpeak（核数×频率×8/16 FLOPs）算效率。
*考察点：从源码到跑分的完整工序——比赛第 8/10 题的通用形态。*

**B2（N 扫参）**：固定 NB=192、P×Q=1×4，扫 N ∈ {2048, 4096, 8192, 12288, 16384}（按内存估算上限酌减）：
1. 填表：N × Gflops
2. 验证"80-90% 内存甜区"：哪个 N 附近到峰值？再往上（如果内存撑得住）为什么反而降？
*考察点：规模曲线的手感；甜区的实证。*

**B3（NB 与网格扫参）**：固定 B2 的最优 N：① 扫 NB ∈ {64, 128, 192, 256}；② 固定最优 NB 扫 P×Q ∈ {1×4, 2×2, 4×1}。填两张表，回答：NB 的最优值为什么和 BLAS 库/缓存大小有关？P×Q 的方形偏好从哪来？
*考察点：两轮扫参的标准流程；参数背后的硬件理由。*

**B4（正确性红线）**：故意把 HPL.dat 的某个参数改坏（如 N 非 NB 倍数、或换 -llapack 不接 BLAS），观察输出（FAIL/段错误/残差超标）。回答：HPL 输出里"正确性通过"的标志是什么行？为什么比赛里"PASS 但残差偏高"也要警惕？
*考察点：先对再快；残差语义。*

## C 挑战题

**C1（HPCG 试跑，⚡ 可选/纸面亦可）**：官网获取 HPCG 源码，MPI+C++ 编译跑 `--nx=104 --ny=104 --nz=104 --rt=60`（若环境不允许，纸面回答）：① HPCG 的三个核心 kernel 是什么？② 它的优化为什么"调 HPL.dat 那套不管用"？③ 你的机器上 HPCG GFLOPS 与 HPL GFLOPS 比值是多少？这个比值说明机器的什么特征？
*考察点：双基准的实操对照；"真实负载利用率"的意义。*

**C2（功耗墙规划，ASC 决赛形态）**：节点 = 2×CPU（各 64 核，TDP 280W）+ 4×GPU（各 300W），整机散热预算 3000W：
1. 全开功耗多少？超支多少？
2. 给出两种合规方案（限频/砍 GPU 数）及其算力代价的定性对比
3. （口答）HPL 与 AI 赛题哪个对功耗更敏感？为什么方案要留 10% 余量？
*考察点：性能/瓦特优化——决赛的元问题。*

**C3（考试预演：Graph500 脚本题）**：对照 HelloHPC 第 8 题：写 graph.sh——module load 环境 → make → 用 128 核 mpirun 跑 `./graph500_reference_bfs_sssp 18` → 不许重定向输出 → 40min 内完成。回答：
1. 为什么题面禁止输出重定向？
2. TEPS 上不去时，按本章方法论列 3 个排查动作（提示：编译选项/运行参数(scale)/并行方式）
3. 这个脚本与 HPL 的 HPL.dat 扫参在"调优形态"上有何异同？
*考察点：基准题的答题骨架——比赛真题第 8 题的完整预演。*
