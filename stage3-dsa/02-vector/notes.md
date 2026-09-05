# 模块 02（stage3）：向量 Vector

> **来源**：清华邓俊辉《数据结构（C++ 语言版）》第 2 章 + 学堂在线《数据结构（上）》
> 课程页：https://www.xuetangx.com/course/THU08091000384 （GB 编码抓取受限，结构依据教材目录整理——见阶段 README 说明）
> 本模块**手写模板类 Vector<T>**（OJ 传统：不用 std::vector），stage2 模块 02/09 的知识在这里全部进厂重铸。

## 0. 一句话定位

向量 = **逻辑与物理次序统一**的线性结构（r 号元素就在 r 号单元）。本章三件大事：**可扩充向量的扩容策略（摊还分析的教科书案例）、有序向量的二分查找家族、向量版排序**。

## 1. Vector ADT 与内部骨架

```cpp
template <typename T>
class Vector {
protected:
    int _size; int _capacity; T* _elem;      // 规模、容量、数据区
    void copyFrom(const T* A, int lo, int hi);       // 区间复制
    void expand();                                    // 满则扩容
    void shrink();                                    // 装填因子过小则缩（<25%）
public:
    Vector(int c = 3, int s = 0, T v = 0)             // 容量/规模/初始值
        : _capacity(c), _size(s), _elem(new T[c]) { /* fill v */ }
    ~Vector() { delete[] _elem; }
    T& operator[](int r) { return _elem[r]; }         // 下标重载：像数组一样用
    int size() const { return _size; }
    int insert(int r, const T& e);                    // 指定位插入
    int remove(int lo, int hi);                       // 区间删除（返回删的个数）
    int deduplicate();                                // 无序去重
    int disordered() const;                           // 逆序对计数
    int uniquify();                                   // 有序去重
    int find(const T& e, int lo, int hi) const;       // 无序查找 O(n)
    int search(const T& e, int lo, int hi) const;     // 有序查找 O(log n)
    void sort(int lo, int hi);                        // 排序
    void traverse(void (*visit)(T&));                 // 遍历器
};
```

三个不变量（invariant）：`0 ≤ _size ≤ _capacity`；**逻辑序 = 物理序**；接口永远维护这两条。

## 2. 可扩充向量：扩容策略（本章的灵魂）

数组定长是死穴（模块 11 原罪）。Vector 的答案：装不下时 `expand()` 开新数组、搬家、释放旧的。**关键决策：新容量开多大？**

| 策略 | 每次 | n 次插入的总搬移 | 摊还 |
|------|------|------------------|------|
| **递增**：+固定量 I | O(n) | 1+2+…≈ n²/I → **Θ(n²)** | 每次 O(n) |
| **倍增**：×2 | O(n) 偶发 | 1+2+4+…+n < **2n** → Θ(n) | **每次 O(1)** |

级数工具箱直接判决（模块 01）：**递增=算术级数、倍增=几何级数**。代价：倍增的装填因子（_size/_capacity）最低 50%——**以两倍空间的冗余，换插入的常数摊还**（空间换时间的又一次明码标价）。

`shrink()` 是对偶操作：装填因子 < 25% 时减半缩容——防止"反复增缩抖动"用滞后区间（25%~100% 之间不动）。

## 3. 无序向量

- `find(e, lo, hi)`：自后向前逐个比对——**找最后出现者**的语义 O(n)
- `insert(r, e)`：r 及之后全体**后移一格**再放 e——O(n)（模块 02 的 338× 案例正式成文）
- `remove(lo, hi)`：整体前移 `hi-lo` 补洞——**区间删除 O(size-hi+lo)**，优于逐个删（每个 O(n)）
- `deduplicate()`：逐个元素查前缀有无重复、有则删——O(n²)（前缀查找 + 删除各 O(n)）

## 4. 有序向量

### uniquify：高低指针一趟清（经典！）

有序后相同元素必然相邻。蛮力版逐个删 O(n²)；**双指针版 O(n)**：

```cpp
template <typename T>
int Vector<T>::uniquify() {
    int i = 0, j = 0;                       // i: 已整理区末；j: 探测指针
    while (++j < _size)
        if (_elem[i] != _elem[j]) _elem[++i] = _elem[j];   // 新元素前移
    _size = ++i;
    return j - i;
}
```

每元素恰被扫一次——**把"发现重复就删"改为"只搬不重"**，删除的 O(n) 被吸收进单趟扫描。

### search：二分查找三版本（邓课的招牌细抠）

**版本 A（三分支）**：

```cpp
// 在 [lo, hi) 中查找 e
while (lo < hi) {
    int mi = (lo + hi) >> 1;
    if      (e < _elem[mi]) hi = mi;       // 左
    else if (_elem[mi] < e) lo = mi + 1;   // 右
    else return mi;                        // 命中
}
return -1;
```

每轮**两次比较**（< 和 >），最坏约 2·log₂n 次——正确但非最优。

**版本 B（二分支）**：只比一次，相等情形不单独分支，最后统一验证——比较次数 ~log₂(n+1)，**每元素深度更均匀**（比较次数集中在 log 附近，避免 A 的"好坏差 1 倍"）。

**版本 C（不变量语义）**：返回**不大于 e 的最后一个元素**的秩（找不到返回 lo-1）——语义统一为"插入位"，使 `insert(search(e)+1, e)` 保持有序。这是**接口设计课**：查找器与插入器共享同一语义，代码不再有特例分支。

```cpp
while (lo < hi) { int mi = (lo + hi) >> 1; (e < A[mi]) ? hi = mi : lo = mi + 1; }
return lo - 1;
```

**Fibonacci 查找**（一瞥）：切分点不取中点而取**黄金分割**——让"向左"（浅）比"向右"（深）多承担，均衡成功查找的平均代价。思想与二分同，系数之争。

## 5. 向量排序

**起泡排序（改进版）**：每趟把最大者冒到尾；`exchange` 标志——**某趟零交换则提前终止**（对已序输入 O(n)，与 stage2 模块 08 一致）。

**归并排序（Vector 版）**：分治 `mergeSort(lo,mi)` + `mergeSort(mi,hi)` + `merge`（B 数组暂存 + 双指针归回）——O(n log n) 稳定，见 stage2 模块 08 的完整推导，此处落到 Vector 成员函数里。

## 6. 常见误区清单

1. 递增扩容"每次只加一点不浪费"——总代价算术级数 O(n²)，摊还 O(n)
2. 倍增后忘释放旧数组 / 忘把 `_size` 维护好——泄漏 + 不变量崩
3. 逐个 remove 做区间删除——应整体前移一趟
4. uniquify 用"见重即删"——O(n²)；双指针单趟 O(n)
5. 二分查找 `while (lo <= hi)` 与 `[lo,hi)` 半开区间约定混用——死循环或漏元素（**先定区间约定再写代码**）
6. 版本 C 的返回值当"命中秩"用——它的语义是插入位，`A[lo-1] == e` 才是真命中

## 7. 与超算比赛的联系

- **扩容策略 = 动态数组的一切**：std::vector 的倍增策略与摊还分析就是本节；HPC 里"预分配够大、避免运行中扩容"是热路径常识（**知道摊还才知道何时值得打破它**）
- 二分查找家族的"语义统一"思想 = 接口设计的通用功力（赛题代码里边界特例越少 bug 越少）
- uniquify 双指针 = 快排分区、归并、**图算法 frontier 扫描**的同族技巧——"对撞/并行双指针"贯穿整个 HPC 算法库
- 邓课 OJ 禁 STL 的意义：**亲手实现过 = 赛场上敢改库源码的底气**
