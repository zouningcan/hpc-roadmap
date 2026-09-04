# 模块 03（stage2）参考答案

## A1

输出 `12 15 20 10`——后进先出（LIFO），整体效果 = **反转序列**（这也是"浏览器后退/撤销历史"的工作原理）。

## A2

a) 能用不该用：裸 vector 语义开放（可插中间、可 find），读代码的人看不出"这题只需要 LIFO"；而 `stack` 一出现，LIFO 语义写在脸上。三重收益：**可读性即文档**、**杜绝错误姿势**（没提供的方法=没机会误用）、**实现可替换**（底层换链表，接口不变）。
b) 限制的价值：把"允许的操作"收缩到问题的真实需要，错误姿势在编译期就不存在，而不是靠测试事后抓。

## A3

`std::stack::pop()` 只删除、**不返回**被删元素（历史设计：返回大对象有异常安全问题）。正确二连：`int x = s.top(); s.pop();`。队列对应：`int x = q.front(); q.pop();`。

## A4

`q.size()` 每次循环都在变小、i 在变大，两头夹击提前相遇——6 个元素只处理一半就退出。正确：预存 `int n = q.size();` 循环内用 n；或 `while (!q.empty())` 排干。

## A5

先弹出的是**右操作数**（它是最后压进去的）。`-` `/` 不可交换：`次弹出的 ○ 先弹出的`（左○右）——写反结果就错，而 `+` `*` 交换律掩盖了顺序问题，所以一到减除就爆。

## A6

设计。stack/queue 是**适配器（adapter）**：刻意只暴露 LIFO/FIFO 操作、不提供迭代器——提供迭代器等于允许绕过纪律偷看中间元素，抽象就漏了。要"看全部"就得弹光（破坏性）或另用 vector 存。

## B1

a) **11**（官方答案）——快照见讲义第 5 节表
b) `7 2 -` → 5，`5 3 *` → **15**
c) `1 2 +` → 3，`3 4 *` → 12，`5 12 +` → 17，`17 3 -` → **14**

## B2 参考

```cpp
std::string rev(std::string s) {
    std::stack<char> st;
    for (char c : s) st.push(c);
    std::string out;
    while (!st.empty()) { out += st.top(); st.pop(); }
    return out;
}
// 轮转 k 次：
for (int i = 0; i < k; i++) { q.push(q.front()); q.pop(); }
```
陷阱复现：队列 {1..6}，输出 **1 2 3** 后停（i=0 时 size=6，pop 后 size=5，i=1... i=3 时 i==size==3 退出——总之只剩一半）。体感：**循环条件里的活值是定时炸弹**。

## B3 参考

```cpp
double evalPostfix(const std::string& expr) {
    std::stack<double> st;
    std::istringstream in(expr);
    std::string tok;
    while (in >> tok) {
        if (isdigit(tok[0]) || (tok.size() > 1 && tok[0] == '-')) {
            st.push(stod(tok));
        } else if (tok.size() == 1 && std::string("+-*/").find(tok[0]) != std::string::npos) {
            if (st.size() < 2) throw std::invalid_argument("operands");
            double r = st.top(); st.pop();          // 先弹 = 右
            double l = st.top(); st.pop();          // 后弹 = 左
            if (tok == "/" && r == 0) throw std::invalid_argument("div0");
            st.push(tok == "+" ? l + r : tok == "-" ? l - r :
                    tok == "*" ? l * r : l / r);
        } else {
            throw std::invalid_argument("bad token");
        }
    }
    if (st.size() != 1) throw std::invalid_argument("leftover");
    return st.top();
}
```
四类防御全部就位：坏 token（else 分支）、操作数不足（size<2）、除零、残留（≠1）。B1 三例应得 11、15、14。

## C1 参考

```cpp
bool balanced(const std::string& s) {
    std::stack<char> st;
    for (char c : s) {
        if (c == '(' || c == '[' || c == '{') st.push(c);
        else if (c == ')' || c == ']' || c == '}') {
            if (st.empty()) return false;
            char open = st.top(); st.pop();
            if ((c == ')' && open != '(') || (c == ']' && open != '[') ||
                (c == '}' && open != '{')) return false;
        }
    }
    return st.empty();
}
```
边界：`([)]` → 弹出 `(` 遇 `]` 配对失败 false；空串 → true；`)` → 栈空时遇右括号 false。嵌套结构 = 栈的天然领土。

## C2 参考

```cpp
class MyStack {
    std::queue<int> a, b;
public:
    void push(int x) {
        a.push(x);
        while (a.size() > 1) { a.push(a.front()); a.pop(); }  // 新元素转到队首
    }
    int top()    { return a.front(); }
    void pop()   { a.pop(); }
};
```
（用一个队列即可：push 后把前面的 n-1 个轮转到队尾。）push O(n)，pop/top O(1)。要点：**FIFO 转 LIFO 的代价就是那次轮转**——两个语义的差恰好是"转半圈"。

## C3 参考现象

默认栈大小（常见 8MB）下，`depth(100000)` 每帧约几十字节即可爆——**Segmentation fault**。`ulimit -s` 查看/调大后能递归更深，但上限永远是**一块有限内存**。防法：深递归改**显式栈的迭代版**（DFS 用 stack 容器自己管），或控制递归深度/改尾递归。比赛现场，deep DFS 爆栈是最常见的"莫名其妙的崩溃"——现在你知道它死在哪块内存了。
