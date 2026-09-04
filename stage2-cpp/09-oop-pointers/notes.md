# 模块 09（stage2）：OOP、指针与动态内存

> **来源**：Stanford CS106B 2026 夏 L15《OOP》+ L16《Pointers and Arrays》+ L17《Dynamic Memory Management》
> `.../lectures/15-oop/` `16-pointers-and-arrays/` `17-dynamic-memory-management/`
> **本模块是 C++ 内存模型的心脏**——HPC 读代码、调 segfault、防泄漏的全部地基。

## 0. 一句话定位

三讲一条线：**类**把数据和操作打包 → **指针**让你直接握住内存地址 → **new/delete** 让数据活过函数的生死。学完后你终于能看懂 vector 的内部（模块 02 的"自动扩容"黑盒打开给你看）。

## 1. 类（L15）：数据 + 行为的打包

**class 是蓝图（类型），object 是按蓝图盖的房子（实例）**——`v1`、`v2` 都是 `Vector<int>` 的实例，内部各自独立。

**范式转变（电梯比喻）**：C 风格是"把电梯递给外部函数 `goToFloor(elevator, 3)`"；OOP 是"电梯自带按钮——按按钮就是调用它自己的功能"。**数据自己带着行为走**。

### 接口与实现分离

```cpp
// quokka.h —— 接口：说"能做什么"
#ifndef QUOKKA_H            // include guard 防重复包含
#define QUOKKA_H
class Quokka {
public:                      // 注意：public/private 不缩进（课程风格特例）
    Quokka();                                    // 构造函数：与类同名
    Quokka(std::string name, int howAdorable, std::string pic);
    ~Quokka();                                   // 析构函数：~开头，出作用域自动调用
    std::string getName() const;
    void setName(std::string name);
private:                     // 外部禁止直碰
    std::string _name;       // 成员变量习惯 _ 前缀
    int _howAdorable;
    std::string _location;
};                           // 🕷 结尾分号，漏了报错离谱
#endif
```

```cpp
// quokka.cpp —— 实现：说"怎么做"
#include "quokka.h"
std::string Quokka::getName() const { return _name; }   // 类名:: 作用域
```

🕷 **头文件里禁止 `using namespace std;`**——它会传染给每一个 include 它的文件。头文件里老实写 `std::`。

### 为什么要 private（封装的三条理由）

1. **防破坏**：若客户能写 `v.size = 1;`，三个元素的 vector 会"以为"自己只有 1 个——对象进入**破碎状态**
2. **限制取值**：`setName` 里校验敏感词（官方例：拒绝 "covfefe"）——强制走 setter 才有机会拦
3. **集中逻辑**：所有修改都过我的函数 → 我对对象状态的假设**终身成立**

### 构造 / 析构 / this

- 构造函数在**实例化瞬间**自动跑：`Quokka q("Muffinface", 5, "m.jpg");`；也可以匿名临时对象直接进容器 `v.add(Quokka("Hemmy", 4, "h.jpg"));`
- 析构 `~Quokka()` 在对象**死亡时**自动跑（栈帧弹出/容器销毁）
- 🕷 **官方实验彩蛋**：把 quokka 装进 vector，析构消息打了一大堆——**每次拷贝（装容器、扩容搬家）都造一个新对象**，也都要死一遍。模块 02 的"扩容拷贝"在析构器上现形
- `this`：成员函数内部的"我"——看不见外面的变量名，用 `this` 把自己传出去（`renderPage(this);`）

## 2. 指针（L16）：握住地址

**每个变量住在某个内存地址里（十六进制显示）。指针 = 存地址的变量**（通讯录里的一条地址）。

```cpp
int x = 42;
int* p = &x;      // & 用在"已有变量"前 = 取地址
*p = 30;          // * 用在"已有指针"前 = 解引用：顺着地址去改 x 本体
```

### 🕷 & 和 * 的双重含义（本讲第一坑）

| 符号 | 出现在声明里 | 出现在表达式里 |
|------|--------------|----------------|
| `&` | 声明**引用**（`int& r`） | 取**地址**（`&x`） |
| `*` | 声明**指针**（`int* p`） | **解引用**（`*p`） |

同一个符号，声明看类型、表达式看动作——**看它在哪个语境**（和 awk 的 $1 vs bash 的 $1 同款教训）。

### 声明风格的陷阱

```cpp
int* p, q, r;    // 🕷 只有 p 是指针！q、r 是普通 int——星号属于变量名不属于类型
int *p, *q, *r;  // 三个都是指针（最诚实的写法）
```

### 指针基本法

- **类型必须匹配**：`char* pc = &myChar;` ✅，`int* pi = &myChar;` ❌
- **一址多针**：`p` 和 `q` 都可以指向 x（各自是独立变量，装同一个地址）——`p == q` 比地址，`*p == *q` 比内容
- **nullptr**："哪儿都不指"。解引用它 = **段错误**（`*p = 50;` 当 p 是 nullptr 时崩）
- **指针可重指向，数组名不行**：`p = anotherArr;` ✅

### 数组与指针的血缘

裸数组的名字**就是首元素地址**（decay 衰变）：

