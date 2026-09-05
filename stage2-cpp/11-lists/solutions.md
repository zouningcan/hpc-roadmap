# 模块 11（stage2）参考答案

## A1

数组三短处：① 定长——浪费或扩容**整体拷贝**；② **中间/头部插入要挪人**（O(n)）；③ **碎片化**时找不到整块连续内存。
链表的交换：O(1) 插入 + 碎片容忍 + 动态生长 ↔ 第 k 元素 O(k)、无法二分、每节点 ~12 字节（约 **3× 空间**）。

## A2

`headInsert` 要**改调用者的 head 指针本身**（头会换成新节点）——按值传时函数拿到指针的**副本**，改副本（挪动、置空）与调用者无关；`Node*&` 让参数成为调用者 head 变量的别名，改动直达本体。`printList` 只读链表、只挪自己的 walker，永远不需要改 head——`Node*` 足够。

## A3

错在**先 delete 后使用**：第二行解引用已释放内存（悬垂指针——值可能还在也可能已被回收，UB）。正确：
```cpp
Node* temp = head;      // 先存旧头
head = head->next;      // head 先撤离
delete temp;            // 再释放旧头
```

## A4

a) 不会——`current` 是**局部指针变量**（还是参数拷贝），给它赋新地址只是让自己的箭头改指，节点的 next 与调用者的 head 分毫未动。
b) `*current = *current->next;` **会改节点**——把下一个节点的内容整体拷进当前节点（覆盖 data 和 next）——这是内容操作，能直接破坏链表结构。`current =`（改指针）与 `*current =`（改指向物）一字之差、天壤之别。

## A5

单向链表每个节点只有"去路"没有"回路"——站在尾节点**够不着前驱**，删除尾节点需要前驱改 next。尾指针只告诉你"尾在哪"，帮不了"前驱在哪"。"次末节点指针"的失败：删除尾后次末成为新尾，**次末指针需要更新到次次末**——而它同样够不着，问题原样递归，无穷倒退。双向链表用 prev 一劳永逸：`tail->prev` 就是前驱，O(1) 尾删成立。

## A6

单节点 12 字节（4 数据 + 8 next）→ 双向 20 字节（多 8 字节 prev），**+67%**。印证的设计经济学：**花内存买运行时间**——数据结构设计的第一条交易规则（后续 B 树、跳表、哈希表都在做同一类买卖）。

## A7

| 情形 | headInsert | headDelete | 要点 |
|------|------------|------------|------|
| 空表 | 新节点即头（next=nullptr） | **先判空**，直接 return | 空表头删不判空 = 解引用 nullptr 段错误 |
| 单节点 | 新节点指旧头 | head 置 **nullptr**（若有 tail，tail 也要清） | 单节点删后变空表，两个指针都要归位 |
| 普通 | 新节点指旧头，head 前移 | 存旧头、head 前移、delete | 标准三步 |

## B1 参考

headInsert 3 → `3`；headInsert 1 → `1->3`；headInsert 4 → `4->1->3`；headDelete → `1->3`；tailInsert(9) 走到尾 → `1->3->9`。
图上必须体现：新节点从"悬空"到"挂进链"的**两步**（指旧头、head 改指）；walker 一格一格走到 nullptr 前停。

## B2 参考

```cpp
struct Node { int data; Node* next; };

void printList(Node* head) {
    for (Node* cur = head; cur != nullptr; cur = cur->next)
        std::cout << cur->data << " -> ";
    std::cout << "null\n";
}
void headInsert(Node*& head, int d) { head = new Node{d, head}; }
void tailInsert(Node*& head, int d) {
    Node* n = new Node{d, nullptr};
    if (!head) { head = n; return; }             // 空表：新节点即头
    Node* cur = head;
    while (cur->next) cur = cur->next;            // 停在最后一个节点
    cur->next = n;
}
void headDelete(Node*& head) {
    if (!head) return;
    Node* temp = head; head = head->next; delete temp;
}
void destroyList(Node*& head) {
    while (head) headDelete(head);                // 复用头删：删到空
    // head 已自然为 nullptr
}
int lengthIter(Node* h) { int n = 0; for (auto* c = h; c; c = c->next) n++; return n; }
int lengthRec(Node* h)  { return h ? 1 + lengthRec(h->next) : 0; }
```

## B3 预期

bad 版调用后**链表原封不动**（打印结果与调用前相同）——`head = head->next` 发生在副本上，函数返回即蒸发。good 版链表真的少了头（注意 good 版故意不 delete 旧头，会泄漏一个节点——本实验只演示指针传递语义）。

## B4 参考

```cpp
class LLQueue {
public:
    ~LLQueue() { while (_head) { Node* t = _head; _head = _head->next; delete t; } }
    void enqueue(int d) {
        Node* n = new Node{d, nullptr};
        if (!_head) _head = _tail = n;            // 空表：一头一尾都是它
        else { _tail->next = n; _tail = n; }
        _size++;
    }
    int dequeue() {
        Node* t = _head; int v = t->data;
        _head = _head->next;
        if (!_head) _tail = nullptr;              // 删空：尾也要清！
        delete t; _size--; return v;
    }
    int  peek() const { return _head->data; }
    int  size() const { return _size; }
    bool isEmpty() const { return _size == 0; }
private:
    struct Node { int data; Node* next; };
    Node* _head = nullptr;  Node* _tail = nullptr;  int _size = 0;
};
```
官方留的泄漏在析构——上面的 `~LLQueue()` 就是补丁。压测时 RSS 平稳（无泄漏版）；若注释掉析构，交替 10 万次且不释放被 dequeue 的节点，RSS 会爬升。

## C1 参考现象

`std::vector` 全量遍历通常比 `std::list` 快 **数倍**（节点 new 出来散落堆各处，每次 `next` 都大概率缓存 miss；vector 顺着缓存行线性扫+硬件预取）。与模块 02 行优先/列优先同因：**算法复杂度同为 O(n)，访存模式决定常数**。这就是"链表的第三宗罪"（官方讲义未展开、比赛必知）——HPC 热路径回避链表的真正原因。

## C2 参考

节点 `{data, prev, next}`，类持 `_head/_tail`。头插动 2 个指针（新.next=旧头、旧头.prev=新、_head 前移——共 3 处赋值），尾删同理镜像。边界矩阵：空表（两个判空）、单节点（删后 head=tail=nullptr；插后 head=tail=新节点）、两节点（删一端后另一节点同时成为头尾）。**双向链表每个操作的指针赋值清单 = 写代码前的 checklist**，漏一处就是断链。

## C3 参考

```cpp
void destroyRec(Node*& head) {          // 先递归后删（官方版本）
    if (!head) return;
    destroyRec(head->next);
    delete head;
    head = nullptr;
}
void printReverse(Node* h) {
    if (!h) return;
    printReverse(h->next);              // 先钻到底
    std::cout << h->data << " ";        // 返回路上打印——天然倒序
}
```
百万节点链表上递归版 destroy/printReverse **爆栈**：递归深度 = 链表长度（每层一帧），8MB 栈在几十万层就崩——与模块 03 C3 的 depth(n) 同一死法。教训：**线性结构的递归深度是 O(n)，树才是 O(log n)**——迭代版（B2 的 while 循环）是长链表的唯一安全写法。
