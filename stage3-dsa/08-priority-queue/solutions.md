# 模块 08 答案：优先级队列与堆

## A 概念题

**A1** 
| 实现 | insert | delMax | 
|------|--------|--------|
| 无序向量 | O(1) | O(n) |
| 有序向量 | O(n) | O(1) |
| 堆 | O(log n) | O(log n) |
折中的本质：堆用"树高"级别的代价，把两个操作绑在同一复杂度上——**没有短板的操作**胜过有长板也有短板的组合（假设两操作同等频繁）。

**A2** 
- 逐个 insert（上滤）：`[9, 5, 8, 4, 3, 2, 7]`？——动手验证：插 4→[4]；插 7 上滤→[7,4]；插 2→[7,4,2]；插 9 上滤（7<9）→[9,4,2,7]；插 5（4<5 上滤）→[9,5,2,7,4]；插 8（2<8 上滤，9>8 停）→[9,5,8,7,4,2]。
- Floyd：从末个内部节点（下标 2）倒序下滤。初数组 [4,7,2,9,5,8]：下标 2（值 2）下滤 → 与 8 换 → [4,7,8,9,5,2]；下标 1（值 7）：孩子 9,5，与 9 换 → [4,9,8,7,5,2]；下标 0（值 4）：孩子 9,8，与 9 换，继续：孩子 7,5，与 7 换 → [9,7,8,4,5,2]。
- 结果不同！**但都满足堆序、都合法**——建堆结果不唯一。复杂度：前者 O(n log n)，后者 O(n)。

**A3** O(n)（只能全扫）。位置：**所有叶子**（数组后半段 n/2 到 n-1）——叶子没有孩子，堆序对它无约束；最底层的左侧先到（完全二叉树性质）。

**A4** NPL（null path length）：节点到"最近的外部节点（空孩子）"的距离，空节点 NPL=-1，叶子=0。左式堆不变量：**任一节点左孩子 NPL ≥ 右孩子 NPL**——右侧路径永远更短，于是 merge 沿右脊（全右链）走，右脊长 O(log n)，合并 O(log n)。"左倾"指**树整体向左歪、右边瘦**（直观上左边沉甸甸，右边是"快速通道"）。

## B 动手题

**B1** 参考实现（核心部分）：
```cpp
struct MaxHeap {
    vector<int> a;
    void up(int i) {
        while (i > 0 && a[(i-1)/2] < a[i]) { swap(a[(i-1)/2], a[i]); i = (i-1)/2; }
    }
    void down(int i) {
        int n = a.size();
        while (true) {
            int l = 2*i+1, r = l+1, big = i;
            if (l < n && a[l] > a[big]) big = l;
            if (r < n && a[r] > a[big]) big = r;
            if (big == i) break;
            swap(a[i], a[big]); i = big;
        }
    }
    void push(int x) { a.push_back(x); up(a.size()-1); }
    int top() { return a[0]; }
    void pop() { a[0] = a.back(); a.pop_back(); down(0); }
};
```
出堆序列验证：9 8 5 3 2 1 ✓（每次都是当前最大）。

**B2** 参考量级（数值随机器浮动，趋势是关键）：
| n | 逐个 push | Floyd | 比值 |
|---|-----------|-------|------|
| 1e5 | ~15 ms | ~2 ms | ~7x |
| 1e6 | ~200 ms | ~25 ms | ~8x |
| 1e7 | ~2600 ms | ~300 ms | ~9x |
比值 ≈ log n 的量级差（n=1e7 时 log n≈23），且**比值随 n 缓慢增大**（log n 增长）——正是 O(n log n)/O(n) 的实测脸孔。

**B3** 
```cpp
void heapSort(vector<int>& v) {
    // Floyd 建堆（就地）：从最后内部节点倒序下滤
    int n = v.size();
    for (int i = n/2 - 1; i >= 0; i--) down(v, i, n);   // down 带 n 参数
    for (int end = n-1; end > 0; end--) {
        swap(v[0], v[end]);       // 当前最大沉底
        down(v, 0, end);          // 在缩小的前缀内下滤
    }
}
```
升序 ✓。原地 O(n log n)：堆排序 = 建堆 + n-1 次"delMax 放到尾部"——`delMax` 与排序的一体两面。

**B4** **小顶堆**：TopK 大的一百个，堆顶是"这 100 个里最小的"——新数来了与堆顶比：比堆顶大才进（弹堆顶），比堆顶小直接扔。若用大顶堆，堆顶是 100 个里最大的，新数"比最大还小"未必该进堆，无法只 O(log k) 判定。复杂度 O(n log k)（每个数最多一次上/下滤，k=100），空间 O(k)。经验：**最大的 K 个用小顶堆（守门员是门槛），最小的 K 个用大顶堆**。

## C 挑战题

**C1** 递归 merge 骨架：
```cpp
Node* merge(Node* h1, Node* h2) {
    if (!h1) return h2;  if (!h2) return h1;
    if (h1->val < h2->val) swap(h1, h2);      // h1 为大顶根
    h1->r = merge(h1->r, h2);                 // 沿右脊合并进右子
    if (npl(h1->l) < npl(h1->r)) swap(h1->l, h1->r);  // 恢复左倾不变量
    h1->npl = npl(h1->r) + 1;
    return h1;
}
Node* push(H& h, int v) { return merge(h.root, new Node(v)); }
Node* pop(H& h)  { Node* r = h.root; h.root = merge(r->l, r->r); return r; }
```
手画验证要点：合并后每个节点的左 NPL ≥ 右 NPL，根 NPL = 右脊长 +1；右脊上不满足就 swap 左右——不变量修复是"回溯时顺手做"的。

**C2** 骨架：
```cpp
struct Event { double t; int a, b; };   // 大顶堆按 -t 建或自定义比较（小顶时间）
// 初始化：每对粒子 (i,j) 算首次碰撞时刻 push
while (!heap.empty() && sim_time < T_END) {
    Event e = heap.popMin();            // 最近的事件
    if (stale(e)) continue;             // 粒子已被别的碰撞改变轨迹 → 事件作废（懒惰删除！）
    sim_time = e.t;
    resolve(e.a, e.b);                  // 弹开速度
    for 邻居 j: push(predict(e.a, j)); push(predict(e.b, j));
}
```
两个教科书要点：① **懒惰删除**——粒子碰撞后旧事件全部作废，但堆不支持高效删除，就打时间戳标记、pop 时跳过（散列章的懒惰删除在事件队列的转世）；② 处理事件数 ~O(N) 而非 N²——只有"可能碰撞对"入堆。这就是真实物理引擎（和 RopeNet 题）事件调度的原型。
