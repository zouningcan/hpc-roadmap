# 模块 02（stage2）：Vector、Grid 与测试

> **来源**：Stanford CS106B 2026 夏 L4《Testing, Vectors, and Grids》
> https://web.stanford.edu/class/archive/cs/cs106b/cs106b.1268/lectures/04-vector-grid/
> **环境对齐**：官方用 Stanford 库 `Vector<T>`/`Grid<T>`（大写开头、报错友好）；本仓库用**纯标准库**，映射表见下。测试用 assert（C 挑战里带你手搓一个 mini 版 SimpleTest）。

## 0. 一句话定位

学会两件大事：**用容器组织数据**（Vector=变长数组，Grid=二维表）和**用测试保护代码**。附带本模块最贵的性能课：`insert(0)` 为什么危险、**行优先遍历为什么快**。

## 1. 官方库 → 标准库映射表（本仓库统一口径）

| 官方（Stanford） | 标准库等价 | 说明 |
|---|---|---|
| `Vector<int> v;` | `vector<int> v;` | 需要 `#include <vector>` |
| `v.add(x)` | `v.push_back(x)` | 尾部追加 |
| `v.insert(i, x)` | `v.insert(v.begin()+i, x)` | 中间插入（挤开别人） |
| `v.remove(i)` | `v.erase(v.begin()+i)` | 删除（后面补位） |
| `v.isEmpty()` | `v.empty()` | |
| `Grid<int> g(r, c);` | `vector<vector<int>> g(r, vector<int>(c));` | 标准库无 Grid，嵌套 vector |
| `g.numRows()/numCols()` | `g.size() / g[0].size()` | |
| `g.inBounds(r,c)` | 手写 `r>=0 && r<(int)g.size() && ...` | 官方帮你查，std 要自己查 |

## 2. 设计原则：函数分解（functional decomposition）

官方开场案例：判断"用户名是否出现在密码里"。烂写法是全塞进 main——**单一职责函数**（`extractAlpha()`、`isValidPassword()`）换来三样东西：**可读、可维护、可测试**（每段逻辑能被单独喂输入验证）。以后每写完一个函数，问一句"这函数能不能用一个 TEST 描述它"。

## 3. 测试：SimpleTest 的思想（我们用 assert 落地）

官方 `STUDENT_TEST("描述") { EXPECT_EQUL(期望, 实际); }` 的三要素：**描述、断言、可重复**。我们的等价写法：

```cpp
#include <cassert>
int extractAlpha(string s) { /* ... */ }

int main() {
    assert(extractAlpha("p4ss") == 3);      // 每个断言 = 一个微型测试
    assert(extractAlpha("") == 0);           // 边界：空串
    assert(extractAlpha("$$$") == 0);        // 全非法字符
    ...
}
```

**官方的测试脑暴清单**（针对 extractAlpha，值得抄进你的肌肉记忆）：全字母、非法字符在头/中/尾、交替、全非法、只有空白、空串、1–3 字符、超长、奇偶长度。**穷举"边界形态"而不是随机戳**——这是测试设计的核心功。另两个官方名词：测试驱动开发（先写测试再写实现）、小黄鸭调试法（对着橡皮鸭把逻辑讲一遍，讲到卡壳处就是 bug 藏身处）。

## 4. Vector：会自己长大的数组

```cpp
vector<int> v = {15, 20, 18};
v.push_back(33);           // {15,20,18,33} 尾部追加
v.insert(v.begin()+2, 90); // {15,20,90,18,33} 中间插入：右边全部挤开
v.erase(v.begin());        // {20,90,18,33} 删除队首：左边全部补位
cout << v.size() << v[0];  // 长度与下标访问（越界 = 崩溃/UB，同 string）
for (int x : v) { ... }    // range-for 遍历
```

性质：**有序、下标 0..n-1、同构（同类型）、底层连续内存**（下标访问 O(1)、对缓存友好）、自动扩容。

### 🕷 本模块最贵的性能课：insert(0) 是"危险动作"

往**头部**插入 = 现有全部元素右移一格。官方计时实测（TIME_OPERATION）：

| 规模 | `add`（尾部） | `insert(0)`（头部） | 差距 |
|------|--------------|--------------------|------|
| 50,000 | 0.005s | 0.086s | 17× |
| 500,000 | 0.029s | 9.794s | **338×** |

原因：尾部追加摊还 O(1)；头部插入 O(n)，n 翻倍工作量翻倍，差距指数式拉开。**推论：循环里逐个 insert(0) 建大数组 = 灾难**（正确姿势：push_back 再 reverse，或倒序填充）。

## 5. Grid：向量套向量，行在列前

```cpp
vector<vector<int>> g(3, vector<int>(4));   // 3 行 4 列，int 默认零初始化
g[2][3] = 18;                               // 永远是【行】【列】——row 在前
for (int r = 0; r < (int)g.size(); r++) {
    for (int c = 0; c < (int)g[0].size(); c++)
        cout << g[r][c] << " ";
    cout << endl;
}
```

**行优先（row-major）**：一行的元素在内存里连续。应用：棋盘、电子表格、**图像/网格计算**（超算赛题里 WRF 的大气网格、海洋模拟的差分网格全是它）。

🕷 **越界**：`g[9][9]` 崩溃或未定义行为；官方库报"index outside valid range"，std 什么都不说——遍历前判断边界是自己的责任。

## 6. 容器传参：引用是默认姿势

上一模块的结论落地：**vector/grid 传值 = 整份拷贝**。只读参数写 `const vector<int>&`，要修改写 `vector<int>&`。官方原话：传引用是"a smaller, faster transaction"。

## 7. 常见误区清单

1. 循环内 `insert(begin(), x)`——O(n²) 灾难（338× 实测）
2. 下标越界——std 不设防，段错误自己扛
3. 行列顺序写反——`g[c][r]` 在行列数不同时越界崩溃，相同时**静默算错**（更险）
4. 大容器传值——拷贝税；默认 `const&`
5. range-for 里改元素——`for (int x : v) x*=2;` 改的是副本，要 `int& x`

## 8. 与超算比赛的联系

- **行优先 = 缓存友好**：嵌套循环外层行内层列，顺着内存走；反过来（外层列）每次跳一行，缓存全 miss——**同一个算法差几倍性能**，这是 stage1 模块 07（perf/缓存）在数据结构侧的映照
- 连续内存是 vector 的全部优势来源——对比 stage2 模块 11 的链表（节点散落各处），就懂 HPC 为什么首选数组型结构
- HelloHPC 第 3 题（位矩阵）本质是"压扁的 grid + 位打包"；第 4 题（海洋模拟）就是网格迭代循环
- 官方计时实验的方法论（TIME_OPERATION = 我们的 chrono 计时）就是比赛优化的基准测量姿势
