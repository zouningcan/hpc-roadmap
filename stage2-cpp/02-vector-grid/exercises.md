# 模块 02（stage2）练习：Vector、Grid 与测试

> 分层：A 概念 / B 动手 / C 挑战。🕷 = 高频易错。编译统一 `g++ -std=c++17 -Wall`。

## A 概念题

**A1** Vector 的五个性质（有序/下标/同构/底层结构/增长）是什么？"底层连续内存"带来哪两个好处？
*考察点：O(1) 下标访问 + 缓存友好（对 HPC 的意义）。*

**A2** 🕷 解释 `insert(0, x)` 为什么是 O(n)？官方实测 50 万元素时它比尾部 add 慢多少倍？为什么规模越大差距越大？
*考察点：挤开元素的成本；O(n) vs 摊还 O(1) 随 n 指数式拉开。*

**A3** a) `g[r][c]` 里 r 和 c 各是什么？b) 行优先（row-major）对内存布局意味着什么？c) 为什么"外层循环遍历列"会慢？
*考察点：行在列前；一行连续；跨行跳访缓存全失。*

**A4** `for (int x : v) x = x * 2;` 执行后 v 变了吗？怎么改才对？
*考察点：range-for 默认拿副本；要 `int& x`。*

**A5** 大 vector 传参的默认姿势是什么写法？两个关键词各解决什么问题？
*考察点：const + 引用：免拷贝 + 只读承诺。*

**A6** 官方测试脑暴清单给了哪些"边界形态"类别？（说出至少 5 类）
*考察点：测试用例设计方法论——穷举形态而非随机戳。*

## B 动手题

**B1（Vector 操作追踪——纸笔+验证）** 预测每步后 v 的内容，再写程序验证：
```cpp
vector<int> v = {15, 20, 18};
v.push_back(33);
v.insert(v.begin() + 2, 90);
v.erase(v.begin());
v.push_back(v[0] + v[1]);
```
*考察点：中间插入/删除的挤位方向。*

**B2（🕷 亲手复现 338× 实验）**：
```cpp
#include <vector>
#include <chrono>
#include <iostream>
using namespace std;

void buildByAppend(int n) { vector<int> v; for (int i = 0; i < n; i++) v.push_back(i); }
void buildByHeadInsert(int n) { vector<int> v; for (int i = 0; i < n; i++) v.insert(v.begin(), i); }

int main() {
    for (int n : {50000, 100000, 200000, 400000}) {
        auto t0 = chrono::steady_clock::now();
        buildByAppend(n);
        double tA = chrono::duration<double>(chrono::steady_clock::now() - t0).count();
        t0 = chrono::steady_clock::now();
        buildByHeadInsert(n);
        double tB = chrono::duration<double>(chrono::steady_clock::now() - t0).count();
        cout << n << ": append " << tA << "s vs head " << tB << "s (" << tB / tA << "x)\n";
    }
}
```
观察：n 翻倍时两种构建的时间各怎么变？和 O(n)/O(n²) 对得上吗？
*考察点：亲手让复杂度"显形"——数字长什么样。*

**B3（Grid 遍历与边界）** 建 3×4 的 int 网格，(2,3) 处放 18，其余放行列和 `r+c`：
1. 嵌套循环打印（每行一行输出）
2. 求全部元素的和
3. 写函数 `bool inBounds(const vector<vector<int>>& g, int r, int c)` 并在访问前使用
4. 实验去掉 inBounds 后访问 `g[5][5]`——发生了什么？
*考察点：行优先嵌套遍历；自建边界保护；越界的真实下场。*

**B4（给函数写测试组）** 实现上模块的 `extractAlpha(string s)`（返回字母字符数），用 assert 写**至少 6 个**测试，覆盖官方脑暴清单里不同类别（空串/全非法/头中尾/交替……）。故意把实现改错一次，确认 assert 真的会炸。
*考察点：测试不是摆设——能抓住你故意埋的错才算测试。*

## C 挑战题

**C1（手搓 mini-SimpleTest，宏的初体验）** 用宏实现官方 SimpleTest 的雏形：
```cpp
#define EXPECT_EQUAL(a, b) do { \
    if (!((a) == (b))) { \
        cerr << "FAIL " << __FILE__ << ":" << __LINE__ \
             << " 期望 " << (a) << " 实际 " << (b) << endl; \
        return 1; \
    } } while (0)
```
用它测你的 `extractAlpha` 和 `caesar`。`__FILE__`/`__LINE__` 是什么？宏参数为什么要加那么多括号？（提示：`EXPECT_EQUAL(x+1, y*2)` 不加括号会怎样）
*考察点：测试框架的最小内核；宏的文本替换本质与括号纪律。*

**C2（缓存友好性初体验，行优先 vs 列优先）** 4000×4000 的 grid（int）：
```cpp
// 版本A：外层行内层列累加
// 版本B：外层列内层行累加
```
分别计时对比。两者加法次数完全相同，时间差说明什么？（这是 stage1 缓存知识的第一次"手感实验"，也是 HPC 优化的第一课）
*考察点：算法复杂度相同 ≠ 性能相同；内存访问模式决定速度。*

**C3（矩阵转置 + 传引用设计）** 写 `void transpose(const vector<vector<int>>& src, vector<vector<int>>& dst)`：dst 是 src 的转置。为什么 src 用 const 引用、dst 用普通引用？在函数体内只读 src、只写 dst，这个签名是不是"自文档化"的？
*考察点：参数修饰词表达函数的读写意图——签名即合同。*
