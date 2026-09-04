# 模块 07：调试及性能分析（Debugging & Profiling）

> **来源**：MIT《The Missing Semester》2020 第 7 讲
> 中文：https://missing-semester-cn.github.io/2020/debugging-profiling/ ｜ 英文：https://missing.csail.mit.edu/2020/debugging-profiling/
> **超算刚需榜第一名**：比赛优化的标准起手式就是"profile 找热点 → 只优化热点"。HelloHPC 第 4、9 题明确要求用 perf 找瓶颈。

## 0. 一句话定位

两件事：**调试**（程序不对，找为什么）与**性能分析**（程序对但慢，找慢在哪）。共同纪律：**先测量，再动手；二分一切**。

## 1. 调试工具谱系（从糙到精）

### 1.1 打印调试（printf debugging）

Kernighan 名言：细致的分析配合恰当位置的打印语句是最有效的调试工具。糙快，但记得删。升级版是**日志**：

- 支持等级（DEBUG/INFO/WARNING/ERROR），可按严重度过滤
- 可写文件、发远端——出事后的第一现场往往就在已有日志里
- 系统日志：`/var/log/`（nginx 等）、`journalctl`（systemd）、`dmesg`（内核）、`logger "msg"` 可从命令行写入

### 1.2 调试器（debugger）：断点 + 单步 + 查变量

Python 用 **pdb**（增强版 ipdb），C/C++ 用 **gdb**/lldb。核心动作（两个语言同构）：

| pdb | gdb 等价 | 作用 |
|-----|----------|------|
| `l(ist)` | `list` | 显示源码附近 11 行 |
| `b(reak) 行号` | `break 文件:行` | 下断点 |
| `n(ext)` | `next` | 步过（函数当一步） |
| `s(tep)` | `step` | 步入函数内部 |
| `p 表达式` | `print 表达式` | 求值打印变量 |
| `r(eturn)` / `c(ontinue)` | `finish`/`continue` | 跑到返回/到下一断点 |
| `q(uit)` | `quit` | 退出 |

`python -m pdb script.py` 或代码里 `import pdb; pdb.set_trace()` 处停下。

### 1.3 不跑代码就能查：静态分析

**pyflakes/mypy**（Python）、**shellcheck**（bash——模块 02 的 `{echo` 案例它秒抓）。编辑器装 linter 插件（vim 的 ale），保存即查。风格/格式化：black、gofmt、prettier。

### 1.4 追踪类核武

- **strace**：追踪系统调用——程序"卡在哪个 syscall"一眼看穿（读文件？连网络？）
- **tcpdump / Wireshark**：网络层抓包分析

## 2. 性能分析（profiling）——本模块的重头

### 2.1 先会计时：real / user / sys

```bash
time ./myprog
# real 0m2.561s   墙上时间（含等待 IO/网络）
# user 0m0.015s   用户态 CPU
# sys  0m0.012s   内核态 CPU
```

**诊断逻辑**：real ≫ user+sys → 时间耗在**等待**（IO/网络/锁）；real ≈ user+sys 且大 → 纯**计算密集**（优化方向：算法/并行）；user 分散在多核时 user 总和可超 real（并行了）。

### 2.2 CPU 剖析器：追踪型 vs 采样型

```bash
python -m cProfile -s tottime script.py     # Python 函数级耗时排名
kernprof -l -v script.py                    # line_profiler：精确到每一行
```

追踪型（cProfile：记录每次函数调用）开销大但信息全；采样型（周期性看程序在哪：perf）开销小适合生产。

### 2.3 内存

C/C++：**valgrind**（查泄漏、越界）；Python：memory_profiler。比赛里"内存超限被杀"（HelloHPC 每题都有 1950M/核 限制）就靠它定位。

### 2.4 perf：Linux 性能剖析旗舰

```bash
perf list                       # 可用事件
perf stat ./myprog              # 统计：指令数、上下文切换、缓存缺失、页错误
perf record ./myprog            # 采样记录
perf report                     # 查看热点函数
```

perf 的杀手锏是**硬件计数器**：缓存命中率（cache-misses）、分支预测失败——这些是"程序慢但 CPU 没满载"类问题的答案。**火焰图**可视化（Y 轴=调用栈深度，X 轴=耗时占比，越宽越热）。

### 2.5 资源监控全家桶

| 场景 | 工具 |
|------|------|
| CPU/内存总览 | `htop`（top 增强版：F6 排序、t 树状进程） |
| 磁盘空间/目录 | `df -h`、`du -h 目录`、ncdu（交互版） |
| 磁盘 IO | iotop |
| 谁占着文件/端口 | `lsof` |
| 网络 | `ss`（替代 netstat）、iftop |
| 制造负载测试 | `stress -c 3` |
| **命令级基准对比** | `hyperfine 'cmd A' 'cmd B'`（预热+多次统计，fd 比 find 快 20 倍就是这么测的） |

## 3. 方法论（比工具更重要）

1. **先读完整报错**（我们实战里两次事故的破案起点都是"认真读报错"：`command not found` 的措辞暴露了 bash 在找程序）
2. **最小复现**：把问题收缩到最小可重复的例子
3. **二分**：注释一半代码/用 `git bisect` 二分提交/在中间打日志——任何"大范围"问题都能对数次数定位
4. **一次只改一个变量**：改两处再看结果，你不知道是哪个起效
5. **优化前必先 profile**：直觉的热点经常是错的；"不测量就优化"是 HPC 第一戒律

## 4. 常见误区清单

1. 拿到慢程序直接凭感觉改代码——先 profile，让数据指路
2. `time` 的 real 大就以为 CPU 慢——可能是等 IO（看 user/sys 分解）
3. printf 调试完忘删，或打印本身改变了时序（并发 bug 会"一打印就消失"，Heisenbug）
4. 只测一次就下结论——benchmark 必须**多次取均值**（hyperfine 默认就做）
5. 在登录节点跑重负载 profile（比赛规矩：profile 也要去计算节点）

## 5. 与超算比赛的联系

- HelloHPC 第 4 题（OceanSim）：题目原文提示"用 perf 找瓶颈"——这讲就是解题说明书
- 第 9 题（绳网模拟）：**不给热点**，自己 profile——完全就是本模块的期末考
- 优化的循环：`perf record → 找最热函数 → 只优化它 → 重新 record 验证`；不进这个循环的优化都是玄学
- HPL 调优同理：先 `perf stat` 看缓存缺失和内存带宽，再决定分块参数