```cpp
int arr[5];
int* p = arr;     // 合法：数组名 = &arr[0]
p[2] = 7;         // 下标本质 = 指针 + 偏移 → 所以访问是 O(1)
```

数组 vs vector：定长、无 size()、**无边界检查**（越界 = 未定义行为：毁内存/段错误/静默错——Stanford Vector 会报友好错误，裸数组不会）。vector 内部就是数组 + 长度字段 + 扩容拷贝——模块 02 的黑盒至此透明。

## 3. 动态内存（L17）：new / delete 与堆

**动机**：返回一个 vector 按值 = 慢拷贝；返回局部变量的指针 = **指向已死内存**（栈帧弹出即毁）。出路：`new` 把数据放到**堆（heap）**——函数返回了它还活着。

### 栈 vs 堆（本讲的宇宙观）

| | 栈 stack | 堆 heap |
|---|---|---|
| 谁管理 | 自动（函数进出） | 你（new/delete） |
| 生命周期 | 随栈帧生灭 | 从 new 到 delete |
| 大小 | 小（~8MB） | 大（受物理内存限制） |
| 速度 | 极快（挪个指针） | 较慢（分配器查找） |

官方双重证明"局部变量函数返回即死"：打印前后地址不同（拷贝去了别处）+ 析构消息在函数退出时打印。

### 铁律与四大死法

> **"For Every new, a Single delete"——每个 new 配一个 delete，且要在弄丢最后一个指针之前删。**

1. **内存泄漏（leak）**：忘了 delete 或弄丢了地址（"孤儿内存"）——官方演示循环 new 十亿次：OS 在程序退出时兜底回收，但**长期运行的程序被慢慢抽干**直至卡死
2. **悬垂指针（dangling）**：delete 后指针还在、继续解引用——内容可能还在（暂时）也可能已被复用。官方比喻：**从垃圾堆里捞回咖啡杯继续喝**
3. **漏写 `[]`**：数组要 `delete[] arr;`（配对 `new T[n]`）
4. **返回局部变量的指针/引用**——指向已爆栈帧，crash

### ArrayBasedStack：官方全流程案例

```cpp
class ArrayBasedStack {
public:
    ArrayBasedStack() { _elements = new int[DEFAULT_CAPACITY]; }
    ~ArrayBasedStack() { delete[] _elements; }        // 析构里收内存——RAII 的雏形
    void push(int value) {
        if (_size == _capacity) {
            int* bigger = new int[_capacity * 2 + 1]; // 📍 *2+1：从 0 起 *2 永远是 0！
            for (int i = 0; i < _size; i++) bigger[i] = _elements[i];
            delete[] _elements;                       // 📍 先删旧数组……
            _elements = bigger;                       // ……再换指针（顺序反了 = 泄漏）
        }
        _elements[_size++] = value;
    }
private:
    int* _elements;  int _size = 0;  int _capacity = DEFAULT_CAPACITY;
};
```

三个官方细节：`*2+1` 逃出零容量陷阱；**先 delete 再重指向**（否则旧地址丢失成孤儿）；析构函数收内存 = **资源获取即初始化（RAII）** 的雏形——官方未点名，但这就是它的形状。

### const 成员函数

`peek() const;` `size() const;`——**承诺不修改任何成员**，编译器强制执行。只读方法一律加 const 是好习惯（模块 01 的 const 引用精神在成员函数上的延伸）。

## 4. 加餐：智能指针（官方未讲，现代 C++ 必备）

手动 new/delete 的四大死法，标准库的解药是**智能指针**：`std::unique_ptr<T>` 在**离开作用域时自动 delete**（把 RAII 从模式变成语言设施）：

```cpp
#include <memory>
auto q = std::make_unique<Quokka>("Muffinface");   // 不用写 delete
// 作用域结束自动释放——泄漏与悬垂同时消失
```

比赛/工程代码里**优先 make_unique，裸 new 只在性能热点或有充分理由时出现**。

## 5. 常见误区清单

1. `int* p, q;` 以为 q 也是指针——星号属于名字
2. 声明里的 `&`/`*` 与表达式里的混淆——语境决定含义
3. delete 后继续用（悬垂）——喝垃圾堆里的咖啡
4. `delete` 配 `new[]`——必须 `delete[]`
5. 扩容先换指针后删旧数组——旧地址一丢，泄漏成立
6. 返回局部变量地址——栈帧已死
7. 头文件里 `using namespace std;`——传染全家
8. class 定义漏结尾分号——报错位置莫名其妙

## 6. 与超算比赛的联系

- **指针是 HPC 的母语**：赛题代码（C/C++/Fortran）里 `double* data` 满天飞——MPI 的缓冲区参数、CUDA 的 device pointer 全是它
- **泄漏杀死长任务**：跑 48 小时的模拟每迭代漏 1MB 就是死——`valgrind`/ASan（stage1 模块 07）就是查这个的
- **栈小堆大**：大数组放栈上直接爆（模块 03 C3），`new[]`/vector 是正路
- **RAII/scope-guard 模式**：锁、文件、CUDA stream 的现代管理全沿用"析构即释放"
- `new[]` 的手动内存 = 性能敏感路径绕过 vector 抽象的出路（HelloHPC 的 C++ 重写题里常见）
