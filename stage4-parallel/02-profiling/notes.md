# 模块 02：性能剖析（profiling）

> **来源**：[perf 官方 tutorial](https://perfwiki.github.io/main/tutorial/)；[Brendan Gregg perf 示例集](https://www.brendangregg.com/perf.html)（火焰图发明者）；[MIT 6.172 Performance Engineering 讲义](https://ocw.mit.edu/courses/6-172-performance-engineering-of-software-systems-fall-2018/pages/lecture-slides/)
> 比赛呼应：HelloHPC OceanSim 题（"用 perf 找瓶颈"写在题面）、RopeNet 题（"不告诉你热点在哪，自己剖析"）；原则：**先测量再优化**（profiling before optimizing）

## 0. 一句话定位

剖析（profiling）= **让程序自己交代时间花在哪**。没有剖析的优化 = 闭眼开药——90% 的直觉热点是错的，真实热点往往在你没想到的那行。

## 1. 两大门派

| 门派 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| **采样**（sampling） | 定时"抓拍"程序计数器（如每 1ms 看一眼在哪个函数） | 近零开销、无需改代码、真实负载 | 统计近似（样本少有小函数漏网） |
| **插桩**（instrumentation） | 在函数出入口塞计时代码/gcov 编译 | 精确到调用次数 | 开销大、扭曲时序 |

perf 是采样派（内核 perf_event 子系统），gprof 是插桩派（-pg 编译）。**现代默认：perf**。

## 2. perf 三件套（必会）

### 2.1 `perf stat`——总分账（硬件计数器）

```bash
perf stat ./oceansim
# 关键输出：
#   1,234,567,890  cycles          # 总周期
#     987,654,321  instructions   # 指令数
#       0.80       insn per cycle (IPC)   # 每周期指令数——CPU 忙不忙的心跳
#       12.3%      cache-misses    # 缓存未命中率——访存是不是瓶颈
#       3.2%       branch-misses   # 分支预测失败率
```

读法：**IPC 低 + cache-miss 高 = memory-bound**（IPC 低 + branch-miss 高 = 分支发散）——roofline 的 CPU 计数器版。

### 2.2 `perf record` + `perf report`——热点账单

```bash
perf record -g ./oceansim        # -g 连调用栈一起采样
perf report                      # 交互式热点排行（按采样占比）
```

输出示例（OceanSim 式症状）：

```
  52.3%  main  OceanSim.cpp:87     ← 一半时间在这一行（某内层循环）
   8.1%  sin@plt                   ← 数学库函数
   ...
```

**热点排行看占比不看绝对值**——52% 的函数优化掉一半 = 全程序快 26%。

### 2.3 火焰图（flame graph）——调用栈的可视化

```bash
perf record -g -F 99 ./prog && perf script > out.stacks
# stackcollapse-perf.pl + flamegraph.pl（github.com/brendangregg/FlameGraph）
# 或现代版：perf report --stdio；hotspot/Nightly 版有 GUI
```

读法：**x 轴 = 字母序的栈（不是时间），宽度 = 采样占比，y 轴 = 调用深度**——最宽的塔就是最贵的调用链，一眼定位"谁调用了谁花掉了时间"。

## 3. 采样的底层：perf_event 与周期

`-F 99`（99 Hz）意味着每秒抓 99 张"现场快照"——**频率别设太高**（开销/扰动），经验 99-997 Hz。除了时间采样，还能按**事件**采样：`perf record -e cache-misses ./prog`——专抓缓存未命中的现场，直接指向访存最痛的指令。

**时间样本要够多**：程序跑 0.1 秒采样 10 个点没有意义——先让 benchmark 跑足够久（或循环多次）。

## 4. 其他工具速览（知道何时用谁）

| 工具 | 场景 |
|------|------|
| `gprof`（-pg 编译） | 教学老工具：函数级调用图+耗时；开销大，现代少用 |
| `valgrind --tool=callgrind` | 精确指令级调用计数（仿真执行，慢 20-100 倍）——小函数计数审计 |
| `perf c2c` | 专查**伪共享**（哪个缓存行被多核打架——03 模块 C2 的取证工具） |
| `htop` | 先看满没满核（单核 100% = 并行没生效） |
| **Nsight Systems / Compute**（NVIDIA） | GPU 版：Systems 看时间线（kernel 间空隙），Compute 看单 kernel 内部——模块 05/07 的搭档 |

## 5. 剖析驱动的优化流程（OceanSim 实战剧本）

```
① perf stat：IPC 高吗？cache-miss 高吗？→ 判定 compute/memory-bound
② perf record/report -g：找到 top1 热点（常是一个内层循环或数学库调用）
③ 对照源码分类热点：
   - 循环内重算不变量（sin/cos 常量）→ 提取/查表（05 Calculation 题型）
   - 跨步访存（列遍历）→ 循环交换/分块改连续访存
   - 冗余函数调用 → 内联/手写
   - 并行没生效（单核满）→ 查 -fopenmp/线程数（03 模块）
④ 改一处 → 重测 → 重复 ①（单一变量原则！一次只改一件事）
⑤ 收益递减时停下：先把对的写对，再谈快
```

🕷 三大剖析纪律：**① 优化前必留基准数据（baseline）**，否则"快了"无从证明；② **一次只改一个变量**（开 -O3 的同时又改循环，功劳算谁的？）；③ **优化后重跑正确性校验**——快的错误答案一文不值（HelloHPC 打表禁令的深层原因）。

## 6. 常见误区清单

1. **没剖析就优化**：凭直觉改"看起来慢"的代码——热点排行会教育你
2. 只跑一次采样就下结论：样本少 + 系统噪声（其他进程）——多跑取中位，关后台程序
3. Debug 版测性能（-O0）：优化器会把热点搬来搬去，剖析结果无意义——**剖析永远用 -O2/-O3 的发布版**
4. 只看 self 占比不看调用栈：叶子函数便宜但被调一亿次——-g 看全栈才知道"谁干的"
5. 采样频率拉满求精确：997Hz 以上开销扭曲程序本身——够了就行
6. 优化后不重跑正确性校验：快的错误答案 = 零分（OceanSim 的 1e-6 容差就是裁判）

## 7. 与超算比赛的联系

- **OceanSim（4 核）**：题面直接点名 perf——流程就是本章第 5 节
- **RopeNet（128 核）**：hint"没有清晰地给出计算热点，请根据常识或 profiling 手段自行判断"——多线程剖析（perf -g 能按线程采样；Nsight Systems 看时间线）
- **Calculation（8 核）**：剖析发现 sin(p)*cos(p) 内层循环与 i 无关 → 提取常量 = 一行代码 30 倍提速的出处
- **纪律即分数**：writeup 里"我用 perf stat 定位到 memory-bound，perf report 显示 60% 在列遍历，循环交换后 cache-miss 从 30% 降到 5%"——这段话就是满分 writeup 的骨架
