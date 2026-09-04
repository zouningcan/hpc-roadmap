# 模块 07（stage2）：递归回溯

> **来源**：Stanford CS106B 2026 夏 L11《Recursive Backtracking and Enumeration》+ L12《More Recursive Backtracking》
> `.../lectures/11-backtracking1/` `12-backtracking2/`
> 模块 06 的"改动-递归-撤销"三步曲在本模块获得正式名字和完整形态。

## 0. 一句话定位

回溯 = **在决策树上深度优先地试错**：每个决策点选一步往下走，撞死胡同就退回来换一步。它是生成所有可能（枚举）和寻找最优（优化）的万能引擎。

## 1. 核心范式：Choose – Explore – Unchoose

| 步骤 | 动作 | 迷宫例 |
|------|------|--------|
| **Choose** | 改状态，落一子/走一步 | 迷宫里前进一格、撒面包屑 |
| **Explore** | 带新状态递归 | 从新位置继续探索 |
| **Unchoose** | **撤销改动**，换下一步 | 退回并拾回面包屑 |

🕷 **面包屑防死循环**：迷宫里走过的格子做标记，否则绕圈导致无限递归——这是"重复状态检测"的最朴素形态。

**Unchoose 何时可省**：数据**传值**时（函数返回自动恢复调用者的副本）；**传引用**时必须显式撤销。两种写法都能对，但混用必错。

## 2. 回溯算法的解剖学（官方通用结构）

```
求解(状态):
    基例: 检查是否成功 → 打印/返回值
    （可选）重复状态检查 → 已见过则返回
    for 每个 可能的走法:
        校验走法（边界、合法性）
        Choose（改状态）
        Explore（递归）
        （处理返回值）
        Unchoose（还原状态）
```

## 3. 枚举样本：printSubsets——soFar/rest 双累积器

生成全部 2ⁿ 个子集（幂集）：

```cpp
void printSubsets(std::string soFar, std::string rest) {
    if (rest.empty()) { std::cout << soFar << "\n"; return; }
    printSubsets(soFar, rest.substr(1));              // 不要首元素
    printSubsets(soFar + rest[0], rest.substr(1));    // 要首元素
}
```

- 每个元素一个**二元决策**（要/不要）→ 决策树 2ⁿ 个叶子
- 🕷 **为什么没有 {b,a}？** 集合无序——{a,b} 与 {b,a} 是同一个，决策树按元素顺序天然无重复
- **计数版改造**（官方标注重要）：基例 return 1，两个递归调用相加——"打印器"变"计数器"只动基例和 return
- 效率注：传值 string 每次 O(n) 拷贝 × 2ⁿ 调用；传引用 + 下标版本零拷贝但要手动 add/remove 记账（面试高频变体）

## 4. 判定样本：等和分区（partition）

问题：`{1,1,2,3,5}` 能否分成和相等的两堆（{1,5} 与 {1,2,3} ✓）；`{1,4,5,6}` ✗。

**和追踪版**：

```cpp
bool isPartitionable(std::vector<int>& v, int sum1, int sum2) {
    if (v.empty()) return sum1 == sum2;
    int x = v.back(); v.pop_back();                  // Choose（从尾部删更便宜！）
    bool ok = isPartitionable(v, sum1 + x, sum2)     // Explore：放堆1
           || isPartitionable(v, sum1, sum2 + x);    //      或放堆2
    v.push_back(x);                                  // Unchoose（关键！）
    return ok;
}
```

🕷 三个官方洞见：

1. **`||` 短路 = 天然剪枝**：左边为真右边整个不跑——"第一个成功路径找到就收工"的惯用法（`&&` 对偶短路假）
2. **去掉 `v.push_back(x)` 会怎样**：状态被污染，`{1,4,5,6}` 这类"第一分支走不通"的用例给出错误答案——亲测见练习
3. **从尾部删除**：`erase(begin())` 要整体搬移 O(n)，`pop_back()` O(1)——模块 02 的 338× 在这里的微观应用

## 5. 优化样本：0-1 背包（L12 的主角）

容量 c，每件物品有重量 wᵢ 和价值 vᵢ，**拿或不拿**（0-1），求不超过容量的最大价值。

### 🕷 先看贪心怎么死（官方两次翻车实录）

| 贪心策略 | 测例结果 |
|---|---|
| 先拿**价值最高**的 | 16（最优 19）✗ |
| 按 **v/w 比值**排序拿 | 一测例过，另一测例 162（最优 176）✗ |

教训：**局部最优拼不出全局最优**——每件物品的价值取决于"还剩多少容量"，这是耦合决策，贪心解耦就错。

### 三种正确写法（官方故意展示三种风格）

**解法一：传引用 + 删尾 + 放回**（同 partition 骨架，重量检查挡住"拿"分支，`max(拿, 不拿)` 取优）

**解法二：下标 k**（不改动 vector）：

```cpp
int knapsack(const std::vector<Item>& items, int capacity, int k) {
    if (k == (int)items.size()) return 0;            // 决策完毕
    int bestLeave = knapsack(items, capacity, k + 1);
    int bestTake = items[k].w <= capacity
        ? items[k].v + knapsack(items, capacity - items[k].w, k + 1) : 0;
    return std::max(bestLeave, bestTake);
}
```

`k` 的**双重读法**（官方原文的精彩处）："考虑第 k 件物品" 或 "前 k 件的决策已做完"。

**解法三：拒绝"arm's-length recursion"（隔臂递归）**——**两个调用都无条件发**，让基例去处理非法路径（容量为负 → 0）。官方定义：用条件**主动躲避**递归调用，而不是调用后让基例兜底，就是 arm's-length recursion——躲避逻辑散落在调用点，不如集中到基例清晰。代价：需要第三个参数 `valueSoFar`（调用方无法"撤销"已累加的值）。

### 复杂度

最坏（全装得下）：每节点两分支 **O(2ⁿ)**；最好（全都太重）：一线到底 O(n)。预告：**记忆化/动态规划**能消掉重复子调用（后续课程/学有余力）。

## 6. 结构体初见（L12 前菜）

```cpp
struct TreasureT {          // T 后缀是"类型名"风格
    int weight;
    int value;
};                          // 🕷 结尾分号必须有！
```

坑：忘分号报错位置离谱；struct 声明放函数**外**（全局），别塞进函数里。

## 7. 常见误区清单

1. 贪心当回溯用——背包两种贪心都翻车，**只有穷举+剪枝保证最优**
2. 传引用版忘 Unchoose——兄弟分支读到污染状态
3. 忘撒面包屑——迷宫/图类无限递归
4. 从 `begin()` 删元素——O(n) 搬移；能从尾部操作就尾部
5. struct 忘结尾分号
6. 把判定逻辑写在调用点（arm's-length）而不是基例——散乱且难维护
7. 用 `Map<weight, value>` 缓存——**重量会撞车**（两件同重不同值）

## 8. 与超算比赛的联系

- 回溯 = **带剪枝的暴力搜索**：参数空间扫描、编译器组合优化（-O 序列）、任务分配方案探索的底层方法
- **剪枝是 HPC 的精神**：短路的 `||`、重量检查、面包屑——每剪掉一支子树省指数级时间，和"先对再快"同源
- 0-1 背包的记忆化预告 = DP：HelloHPC 第 5 题（Calculation）的内层 sin/cos 循环用"预计算表"换重复计算，同一思想
- struct 打包数据（weight/value）→ 模块 09 OOP 的前菜：比赛里把赛题参数打包成 struct 是标准工程动作
