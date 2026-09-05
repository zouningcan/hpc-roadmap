# 模块 11（stage2）练习：链表

> 分层：A 概念 / B 动手 / C 挑战。🕷 = 高频易错。
> 铁律：每道指针操作题**先画图**（节点方框 + head 独立小框 + nullptr），再写代码。

## A 概念题

**A1** 数组的三个短处？链表用什么换来了什么（至少两对 trade-off）？
*考察点：扩容拷贝/中间插入挪人/碎片化；O(1)插入+碎片容忍 ↔ O(k)访问+3×空间。*

**A2** 📍 `headInsert` 为什么必须 `Node*&` 而 `printList` 用 `Node*` 就够？"指针按值传时改副本"是什么意思？
*考察点：要改"指针本身"就得传指针的引用；illuminate 实验。*

**A3** 🕷 下面头删代码错在哪？写出正确版本。
```cpp
delete head;
head = head->next;
```
*考察点：先删后用=悬垂；先存后删。*

**A4** a) `current = current->next;` 会毁调用者的链表吗？为什么？b) `*current = *current->next;` 呢？
*考察点：改指针 vs 改节点内容——一字之差。*

**A5** 尾指针把尾插从 O(n) 变 O(1)；为什么**尾删**还要双向链表才 O(1)？"次末节点指针"的提案为什么失败？
*考察点：单向到不了前驱；次末指针自己也要"往后走"从而需要次次末——无穷倒退。*

**A6** 双向链表多花多少空间（官方数字）？这印证了哪条设计经济学？
*考察点：12→20 字节 +67%；花内存买时间。*

**A7** 空表、单节点、普通情形——headInsert/headDelete/offWithItsHead 各需要照顾哪些指针？
*考察点：边界三态的指针矩阵。*

## B 动手题

**B1（手动画图追踪）** 纸上逐步画（含 head 框与 nullptr）：
1. `headInsert` 依次插入 3、1、4——每步画出新节点的进场
2. `headDelete` 一次
3. 无尾指针版 `tailInsert(9)`——画出走路人的移动轨迹
写出手算的最终序列，用 B2 程序验证。
*考察点：官方指定训练方式——图先行。*

**B2（基础全家桶）** 实现并测试：
```cpp
struct Node { int data; Node* next; };
void printList(Node* head);              // walker 遍历
void headInsert(Node*& head, int d);     // O(1)
void tailInsert(Node*& head, int d);     // 无尾指针版 O(n)
void headDelete(Node*& head);            // 先存后删
void destroyList(Node*& head);           // 官方练习：逐个 delete，最后 head=nullptr
int  length(Node* head);                 // 递归版 + 迭代版各写一个
```
测试序列自选；结束时 `destroyList` 后打印 head 确认是 nullptr。
*考察点：六个基础函数的完整落地；销毁不留泄漏。*

**B3（🕷 Node*& 对照实验）**：
```cpp
void badDelete(Node* head)  { head = head->next; }      // 按值
void goodDelete(Node*& head){ head = head->next; }      // 按引用（暂不 delete，只挪头）
```
对同一链表分别调用后打印——bad 版链表变了吗？**调用方的 head 为什么纹丝不动**？
*考察点：官方 illuminate 实验的链表版——按值改副本的铁证。*

**B4（LLQueue 完整版——官方案例+补泄漏）** 实现 `LLQueue`（`_head/_tail/_size`，头删尾进，peek/size 标 const），并**补上官方故意留的那个泄漏**：析构函数逐节点释放。10 万次 enqueue/dequeue 交替压测（期间用 `/proc/self/status` 的 VmRSS 观察内存平稳）。
*考察点：尾指针队列；官方 Exam Prep #7 的落地。*

## C 挑战题

**C1（缓存实测：vector vs list 的第三宗罪）** 100 万元素各自存进 `std::vector<int>` 和 `std::list<int>`，做**一次全量遍历求和**，各计时（重复 10 次取最小值）。差距说明什么？结合模块 02 的"行优先 vs 列优先"解释为什么。
*考察点：链表节点散落 → 逐节点缓存 miss；HPC 为什么回避链表。*

**C2（双向链表：补全 tail 的两端 O(1)）** 实现 `DoublyLinkedList`（节点带 prev/next，类持 head/tail）：头尾各 O(1) 的插入与删除、正反向打印。边界矩阵全测：空表、单节点、两节点。
*考察点：双向的指针矩阵（每次操作动 2~4 个指针）；+67% 空间换来的两端自由。*

**C3（递归与链表的化学反应）** 用递归实现三个：`length`、`destroyList`（官方：先递归后 delete）、`printReverse`（先递归后打印——模块 06 printStringReverse 的结构迁移到链表）。递归版 destroyList 在百万节点链表上会发生什么（模块 03 C3 的回归）？
*考察点：递归结构上的递归；爆栈的再现——链表版深递归的警示。*
