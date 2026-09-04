# 模块 09（stage2）参考答案

## A1

a) class 是**蓝图（类型）**，object 是**按蓝图盖的房子（实例）**——同一蓝图的房子内部各自独立。
b) C 风格把电梯递给外部函数处理（`goToFloor(elevator, 3)`）；OOP 把按钮装在电梯上——按按钮就是调用它自己的功能，数据带着行为走。
c) 防止客户把对象改入**破碎状态**（`v.size=1`）；**限制取值**（setter 校验）；修改逻辑**集中一处**，对状态的假设终身成立。

## A2

a) ✅ 取地址初始化指针。b) ❌ 把 int 赋给指针（类型不符，编译错）。c) ✅ 指针可置空。d) ❌ 解引用 nullptr——**段错误**。e) ❌ double* 不能接 int 的地址（指针类型必须匹配目标）。

## A3

只有 p 是指针，q、r 是普通 int——**星号属于变量名不属于类型**。诚实写法：`int *p, *q, *r;`（或一行一个）。

## A4

栈：自动管理（函数进出）、随栈帧生灭、小（~8MB）、极快；堆：手动（new/delete）、由你决定生死、大、较慢。
完整铁律：每个 new 配**恰好一个** delete，**且删在弄丢最后一个指向它的指针之前**。

## A5

a) delete 后指针仍在、内容已属未知——继续解引用即悬垂。官方比喻：**从垃圾堆里捞回咖啡杯继续喝**。
b) OS 只在**进程退出**时兜底；长期运行的程序（服务器、48 小时模拟）在退出前就被泄漏**慢慢抽干**直至卡死——兜底救不了"活着的时候"。
c) `new T` 配 `delete`；`new T[n]` 配 `delete[]`——混配是未定义行为。

## A6

容量 0 时 `0*2=0`，扩容永不发生——`+1` 逃出零陷阱。先删后指：若先把 `_elements` 指向新数组，旧数组的地址就**丢了**——没有任何指针记得它，想删都删不了，孤儿内存成立。

## A7

const 成员函数承诺**不修改任何成员变量**，编译器强制（写了编不过）。只读方法（getter/peek/size）加 const：让 const 对象也能调用它们、把"我只读"写进接口合同——签名即承诺。

## B1 参考

```cpp
// BankAccount.h
#ifndef BANKACCOUNT_H
#define BANKACCOUNT_H
#include <string>
class BankAccount {
public:
    BankAccount();
    BankAccount(std::string owner, double balance);
    ~BankAccount();
    void deposit(double amount);
    void withdraw(double amount);
    double balance() const;
    std::string owner() const;
private:
    std::string _owner;
    double _balance;
};
#endif
```
实现要点：deposit/withdraw 先校验（负数、超额）再改；main 结束/变量出作用域时打印"账户关闭"。观察：把账户塞进 vector 再扩容——析构消息暴增（每次拷贝都是一个新对象）。

## B2 预期输出

1) `42 99`　2) `50`（解引用改本体）　3) `1 1`（q 改指 x 后：内容同、地址也同）
4) 段错误（Segmentation fault）——nullptr 解引用
5) `3 3`（`s[2]` 与 `*(s+2)` 是同一个东西：下标=指针偏移）

## B3 参考

```cpp
void ptrSwap(int* a, int* b) { int t = *a; *a = *b; *b = t; }
// 调用：ptrSwap(&x, &y);   ← 调用方必须显式取地址
```
差别：指针**能为空**（调用方可能传 nullptr，函数得防）、可重指向、调用要写 `&`；引用**不可为空**、不可重绑、调用语法干净。改调用者变量的能力二者等价。

## B4 参考

实现骨架见讲义。压测现象：
2) 注释掉扩容处的 `delete[]` 后，循环建栈进程 **RSS 持续上涨**（每次扩容的旧数组全变孤儿）；恢复后 RSS 平稳——泄漏第一次有了数字。
3) `*2` 版在容量 0 时 `_capacity*2+...` 等于 0，扩容死循环或 push 永远失败（写不进第一个元素）——`+1` 是从无到有的那一步。

## C1 参考

`std::unique_ptr<int[]> _elements;` 后：**析构函数可以删空**（unique_ptr 出作用域自动 delete[]）；扩容"先删后指"变成纯赋值 `_elements = std::make_unique<int[]>(newCap)`（旧数组由旧 unique_ptr 的赋值自动释放）。被根除的两类死法：**泄漏**（忘 delete）与**先丢指针**（顺序错误）——所有权自动化后这类手误不再可能。

## C2 参考

```cpp
int** m = new int*[rows];
for (int r = 0; r < rows; r++) m[r] = new int[cols];
// ... 使用 m[r][c] ...
for (int r = 0; r < rows; r++) delete[] m[r];   // 先内层
delete[] m;                                       // 后外层
```
顺序不能反：先 `delete[] m` 则所有内层指针**随外层一起丢失**——无从释放，泄漏 rows 个数组。手动版新增的两个义务：**自己记尺寸**（裸指针不带 size）与**自己管生死**（vector 版两者全免）。

## C3 参考现象

1) 按值返回：现代编译器 **RVO（返回值优化）/移动**——打印地址会显示返回前后是同一块内存，没有深拷贝；"按值返回很慢"是 C++98 时代的旧常识
2) 返回 new 的指针：能用，但调用方背上 delete 义务（忘=泄漏，删错地方=悬垂）——所有权不清
3) 返回局部 vector 的引用：**未定义行为**——栈帧已爆，引用指向尸体；实测常见值诡异/直接崩
结论：默认按值 + RVO/移动语义 = 安全与性能兼得；指针返回仅在跨作用域共享所有权时考虑（且优先智能指针）。
