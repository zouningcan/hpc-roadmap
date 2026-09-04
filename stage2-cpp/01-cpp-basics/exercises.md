# 模块 01（stage2）练习：C++ 基础与字符串

> 分层：A 概念 / B 动手 / C 挑战。🕷 = 高频易错。
> 全部用 WSL 的 g++ 编译运行：`g++ -std=c++17 -Wall 文件.cpp -o 程序 && ./程序`。`-Wall` 报的警告要当错误对待。

## A 概念题

**A1** a) 编译型与解释型语言的区别？b) 这个区别和"HPC 为什么选 C++"有什么关系？
*考察点：一次性翻译 vs 逐行解释；性能与编译期查错。*

**A2** 🕷 下面代码输出什么？为什么？
```cpp
int x;
cout << x << endl;
if (x == 1 || 2) cout << "yes" << endl;
```
*考察点：未初始化 = 垃圾值；`==1||2` 恒真。*

**A3** C++ 的 string 和 Python 的 string 在"可变性"上有什么区别？`s[0] = 'H'` 在两边分别意味着什么？
*考察点：C++ 原地改字符 vs Python 造新串。*

**A4** 传值和传引用的区别？官方用什么比喻？两个"需要传引用"的动机是什么？
*考察点：副本 vs 传送门；多返回值 + 免拷贝。*

**A5** a) `s.find("xyz")` 找不到时返回什么？b) 为什么 `if (s.find("xyz"))` 不是判断"找不到"的正确写法？
*考察点：string::npos；npos 是巨大的无符号数（truthy），正确判断是 `== string::npos`。*

## B 动手题

**B1（第一个程序与编译流程）**：
```bash
mkdir -p ~/ex/cpp && cd ~/ex/cpp
cat > hello.cpp <<'EOF'
#include <iostream>
using namespace std;

int main() {
    string s = "giraffe";
    for (char ch : s) cout << ch << endl;
    return 0;
}
EOF
g++ -std=c++17 -Wall hello.cpp -o hello && ./hello
```
然后改造：用**带下标的 for 循环**打印每个字符及其下标（`0: g` 格式）。
*考察点：两种遍历姿势的分工——要下标信息就不能用 range-for。*

**B2（🕷 mySwap——传引用初体验）**：
```cpp
#include <iostream>
using namespace std;

void badSwap(int a, int b) { int t = a; a = b; b = t; }
void mySwap(int& a, int& b) { int t = a; a = b; b = t; }

int main() {
    int x = 1, y = 2;
    badSwap(x, y);  cout << x << " " << y << endl;   // 预测输出？
    mySwap(x, y);   cout << x << " " << y << endl;   // 预测输出？
    return 0;
}
```
先预测再运行。用一句话向没学过的人解释 badSwap 为什么"白干"。
*考察点：传值副本丢弃 vs 传送门落地。*

**B3（字符串工具箱实操）**：
```cpp
string s = "Hello, C++ World";
// 依次求（每问用一条语句/一个小循环）：
// 1) 长度          2) 子串 "C++"（substr）
// 3) "World" 的下标（find）  4) 全部转小写后打印
// 5) 统计其中字母、数字、空格各多少个（用 <cctype>）
```
*考察点：length/substr/find；char 处理三件套（isalpha/isdigit/isspace）。*

**B4（char 算术：凯撒密码）** 明文每个小写字母后移 3 位（x→a, y→b, z→c 循环），其他字符不动：
```cpp
string caesar(string s, int shift) {
    // 你的实现
}
// 测试：caesar("hello xyz", 3) 应得 "khoor abc"
```
提示：`(ch - 'a' + shift) % 26 + 'a'`——先想清楚每个减加在干什么（相对序号再搬回字符域）。
*考察点：ASCII 相对算术；模块 03 Vim 的 B2 陷阱同款思想在算法侧的复现。*

## C 挑战题

**C1（手写 stringToInteger）** 不用 `stoi`，实现 `int myStoi(string s)`：把 `"4823"` 变成整数 4823。提示：`s[i] - '0'` 得数字值，从左到右 `result = result * 10 + 当前位`。扩展：处理负号。
*考察点：字符到数值的转换本质——一切 parseInt 的内核。*

**C2（二次方程求根——传引用带出多结果）** 函数返回值给"有几个实根"，两个根用引用参数带出：
```cpp
int quadratic(double a, double b, double c, double& r1, double& r2);
// 返回 0（无实根）/1（一个根）/2（两个根），r1/r2 装结果
```
为什么这个签名"必须"用引用？（提示：return 只能带一个值）
*考察点：传引用的动机一——多返回值。*

**C3（🕷 传值 vs 传引用的性能实测）**：
```cpp
#include <iostream>
#include <string>
using namespace std;

long totalByValue(string s)      { long t = 0; for (char c : s) t += c; return t; }
long totalByRef(const string& s) { long t = 0; for (char c : s) t += c; return t; }

int main() {
    string big(50'000'000, 'a');          // 50MB 字符串
    // 各调用一次，用 time ./prog 或 chrono 计时对比
}
```
`<chrono>` 计时：`auto t0 = chrono::steady_clock::now();` …… `chrono::duration<double>(clock::now()-t0).count()`。差距说明什么？`const string&` 里的 const 又保证了什么？
*考察点：拷贝成本可视化；const 引用 = 免拷贝 + 只读承诺（这正是比赛改传参拿分的第一课）。*
