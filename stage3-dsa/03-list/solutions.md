# 模块 03（stage3）参考答案

## A1

① 真实节点永不为首/末——插删操作**没有头尾特例分支**；② `header->succ`/`trailer->pred` 恒为首/末元素，边界访问统一；③ 空表也是合法结构（header↔trailer 互指）。CS106B 版需要逐案处理的"空表 head=nullptr / 单节点 head=tail"矩阵，被**结构本身**吸收——代码路径少一半，不变量一句话说完。

## A2

秩接口 O(r)（内部走链），位置接口 O(1)。`L[i]` 在双重循环里：内层每次 O(i)，总代价 Σi = **O(n²)**——表面是两个 O(n) 循环，实际阶级爆炸。规则：**循位置的循环里禁用下标访问**。

## A3

向量：数据连续存放，"搬"是 memmove 级的廉价操作——保留一个搬掉一串（双指针）；列表：搬数据要逐节点复制，但**删除/接驳只是改 2~4 个指针**——留一个删一个。手段相反、复杂度同 O(n)：**选结构擅长的动作**。

## A4

向量归并：**分**免费（下标折半 O(1)），**合**贵（归并要整层拷贝 O(n)）；列表归并：**分**贵（走链找中点 O(n)），**合**免费（接驳零数据搬移 O(1) 每次接驳、每层共 O(n)）。递归式同为 T(n)=2T(n/2)+O(n)=O(n log n)——**成本在分与合之间搬家，总量不变**。

## A5

从后向前找：插入目标位是"不大于新值的最后一个"，从已排序区**尾部**向头部扫，遇到第一个 ≤ e 的位置即停——平均只扫一小段；相等即停（`!（p->data > e)` 时停在相等者之后插入）意味着新元素落在**同值元素之后**——原序保持，稳定。

## A6

find 与 insertB 配对：找到目标节点 p 后，`insertB(p, e)` 把 e 放在 p 紧前——与"从前缀查找"的方向**天然吻合**，保证"查到即可接、接完即保序"；若向后找则查完还要折返，语义也要重新定义。

## B1 参考

```cpp
template <typename T> void List<T>::init() {
    header = new ListNode<T>; trailer = new ListNode<T>;
    header->succ = trailer; header->pred = nullptr;
    trailer->pred = header; trailer->succ = nullptr;
    _size = 0;
}
template <typename T> ListNode<T>* List<T>::insertA(ListNode<T>* p, T e) {
    _size++;
    return p->succ = p->succ->pred = new ListNode<T>(e, p, p->succ);
    // 四指针：new 的 pred=p、succ=p->succ；原后继的 pred=新；p 的 succ=新
}
```
验收要点：traverse 范围 `header->succ .. trailer->pred`；ASan 零报告。

## B2 参考

漏 `succ->pred = x`（原后继的回指）：**后继节点从此指不回新节点**——正向遍历看似完好，一旦从后继节点**反向**走或删除它，路径断裂（读到错误的 pred）。最小复现：插入后立即 `p->succ->succ->pred` 与期望比对。病理：**断链不立刻崩，延迟引爆**——这是链表 bug 难查的根源（对照模块 07"删掉 unswap"的延迟污染）。

## B3 参考

1) 三排序对随机数据输出一致（升序）；2) 接驳证据：给节点存插入时序号 id，排序后按 traverse 打印 id——**id 升序但节点地址未变**（打印 &node 对比归并前后集合相同）；3) 归并 5 万元素毫秒级，插入/选择秒级——O(n log n) vs O(n²) 的量级差。

## B4 参考

```cpp
ListNode<T>* middle() {
    auto *slow = first(), *fast = first();
    while (fast != trailer && fast->succ != trailer) {
        slow = slow->succ; fast = fast->succ->succ;
    }
    return slow;
}
```
偶数长度返回**后半首**（或前半尾，按停法）——写清约定；空表返回 trailer（哨兵再次兜底）；单节点返回自身。

## C1 参考

```cpp
int get(int key) {
    if (!map.count(key)) return -1;
    auto* p = map[key];
    int v = p->data.val;
    // 接驳到头：摘下 p，insertAsFirst 重挂（或标准 splice）
    moveToFirst(p);
    return v;
}
```
**哈希负责"找到"**（key→节点指针，O(1)）；**列表负责"移动与淘汰"**（O(1) 接驳到头、O(1) 删尾，尾节点指针也存进哈希或由 trailer 直接取）。缺任一半都退化为 O(n)。这是 LeetCode 146 / OS 页置换 / Redis LRU 的公共内核。

## C2 参考现象

位置版 O(n)：10 万元素微秒~毫秒级；秩版 O(n²)：**秒级以上**（每次 L[i] 从头/尾走 i/2 步）。教训印刷体：**循环体里的每个操作都带自己的复杂度——接口选错，好人当猪跑**。

## C3 参考

```cpp
auto* p = first();
while (_size > 0) {
    for (int i = 1; i < m; i++) {                 // 报数 m-1 次
        p = p->succ;
        if (p == trailer) p = header->succ;       // 绕圈：跳过哨兵
    }
    auto* next = p->succ; if (next == trailer) next = header->succ;
    out.push_back(remove(p));                      // O(1) 出列
    p = next;
}
```
列表版 O(n·m)；数组搬移版每出列 O(n) 总 O(n²)。n=10000、m=3：列表版毫秒级，数组版秒级——**O(1) 删除在环形/高频删除场景是决定性的**（哨兵让"绕圈"只需一行特判）。
