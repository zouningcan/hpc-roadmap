# 模块 12（stage2）参考答案

## A1

前序 `23, 11, 19, 57, 89, 81, 99`；中序 `11, 19, 23, 57, 81, 89, 99`；后序 `19, 11, 81, 99, 89, 57, 23`。
**中序 = 升序序列**——BST 的免费性质（调试与合法性校验神器）。

## A2

插入永远发生在"**空着的 child 指针**"上：递归到 `root->left == nullptr` 时，需要被赋值的是**那个指针本身**。`TreeNode*&` 让参数 root 成为父节点 left 字段的**别名**——`root = new ...` 直接把新节点挂进树。按值传则写的是空指针的副本，白写（模块 11 的 `Node*&` 同一原理的树版）。

## A3

叶子：直接删；单孩子：孩子顶替（子树跟随）；双孩子：**取左子树的最大值**（一路向右）顶替本节点的值，再**递归删除左子树中那个 max 节点**——不递归删的话，同一个值在树里出现两份，有序性破坏。右子树 min 同样正确，课程标准统一用左 max 便于判卷。

## A4

**顺序/逆序输入**（1,2,3,...,n）——每个新节点都往同一边钻，树退化成单腿链表，查找 O(n)。自平衡树（AVL/红黑/2-4）在插删后做**旋转/变色**保持左右高度差 ≤1，把高度钉在 O(log n)。**std::map / std::set** 的内部就是红黑树。

## A5

后序 = 左右都处理完才轮到自己——烧掉两个孩子后节点成了孤叶，删它安全。前序先删自己，随后访问 `node->left` 是**解引用已释放内存**（悬垂，UB）：值可能还在，也可能已被人占用——正是模块 09 的"垃圾堆咖啡杯"。

## A6

a) 任何字符的编码都不是另一个字符编码的**前缀**。
b) 码 A 是码 B 的前缀 ⟺ A 的终点（字符 B）在 A 终点的**路径下方** ⟺ A 的终点是 B 的祖先——字符全在**叶子**上则无人是别人的祖先，前缀关系不可能存在。
c) 高频字符挂在浅处 → 路径短 → 码短；低频挂深处付出长码。总位数 = Σ(频次×深度) 最小化——**失衡正是优化的形状**（与 BST 追求均衡恰好相反）。

## A7

每轮要"取权重最小的两棵树"——最小值动态出队正是优先队列的定义操作（模块 10 的堆兑现处）。平局任意破 → 不同但**等优**的树 → 码表可能不同——**编码树（或等价码表）必须随压缩数据一起打包传输**，解码端重建出同一棵树才能还原。

## B1 参考

1) 见 A1。2) 层序：`23, 11, 57, 19, 89, 81, 99`（队列逐层横扫）。3) 普通树中序输出**无序**——"中序=升序"是 BST 有序性的推论，不是遍历本身的性质。

## B2 参考

```cpp
bool contains(TreeNode* node, int key) {          // BST 版：只走一边
    while (node) {
        if (key == node->value) return true;
        node = key < node->value ? node->left : node->right;
    }
    return false;                                  // O(h)
}
void bstDelete(TreeNode*& root, int key) {
    if (!root) return;
    if (key < root->value) bstDelete(root->left, key);
    else if (key > root->value) bstDelete(root->right, key);
    else {
        if (!root->left && !root->right) { delete root; root = nullptr; }      // 叶
        else if (!root->left) { TreeNode* t = root; root = root->right; delete t; }   // 单孩
        else if (!root->right) { TreeNode* t = root; root = root->left; delete t; }
        else {                                                                  // 双孩
            TreeNode* m = root->left;
            while (m->right) m = m->right;          // 左子树最大
            root->value = m->value;
            bstDelete(root->left, m->value);        // 递归删它
        }
    }
}
```
每次删除后中序仍应升序且元素集合正确——**中序校验法**是最便宜的回归测试。

## B3 参考

```cpp
void forestFire(TreeNode*& root) {
    if (!root) return;
    forestFire(root->left);
    forestFire(root->right);
    delete root;
    root = nullptr;
}
```
ASan 下正常版无报告；前序版触发 **heap-use-after-free**（ASan 直接打印被释放地址的分配/释放栈——悬垂的铁证）。

## B4 参考（"abracadabra"）

频次：a5 b2 r2 c1 d1。建树过程（一种破平局法）：
c+d→2；{2(b),2(c+d),2(r)} 两两合并：b+cd→4、r? →按权重流：2+2→4，2+4→6……最终一种树：a=0，r=10，d=1100，c=1101，b=111（依破平局而异，总长相同）。
编码总长 = 5×1 + 2×3 + 2×3 + 1×4 + 1×4 = 5+6+6+8 = **23 位** vs ASCII 88 位（约 **26%**）。解码验证：沿 0/1 走到叶出字符回根——前 10 位应还原 "abracada…" 的开头。

## C1 参考

```cpp
struct Node { char ch; int weight; Node *left=nullptr, *right=nullptr; };
struct Cmp { bool operator()(Node* a, Node* b) const { return a->weight > b->weight; } };

Node* build(const std::string& text) {
    std::map<char,int> freq;
    for (char c : text) freq[c]++;
    std::priority_queue<Node*, std::vector<Node*>, Cmp> forest;
    for (auto& [c,w] : freq) forest.push(new Node{c, w});
    while (forest.size() > 1) {
        Node* a = forest.top(); forest.pop();
        Node* b = forest.top(); forest.pop();
        forest.push(new Node{'\0', a->weight + b->weight, a, b});
    }
    return forest.top();
}
```
encode：DFS 建 `map<char,string>` 码表；decode：根出发按位走。**往返断言** `decode(encode(s)) == s` 对随机串全过即正确性证明（性质测试）。"happy hip hop" 应得 ~34 位（33%——与官方一致，破平局不同无妨，总长唯一）。

## C2 参考现象

顺序插入的退化 BST：contains(99999) **毫秒级**（10 万层树高，O(n)）；随机序的平衡树：**微秒级**（~17 层）；std::set 两种输入**同样快**——红黑树的旋转让"形状与输入顺序无关"。差距通常 100~1000×：这就是"平均 O(log n)"和"保证 O(log n)"的鸿沟。

## C3 参考

```cpp
int height(TreeNode* n) { return n ? 1 + std::max(height(n->left), height(n->right)) : -1; }
int countLeaves(TreeNode* n) {
    if (!n) return 0;
    if (!n->left && !n->right) return 1;
    return countLeaves(n->left) + countLeaves(n->right);
}
bool isBST(TreeNode* n, long lo = LONG_MIN, long hi = LONG_MAX) {
    if (!n) return true;
    if (n->value <= lo || n->value >= hi) return false;
    return isBST(n->left, lo, n->value) && isBST(n->right, n->value, hi);
}
```
空树返回 -1（单节点高度 0 的自然约定）。isBST 的区间法：每个节点必须落在 (lo,hi) 开区间里，进入左子树收紧 hi、右子树收紧 lo——反例 `{50, 左子树含 {70}}`：70 与 50 比不违规（50<70 在右侧？不——70 在左子树，逐层区间会抓到它），但"只查父子"的版本放它过关。**局部合法 ≠ 全局合法**，区间把祖先的约束一路带下去。
