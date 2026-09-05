# 模块 11（stage2）：链表

> **来源**：Stanford CS106B 2026 夏 L19《Introduction to Linked Lists》+ L20《More Linked Lists》
> `.../lectures/19-lists1/` `20-lists2/`
> 官方口号：**DRAW LOTS OF DIAGRAMS!（多画图！）**——链表题不画图等于裸奔。

## 0. 一句话定位

节点散落内存、靠指针串起来的线性结构：**插入永不挪人、永不扩容**，代价是**第 k 个元素要走着去**。它与 vector 是一对贯穿生涯的权衡样本。

## 1. 动机：数组的三宗罪 vs 链表的解法

**数组（vector 底层）**：

- ✅ 连续内存 → **O(1) 随机访问**（基地址+偏移），能二分
- ❌ 定长：大了浪费、小了**扩容拷贝**
- ❌ 保持有序的中间插入要"**大家挪一挪**"（头插 O(n)——模块 02 的 338× 现场第三次点名）
- ❌ **碎片化**：内存紧张时找不到一整块连续空间

**链表**：节点（数据 + "下一家的地址"）散落各处：

- ✅ 动态生长永不"满"；头插 O(1)；**碎片内存照常工作**
- ❌ 第 k 个元素 O(k)；二分不可能；每节点 ~12 字节（4 数据 + 8 指针），**约 3× 空间开销**

## 2. 节点与结尾标记

```cpp
struct Node {
    int data;
    Node* next;        // "下一家在哪"
};
// 尾节点 next = nullptr —— 链表的"到此为止"
```

🕷 `head` **本身不是节点，是指向首节点的指针**（变量而已，画图时画成独立小方框）——抽象图会掩盖这一点，官方要求**图上标内存地址**。

## 3. 遍历：走路人指针（walker）

```cpp
void printList(Node* head) {
    for (Node* current = head; current != nullptr; current = current->next)
        std::cout << current->data << " -> ";
}
```

- 核心惯性：`current = current->next;`（走到下一家）
- **为什么循环里随便挪 current 不毁链表**：参数是**拷贝**——挪的是自己的指针变量，不是节点的 next。`current = ...` 改指针；`*current = ...` 才改节点。一字之差
- 判空再解引用（`current != nullptr`）是防御性底线：**解引用 nullptr = 段错误**（模块 09 已知死法）

## 4. 📍 本模块第一难关：`Node*&`（指针的引用）

想**让函数改动调用者的 head**（插入/删除首节点必须改它）？

```cpp
void headInsert(Node*& head, int data) {   // 指针按引用传
    Node* node = new Node{data, head};     // 新节点指旧头
    head = node;                           // head 改指新节点
}
```

官方原话 "SUPER CRITICAL"：**指针按值传时函数拿到的是指针的副本**——改副本（甚至置 nullptr）对调用者的 head 毫无影响（illuminate 实验：置空了也没用）。**要改"指针本身"，就得传指针的引用 `Node*&`**。

判别口诀：**只读/只走路 → `Node*`；要动头 → `Node*&`**（printList 不需要，headInsert 必须）。

## 5. 头尾操作与尾指针

| 操作 | 做法 | 复杂度 |
|------|------|--------|
| 头插 | 新节点指旧头，head 前移 | O(1) |
| 头删 | 存旧头，head 前移，delete 旧头 | O(1) |
| 尾插（无尾指针） | 走到 `current->next == nullptr` 再挂 | O(n) |
| **尾插（有尾指针）** | 直接挂 tail 后，tail 前移 | **O(1)** |
| 尾删 | 单向链表走不到前驱 | O(n)；**双向链表 + 尾指针才 O(1)** |

🕷 **头删的正确姿势（官方坏代码解剖）**：

```cpp
// ❌ 坏：先 delete 后使用 —— 解引用已释放内存（悬垂，模块 09 死法之三）
delete head;  head = head->next;
// ✅ 好：先存后删
Node* temp = head;  head = head->next;  delete temp;
```

**双向链表**：节点加 `prev`（12→20 字节，+67% 空间），换来 O(1) 尾删与双向遍历。官方结论一句话：**花内存可以大幅买运行时间**——数据结构设计的第一条经济学。

## 6. 用链表实现栈和队列（L20 收官案例）

- **栈**：头进头出——push/pop 全 O(1)
- **队列**：头出尾进——**必须带尾指针**，否则入队 O(n)。官方给了完整 `LLQueue` 类（`_head/_tail/_size` 三成员 + enqueue/dequeue/peek/size/isEmpty，peek/size 标 const——模块 09 的规矩延续）
- 🕷 官方自曝：LLQueue 析构函数注释着 **"This has a memory leak!"**——写正确的 O(n) 析构留作考试练习（走一遍逐个 delete，最后 head/tail 置 nullptr）

## 7. 边界三态（官方反复强调）

1. **空表**：head（和 tail）== nullptr；删到空时**记得把 tail 也清空**
2. **单节点**：它同时是头和尾——插删**两个指针都要更新**
3. **未初始化的 head 是垃圾地址**——声明即 `Node* head = nullptr;`

## 8. 常见误区清单

1. 传 head 按值还想改它——改动只活在自己栈帧里
2. `delete head; head->next`——先删后用（悬垂）
3. 以为 `current = current->next` 会毁链表——改指针 ≠ 改节点
4. 删空链表忘清 tail；单节点操作漏更新一头
5. 不画图直接写指针操作——顺序错了全是悬垂/泄漏
6. 遍历条件写 `current->next != nullptr`——**漏掉最后一个节点**（除非刻意需要"停在倒数第二"）

## 9. 与超算比赛的联系

- **缓存是把双刃剑**：vector 连续 → 预取器友好；链表节点散落 → **每次 next 都可能缓存 miss**——HPC 热路径基本不选链表（这是"3× 空间"之外的第三宗罪，官方未展开但比赛必考）
- 但链表在**系统层无处不在**：内存分配器的 free list、调度器的任务链、内核的进程表——读 OS/分配器源码的第一关
- "空间换时间"（双向链表）与"时间换空间"的权衡 = 每道优化题的思考框架
- std::list 存在但 HPC 很少用——**选数据结构先看访存模式**，这是本模块给比赛的最大遗产
