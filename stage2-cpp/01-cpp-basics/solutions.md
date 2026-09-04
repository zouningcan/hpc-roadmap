# 模块 01（stage2）参考答案

## A1

a) 编译型：整个程序先一次性翻译成机器码，再运行（g++）；解释型：逐行读逐行执行（Python 的 REPL 模式）。
b) 关系：①一次性编译 = 无解释器开销，运行速度接近硬件极限；②编译期就能抓类型错误（阶段1 bash 的运行时玄学 vs C++ 的编译期报错）；③可精细控制优化等级（-O2/-O3）与内存布局——HPC 的全部刚需。

## A2

- 第一行：**打印垃圾值**（未初始化的局部变量内容是内存残渣，不可预测）
- 第二行：**一定打印 yes**。`x == 1 || 2` 解析为 `(x == 1) || (2)`——x 是垃圾值所以 x==1 大概率假，但 `2` 本身非零即真，整体恒真。正确写法 `x == 1 || x == 2`。

## A3

C++ string **可变**：`s[0] = 'H'` 是原地修改那一格字符，对象的身份不变。Python string 不可变：任何"修改"都是**构造一个新字符串对象**再把名字绑过去。工程后果：Python 循环里 `s += ch` 造出大量中间对象；C++ 原地改零额外分配。

## A4

- 传值：参数是调用者变量的**副本**，函数内改动随栈帧销毁（官方比喻：只能抢到"复制品宝藏"）
- 传引用（`&`）：参数是通向调用者变量的**传送门（portal）**，改动直接落在原变量上
- 动机一：**带出多个结果**（return 只有一个，swap/求两根靠引用）
- 动机二：**免拷贝**（1GB 的 string 传值=复制 1GB；引用约 64 位门牌号）

## A5

a) 返回 `string::npos`（一个特殊常量 = size_t 最大值）。
b) `npos` 是巨大的非零数——在 if 里是 **truthy**！`if (s.find("xyz"))` 会在"找不到"时进入分支（而在找到且下标为 0 时反而不进）。正确写法：`if (s.find("xyz") == string::npos)`（找不到）或 `!= string::npos`（找到）。

## B1

range-for 版每行一个字符。带下标版：
```cpp
for (int i = 0; i < (int)s.size(); i++)
    cout << i << ": " << s[i] << endl;
```
（`i < s.size()` 有符号/无符号比较警告时，`(int)` 转换或用 `size_t i` 均可——见到 -Wall 的这个警告别无视。）

## B2

badSwap 后：`1 2`（交换发生在副本上，原值分毫未动）；mySwap 后：`2 1`。
一句话解释：badSwap 拿到的是 x、y 的**复印件**，改复印件当然改不了原件；mySwap 的 `&` 是直通原件的传送门。

## B3 参考

```cpp
string s = "Hello, C++ World";
s.size()                          // 15
s.substr(7, 3)                    // "C++"
s.find("World")                   // 11
for (char& c : s) c = tolower(c); // 全小写（注意接住/原地）
int letter = 0, digit = 0, space = 0;
for (char c : s) {
    if (isalpha((unsigned char)c)) letter++;
    else if (isdigit((unsigned char)c)) digit++;
    else if (isspace((unsigned char)c)) space++;
}
```
计数：字母 11、数字 0、空格 2、标点 2。（char 传给 isalpha 前转 unsigned char 是防负值 UB 的严谨写法。）

## B4 参考

```cpp
string caesar(string s, int shift) {
    for (char& ch : s)
        if (ch >= 'a' && ch <= 'z')
            ch = (ch - 'a' + shift) % 26 + 'a';
    return s;
}
```
拆解：`ch - 'a'` 把字符搬进 0–25 的相对序号域；`+shift` 后移；`%26` 循环回绕（z+3→c）；`+'a'` 搬回字符域。空格等非小写字符原样跳过。`char& ch` 的引用让原地修改成立（不用引用就得 `s[i] = ...`）。

## C1 参考

```cpp
int myStoi(string s) {
    int result = 0, i = 0, sign = 1;
    if (s[0] == '-') { sign = -1; i = 1; }
    for (; i < (int)s.size(); i++)
        result = result * 10 + (s[i] - '0');
    return sign * result;
}
```
内核：`s[i] - '0'`（字符→数值），`result*10 + 位`（从高位向低位进位累加）。这就是一切语言 parseInt 的骨架。

## C2 参考

```cpp
#include <cmath>
int quadratic(double a, double b, double c, double& r1, double& r2) {
    double d = b * b - 4 * a * c;
    if (d < 0) return 0;
    if (d == 0) { r1 = -b / (2 * a); return 1; }
    r1 = (-b - sqrt(d)) / (2 * a);
    r2 = (-b + sqrt(d)) / (2 * a);
    return 2;
}
```
必须用引用的原因：信息有三份（根数 + 两个根），而 return 通道只有一条——引用参数是另外两条出口。调用方先声明 `double r1, r2;` 传入。

## C3 参考结论

50MB 字符串下，传值版每次调用多出一次 50MB 拷贝——计时通常差出一个量级（具体倍数看机器，数量级不变）。
- 差距来源：拷贝 50,000,000 字节的内存带宽与分配成本，计算量本身两边相同
- `const string&` 的 const：**只读承诺**——享受传送门（免拷贝）的同时，编译器保证函数不改原串（改了编译错）。这是 C++ 传大对象入函数的默认姿势，也是比赛代码里"光改参数就提速"的经典第一刀
