# 模块 10 答案：图算法

## A 概念题

**A1** 
- n=1000, e=5000：矩阵 10⁶ 单元 ≈ 1MB（int），邻接表 ≈ n+2e = 11000 项。都装得下，但矩阵 99.5% 是零——**邻接表**（稀疏）
- n=500, e=100000：矩阵 250000 单元，但边有 2e=200000 条——**密度 80%，稠密图，矩阵合算**（O(1) 查边还白赚）
分界线思路：比较 2e 与 n² 的比值（边/位图密度）。

**A2** BFS 队列驱动：下一批 = 当前顶点的**全部未访问邻居**（先来先服务，FIFO）；DFS 栈驱动：下一站 = 当前顶点的**一个未访问邻居就扎进去**（最近发现优先，LIFO）。性格：BFS 是"荡开的洋葱"（逐层公平），DFS 是"一条道走到黑的探险家"（深钻回溯）。

**A3** BFS 首次到达某顶点时走过的层数 = 起点到它的跳数（队列保证按层处理，跳数小的层先处理完）——无权图"跳数最短 = 边数最短 = 最短路"。反例（3 顶点）：`0 --(权重100)--> 1`、`0 --(1)--> 2 --(1)--> 1`。BFS 给 0→1 的"最短路"是直达边 100，真最短是 0→2→1 = 2——**权重让"边少"不再等于"路短"**。

**A4** 不变量：顶点 u 被从堆中弹出（确定）时，`dist[u]` 已是从源到 u 的真实最短距离。论证用的是"u 的任何未确定路径都得先经过某个已确定点，而那个点的 dist 已 ≥ dist[u]"——**这条链路里每段边权 ≥ 0** 才能保证绕路不会更短。负权边击穿：绕路的负边能把总长拉低于 dist[u]，而 u 已经"确定"不再更新——静默出错。

**A5** Prim：加点——一棵树从种子长大，每步跨割拉最轻边。Kruskal：加边——森林里按边权升序收编，不成环就要。
① 稠密图（e≈n²/2）：Kruskal 要对 e log e ≈ n²log n 条边排序；Prim+堆 O(e log n) 同阶但**朴素 O(n²) 版更香**（无堆开销）→ Prim。
② 稀疏图（e≈n）：两边都是 O(e log n)，但 Kruskal 的排序开销小、实现简单 → **Kruskal**。

## B 动手题

**B1** 参考（BFS 为主，DFS 同理换容器）：
```cpp
vector<vector<int>> adj(n);
vector<int> dist(n, -1);
queue<int> q; q.push(0); dist[0] = 0;
while (!q.empty()) {
    int u = q.front(); q.pop();
    for (int v : adj[u]) if (dist[v] < 0) { dist[v] = dist[u]+1; q.push(v); }
}
```
测试图答案：BFS 序 `0 1 2 3 5 4`（或 0 2 1 3 5 4，邻居序决定）；跳数 dist = `0 1 1 2 3 2`。DFS 序（按邻居编号序递归）：`0 1 3 2 5 4`。注意 BFS/DFS 序**不唯一**（邻居访问顺序影响）——报告时说明自己的序约定。

**B2** 
① 三色 DFS：
```cpp
// 0=白(未访) 1=灰(在递归栈中) 2=黑(完成)
bool dfs(int u) {
    color[u] = 1;
    for (int v : adj[u]) {
        if (color[v] == 1) return true;       // 灰遇灰 → back edge → 有环
        if (color[v] == 0 && dfs(v)) return true;
    }
    color[u] = 2; return false;
}
```
② 并查集判环（无向）：逐边 `find(u)==find(v)` → 环；否则 union。
测试：`0→1→2→0` 版本 ① 报有环；`0-1,1-2` 版本 ② 报无环。

**B3** 要点：
```cpp
priority_queue 替代品：MinHeap 存 (dist, v)，按 dist 比较小顶
while (!heap.empty()) {
    auto [d, u] = heap.pop();
    if (done[u]) continue;          // 懒惰删除：旧版本跳过
    done[u] = true;
    for (auto [v, w] : adj[u])
        if (d + w < dist[v]) { dist[v] = d + w; heap.push({dist[v], v}); }
}
```
测试答案：dist = [0, 3, 1, 4, 7]（0→2→1→3→4：1+2+1+3 = 7）✓。push 次数可超 e——每次松弛都 push，旧项靠 done 跳过。

**B4** Kahn 骨架：
```cpp
vector<int> indeg(n, 0);
for (u→v 边) indeg[v]++;
queue<int> q; for (i) if (!indeg[i]) q.push(i);
int cnt = 0;
while (!q.empty()) {
    int u = q.front(); q.pop(); order.push_back(u); cnt++;
    for (int v : adj[u]) if (--indeg[v] == 0) q.push(v);
}
if (cnt < n) puts("IMPOSSIBLE");      // 剩下的都在环上
```
测试一：合法序如 `0 1 2 3 4 5`（1/0 可互换）；测试二加边 `4→0`：形成 0→2→3→4→0 环 → IMPOSSIBLE。

## C 挑战题

**C1** 
```cpp
struct DSU {
    vector<int> f, rk;
    int find(int x) { return f[x]==x ? x : f[x]=find(f[x]); }   // 路径压缩
    bool unite(int a, int b) {
        a = find(a); b = find(b);
        if (a == b) return false;                // 已连通 → 这条边会成环
        if (rk[a] < rk[b]) swap(a, b);
        f[b] = a; if (rk[a]==rk[b]) rk[a]++;
        return true;
    }
};
// Kruskal：sort 边 → 逐条 unite，收满 n-1 条
```
测试答案：取 1(0-1)、2(1-2)、4(2-3)，总权 7 ✓（3 和 5 被跳过：3 成环、5 也成环）。

**C2** 
```cpp
for (int k = 0; k < n; k++)
    for (int i = 0; i < n; i++)
        for (int j = 0; j < n; j++)
            d[i][j] = min(d[i][j], d[i][k] + d[k][j]);
```
k 外层的语义：第 k 轮结束时 `d[i][j]` = 只允许经 {0..k} 号中转点的最短路——**k 层结果依赖 k-1 层**，是标准 DP（d^k 由 d^(k-1) 推出）。k 放内层则某轮提前用上"中转点 k+1"的信息，语义崩坏（部分图会错）。

**C3** 
1. **CSR**（行偏移+列数组两个连续数组）：指针链的随机跳转变成顺序扫描——缓存预取友好，Graph500/HPCG 的标准输入格式
2. visited 用 `uint64_t` 位图：1/8 空间（bool 数组的 1/8）；更关键的是 parent 树可以省掉、位测试+置位一条指令——**带宽省 8 倍**
3. 不同顶点的邻居数悬殊（度分布长尾）→ 静态均分会让"大度数顶点的主人"拖死全队 → `schedule(dynamic, chunk)` 抢活制；跨节点则把图 2D 分区、边界顶点交换（04 模块 halo 交换的图版）。Graph500 的 TEPS 优化就这三件事在大规模上的极致化。
