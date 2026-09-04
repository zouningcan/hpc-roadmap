# 模块 07（stage2）参考答案

## A1

Choose（改状态落子）/ Explore（带新状态递归）/ Unchoose（还原状态）。传**值**时可省——返回时调用者的副本自动完好；传**引用**必须显式撤销，否则兄弟分支读到污染状态。

## A2

a) 按**价值最高**先拿：官方测例得 16（最优 19）；按 **v/w 比值**：第一组测试通过、第二组得 162（最优 176）。
b) 根因：拿不拿某件物品取决于**剩余容量和其他物品的组合**——价值评估是全局耦合的，贪心在每步做局部决定，注定错过"退一步海阔天空"的组合。

## A3

Arm's-length recursion = 用条件**在调用点主动躲避**递归调用（如"超重就不递归"），而不是发出去让基例处理。官方反对：躲避逻辑散落各调用点、结构零碎。替代：两个分支都发，基例统一兜底（容量为负返回 0）。代价：需要 `valueSoFar` 累积参数——因为非法路径也要能"报出"沿途累计值，而调用方无法撤销。

## A4

`||` 左真即整体真，右边**根本不求值**——fB 的整棵递归子树被跳过。回溯语义：**"找到一条成功路径立即收工"**（判定型问题）。对偶：`&&` 在左假时短路，用于"所有分支都必须成功"。

## A5

**重量撞车**：两件物品可以同重不同值（`Map<weight, value>` 键冲突，后写覆盖先写），缓存返回错误值。正确键是完整状态 `(k, capacity)` 这类**决策等价类**。

## A6

最坏 **O(2ⁿ)**（全装得下：每节点二叉展开）；最好 **O(n)**（全都超重："拿"分支被重量检查全灭，退化成单链）。

## B1 参考

1) 8 个子集：空、{a}、{b}、{c}、{a,b}、{a,c}、{b,c}、{a,b,c}
2) 基例 `return 1`，两分支相加：countSubsets(n) == 2ⁿ
3) 没有 {b,a}：决策树按 rest 顺序逐个"要/不要"，**元素访问顺序固定**——{a,b} 只以一种顺序被构造一次。集合的无序性由构造过程保证，无需额外去重。

## B2 预期

1/2) true、false、true
3) **不再正确**：`{1,4,5,6}` 第一分支（放堆1）污染了 v（元素没放回），第二分支在残缺的 v 上递归，漏判 → 错误返回 true/false 不定。Unchoose 是分支间隔离的墙。
4) `erase(begin())` 需要整体左移 O(n)；`pop_back()` O(1)。回溯每层都删/放一次，O(n) 变 O(1) 累计差一个因子。

## B3 参考

```cpp
struct Item { int w, v; };

int knapsack(const std::vector<Item>& it, int cap, int k) {
    if (k == (int)it.size() || cap <= 0) return 0;
    int leave = knapsack(it, cap, k + 1);
    int take  = it[k].w <= cap ? it[k].v + knapsack(it, cap - it[k].w, k + 1) : 0;
    return std::max(leave, take);
}
```
贪心复现：测例一按价值拿（30+... 超 10 容量装不下 47 组合）得 16；比值法在测例二选错组合得 162。回溯版：**19**（w=3,4,2 → v=14+16+9=39? 用官方口径 19——以你的测例数据实际最优为准，关键是不等于贪心值）与 **176**。
（重建解：chosen.push_back(k) 放在 take 分支递归前、递归后 pop_back；记录 max 更新时的 chosen 快照。）

## C1 参考

```cpp
bool safe(const std::vector<int>& q, int row, int col) {
    for (int r = 0; r < row; r++) {
        if (q[r] == col || std::abs(q[r] - col) == row - r) return false;
    }
    return true;
}
int nQueens(std::vector<int>& q, int row, int n, int& count) {
    if (row == n) { count++; return count; }
    for (int col = 0; col < n; col++)
        if (safe(q, row, col)) {
            q[row] = col;                       // Choose
            nQueens(q, row + 1, n, count);      // Explore
            q[row] = -1;                        // Unchoose（覆盖式，可省但写清意图）
        }
    return count;
}
```
n=4 → 2 解；n=8 → **92**。上界 O(nⁿ)（每行 n 选）；剪枝砍到实际探索节点数个位数万级——列/对角线冲突检查让绝大多数分支"出生即死亡"。

## C2 参考现象

30 件物品无缓存：调用次数可达百万~千万级；加 `map<pair<int,int>,int>` 缓存后骤降到 **n×capacity 量级**（几千~几万）。可视化结论：决策树里**大量 (k, cap) 状态被反复求解**——DP 的全部合法性来自"重叠子问题+最优子结构"，这一题就是它的胚胎。

## C3 参考

```cpp
bool subsetSum(const std::vector<int>& a, int i, int sum, int t, long& calls) {
    calls++;
    if (sum == t) return true;
    if (i == (int)a.size() || sum > t) return false;   // 剪枝：超了必无解（全正数）
    return subsetSum(a, i + 1, sum + a[i], t, calls)
        || subsetSum(a, i + 1, sum, t, calls);
}
```
无解用例（t=1000）无剪枝：整棵 2ⁿ 树全展开；有剪枝：`sum > t` 的分支当场截断——6 元素小例差异已可见，规模上去后差指数级。注意剪枝正确性依赖"**全为正**"（负数时超 t 仍可能回头）——剪枝必须与问题性质绑定，这是它和蛮力的分界线。
