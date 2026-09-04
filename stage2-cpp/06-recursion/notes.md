# 模块 06（stage2）：递归

> **来源**：Stanford CS106B 2026 夏 L8《Introduction to Recursion》+ L9《More Recursion》+ L10《Recursive Problem Solving》（三讲合模块——课程的灵魂）
> `.../lectures/08-recursion1/` `09-recursion2/` `10-recursion3/`
> 官方开场白：**DON'T PANIC**——递归第一次见都懵，练出来的。

## 0. 一句话定位

递归 = **函数调用自己**。它不是炫技：分治、回溯、树的遍历、图搜索——本课程后半程和 HPC 的并行分治全部站在它上面。

## 1. 两要素（一切递归的骨架）

1. **基例（base case）**：某个输入的答案**直接已知**，立即返回——递归的"地基"
2. **递归步（recursive call）**：把问题分解成**更小的同类子问题**，交给自己的另一次调用——且子问题必须**朝基例推进**

```cpp
int factorial(int n) {
    if (n == 0) return 1;          // 基例：0! = 1
    return n * factorial(n - 1);   // 递归步：n! = n × (n-1)!
}
```

🕷 官方坑：基例写成 `n == 1` 会怎样？**`factorial(0)` 永远到不了基例**——一路减到负无穷，爆栈。基例必须**覆盖所有合法输入的收敛终点**。

## 2. 物理底座：调用栈（模块 03 的闭环）

`factorial(5)` 执行时，**每层调用压一帧**（参数、局部变量、返回地址）：

```
factorial(5) → factorial(4) → ... → factorial(0)   // 6 帧堆在栈上
factorial(0) 返回 1 → 逐层弹栈算 1×2×3×4×5 = 120
```

**无限递归 = 栈被塞爆**。官方课堂演示崩在约 n ≈ 261,000——每帧就几十字节，8MB 栈就是这么没的（模块 03 C3 你亲手炸过一次）。

## 3. 经典案例群（L8）

| 案例 | 分解方式 | 基例 |
|------|----------|------|
| `isPalindrome("racecar")` | 首尾相等 → 递归中间子串 | 长度 ≤ 1 |
| `printString(s)` | 打印首字符 → 递归剩余 | 空串 |
| `printStringReverse(s)` | **先递归剩余、回来再打印首字符** | 空串 |

🕷 **神例——printStringReverse**：与 printString 只差**两行换序**：

```cpp
void printStringReverse(std::string s) {
    if (s.empty()) return;
    printStringReverse(s.substr(1));   // 先钻到底
    std::cout << s[0];                 // 弹栈路上才打印
}
```

输出自动反转——**递归调用"钻下去"、返回路上"吐出来"**，栈的天然次序就是倒序。理解了它，你就理解了栈帧的生命周期。

**包装函数（wrapper）**：把"只做一次的杂务"（打印换行、初始化边界）放在外面，递归内核保持纯粹——比如二分查找的对外接口和递归内核分离。

## 4. 枚举生成器：soFar/rest 模式（L9 的核心）

**coinFlip**：生成 n 次抛硬币的全部 2ⁿ 种序列：

```cpp
void coinFlip(std::string soFar, int n) {
    if (n == 0) { std::cout << soFar << "\n"; return; }
    coinFlip(soFar + "H", n - 1);      // 分身一：这边加 H
    coinFlip(soFar + "T", n - 1);      // 分身二：这边加 T
}
```

**递归树就是决策树**：每层二选一，叶子是全部组合。`soFar`（已决定的部分）随身携带、逐层延长。

**permute（全排列）**：同一家族——循环里每次**摘掉 rest 的一个字符**接到 soFar 尾巴：

```cpp
void permute(std::string soFar, std::string rest) {
    if (rest.empty()) { std::cout << soFar << "\n"; return; }
    for (int i = 0; i < (int)rest.size(); i++)
        permute(soFar + rest[i], rest.substr(0, i) + rest.substr(i + 1));
}
// permute("", "act") → act, atc, cat, cta, tac, tca
```

## 5. permute 的第二种写法：原位交换 + **撤销**（L10）

```cpp
void permuteSwap(std::string& s, int k) {
    if (k == (int)s.size()) { std::cout << s << "\n"; return; }
    for (int i = k; i < (int)s.size(); i++) {
        std::swap(s[k], s[i]);       // 把第 i 个换到位置 k
        permuteSwap(s, k + 1);       // 递归处理后面
        std::swap(s[k], s[i]);       // 🕷 撤销交换（restore）
    }
}
```

