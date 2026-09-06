# 模块 06：进阶优化（SIMD/NEON、访存、编译器）

> **来源**：ARM NEON intrinsics（[Arm NEON 指南](https://developer.arm.com/documentation)与 rogerou/Arm-neon-intrinsics 速查）；鲲鹏平台文档；[Compiler Explorer](https://godbolt.org/)（看汇编的标配）
> 对齐：HelloHPC 的 ARM 鲲鹏（aarch64 + NEON）实战环境；模块 03 优化阶梯 L2（`parallel for simd`）的底层展开
> 前置：模块 02 剖析（先找到热点再优化）、模块 03（OpenMP+SIMD 组合）

## 0. 一句话定位

单线程已经最优、OpenMP 也加了，还想快——**指令级**（SIMD 向量化）、**微架构级**（缓存/分支/对齐）、**编译器级**（flag/指令内联）三层榨干最后一滴。HelloHPC 的位运算题（Masked Bitmatrix，1 核）就是这三层的舞台。

## 1. SIMD：一条指令算一批

SIMD（Single Instruction Multiple Data）= 向量寄存器装多个数据、一条指令同时算：

| 平台 | 向量寄存器 | 宽度 | 每条指令 int32 通道数 |
|------|-----------|------|---------------------|
| x86 SSE/AVX2/AVX-512 | xmm/ymm/zmm | 128/256/512b | 4/8/16 |
| **ARM NEON（鲲鹏）** | q0-q31 | **128b 固定** | 4（int32/float32） |

```
NEON 例：128 位向量 = 4 个 int32
     [ a0 a1 a2 a3 ]   +   [ b0 b1 b2 b3 ]
  =  [ a0+b0 a1+b1 a2+b2 a3+b3 ]     ← vaddq_s32 一条指令
```

**Amdahl 的向量版**：向量化 4 倍的是"被向量化的那部分"，整体加速 = 1/((1-p) + p/4)——**热点占比 p 决定收益**（剖析又是爹）。

## 2. NEON intrinsics 速成（鲲鹏实战款）

**intrinsic = 编译器内置函数**：像调函数，实际生成单条向量指令（比裸汇友好，比自动向量化可控）。命名拆解：`v` + 操作 + 类型 + `q`(128b) + `_s32/_f32`（元素类型）：

```c
#include <arm_neon.h>
// 向量加
int32x4_t va = vld1q_s32(&a[i]);          // 装载 4 个 int32
int32x4_t vb = vld1q_s32(&b[i]);
int32x4_t vc = vaddq_s32(va, vb);          // 4 路加
vst1q_s32(&c[i], vc);                      // 存回
// 常用族：vaddq/vsubq/vmulq（算术）、vmlaq（乘加）、vabsq、vminq/vmaxq、
//         vshlq（移位）、vandq/vorrq/veorq（位运算！Bitmatrix 题的主力）、vld1q/vst1q（IO）
```

**尾处理**：n 不是 4 的倍数——主体向量化 + 尾巴标量循环（ peel loop），或者题目保证对齐（Bitmatrix 的 N 是 64 的倍数——出题人已经替你想好了）。

**水平方向归约**：向量算完 `vaddvq_s32`（4 通道加成一个）收尾——SIMD 加得快、合并要一步。

## 3. 访存优化三板斧（与 GPU 合并访存同宗）

1. **连续访问**：行主序遍历、循环交换（02 模块 C1 的处方）
2. **分块（blocking/tiling）**：让工作集住进 L1/L2——矩阵乘的 ikj 循环序或 tile 版
3. **预取与对齐**：`__builtin_prefetch`（手动预取，谨慎）；数据结构对齐 `alignas(64)`（防伪共享/防跨行）

**缓存行意识**：64B 一行——顺序扫描 = 每行 64 字节白赚；随机访问 = 每行用 4 字节亏 60B。

## 4. 编译器：你的第一个优化器

```bash
-O2（安全基线）→ -O3（激进：更彻底向量化/内联）
-march=native（让编译器用上本机全部指令集——鲲鹏上 -march=armv8.2-a+simd）
-ffast-math（数学重排/近似——有容差才敢开，03/02 模块反复强调）
-funroll-loops（循环展开）
-flto（链接时优化：跨文件内联）
```

**Compiler Explorer（godbolt.org）工作流**：左边贴 C++，右边实时看汇编——**验证"编译器到底有没有向量化"的唯一铁证**（找 `vaddq` 等向量指令 / x86 找 `vaddps`）。`-fopt-info-vec`（gcc）也能报告向量化成败与原因。

🕷 **自动向量化失败 Top3**：① 循环内有依赖（`a[i] += a[i-1]`）② 指针可能重叠（加 `__restrict` 告诉编译器"不别名"）③ 条件分支过复杂。三招：重构循环、`__restrict`、分支改查表/算术。

## 5. 位运算技巧箱（Bitmatrix 题弹药库）

```c
x >> k & 1              // 取第 k 位
x & (x-1)               // 清最低位的 1
__builtin_popcountll(x) // 1 的个数（NEON 有向量版 vcntq）——按位统计的核心！
x & -x                  // 最低位的 1（隔离）
^ 异或                  // 无进位加/校验/去重（m4c 的 checksum）
```

**查表法**：一次算 8 位或 16 位的答案存表，运行时 O(1) 查——"算术换访存"的边界要拿 perf 量（表若被缓存挤出去反而慢）。

## 6. 常见误区清单

1. 无脑手写 NEON：先看**编译器自动向量化**做到哪（-O3 + -march），手写只在编译器失败/差最后一口时上——可读性是真成本
2. 向量化了访存没跟上：算得快喂不上（memory-bound 病人向量化收益≈0）——先 perf stat 判型
3. NEON 尾循环忘写：结果错 n mod 4 个元素——边界测试（n=1,3,4,5）必做
4. `-ffast-math` 当免费午餐：NaN/Inf 语义破坏 + 重排误差——有容差的题才能开
5. `__restrict` 滥用：编译器信了你的话，指针真重叠时结果错——担保要真实
6. 微优化（位技巧/手工展开）在热点外的地方写：白费+可读性税——**一切听剖析的**

## 7. 与超算比赛的联系

- **Masked Bitmatrix（1 核 120 分）**：位矩阵按列统计——`popcount` 族 + NEON 向量化 + 转置布局是满分的台阶（题面点名"位运算知识、向量化"）
- **鲲鹏平台**：所有提交在 aarch64 上编译——`module load bisheng`（毕昇编译器，LLVM 系，NEON 支持好）；x86 本地写的 AVX intrinsics **不通用**，便携写法要么 auto-vectorize 要么宏分流
- **m4c**：异或校验/去重的字节操作可用 NEON veorq 批量做
- **CSE234/GPU 的 SIMD 血统**：warp = 32 线程锁步 = 超大 SIMD——SIMD 思想从 CPU 向量寄存器一路贯穿到 GPU，学一次受用三层