🕷 **经典思考题（官方原题）：删掉第二个 swap 会怎样？** 答案不再完整且重复——循环继续时 s 已被污染，后续分支建立在"没还原"的状态上。**"改动-递归-撤销"三步曲是回溯（下一模块）的全部雏形**。

两版取舍：soFar/rest 逻辑好 trace 但每次拼接拷贝字符串（慢）；swap 版传引用零拷贝（快），代价是要记得撤销。

## 6. 分形：自相似的递归（L10）

Koch 雪花：一条线段，中间 1/3 "拱起"成等边三角形，得到 4 段——**每段递归同样处理**，直到层数耗尽。自然界的递归：罗马花椰菜、山脉、闪电、河流。要点：

- **深度优先执行**：动画里 4 个递归调用"同时"展开是假象，代码里**第一个分支连同它的全部后代跑完**，才轮到第二个——理解递归树遍历顺序的关键

## 7. 性能三课（L9-L10 官方洞见）

1. **coinFlip 其实是 O(n·2ⁿ) 不是 O(2ⁿ)**——string **传值**，每层拷贝 soFar（又见拷贝税！传引用降到 O(2ⁿ)）
2. **逐个 cout 慢，vector 收集快**：把 6⁴=1296 个结果 `push_back` 进传引用的 vector 最后统一打印——接近瞬间。I/O 比内存操作贵几个量级
3. **permute 是 O(n!)**：n! 个排列 × 每次调用 O(1) 工作；soFar 版因字符串拼接是 O(n·n!)——"多一个因子"的活教材

## 8. 二分查找与中点溢出（L9 的面试名题）

```cpp
int binarySearch(const std::vector<int>& v, int key, int lo, int hi) {
    if (lo > hi) return -1;                    // 基例：找不到
    int mid = lo + (hi - lo) / 2;              // 📍 神奇写法
    if (v[mid] == key) return mid;
    if (v[mid] < key)  return binarySearch(v, key, mid + 1, hi);
    return binarySearch(v, key, lo, mid - 1);
}
```

🕷 为什么 `mid = lo + (hi - lo) / 2` 而不是 `(lo + hi) / 2`？**后者在 lo、hi 都接近 21 亿时加法溢出**成负数；前者先做减法永不越界——数学等价、工程不等价。有序数组里找数：最坏 **O(log n)**，10 亿元素约 30 次比较（模块 05 的 log 直觉落地）。线性查找最好 O(1) 最坏 O(n)——**默认谈"最坏"**。

## 9. 官方四大坑（考试高频）

1. **非 void 函数漏 return**：编译器默认不拦，运行到路径尽头返回垃圾值
2. **基例覆盖不全**：`n==1` 挡不住 `factorial(0)` → 爆栈
3. **🕷 弱测试**：只测 true 用例的回文测试，**恒返回 true 的函数也能全过**——false 用例是尊严线
4. **复制函数后改名**：忘改函数体内的递归调用名 → 调用了旧函数

## 10. 整数除法的类型化学（L10 附赠）

`1/3 == 0`（整数除法截断）——想要 0.333 必须写 `1.0/3`。类型混合像显性基因：**任一操作数是 double，结果就是 double**；int+int 永远 int。递归图形/概率计算的头号隐形杀手。

## 11. 常见误区清单

1. 基例不全或子问题不朝基例推进——无限递归
2. 忘记撤销（swap/标记）——枚举重复或缺失（第 5 节）
3. `(lo+hi)/2` 溢出——用 `lo+(hi-lo)/2`
4. `1/3` 当 0.333 用——类型化学
5. 测试只有"是"用例——恒真函数骗过你
6. 递归体内大量字符串/容器传值——拷贝税叠加成阶乘级

## 12. 与超算比赛的联系

- **分治是并行的天然来源**：归并排序、FFT、矩阵乘的递归分解——每个子问题可交给不同核/节点（stage4 OpenMP task、MPI 分治的根都在这）
- 快排/归并（模块 08）的递归树 = 并行任务图
- 递归深度 = 栈预算：HPC 代码里深递归改显式栈迭代是常规防御
- 枚举器（coinFlip/permute）是回溯搜索（模块 07）的引擎——比赛里 N 皇后式参数搜索同款
