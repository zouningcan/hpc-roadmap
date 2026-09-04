# 模块 04（stage2）：Set 与 Map

> **来源**：Stanford CS106B 2026 夏 L6《Sets and Maps》
> https://web.stanford.edu/class/archive/cs/cs106b/cs106b.1268/lectures/06-set-map/
> **官方库 → 标准库映射**：`Set<T>`→`std::set<T>`、`Map<K,V>`→`std::map<K,V>`（有序版）；`HashSet/HashMap`→`std::unordered_set/unordered_map`（无序更快版）。官方的 `+ - *` 集合运算符 std 没有——练习里自己写。

## 0. 一句话定位

从"位置"思维（vector 的下标）切换到**"成员/关联"思维**：Set 回答"**在不在**"，Map 回答"**键→值是多少**"。两者查找都是"飞快"（对比逐个扫 vector）——为什么快是下一讲 Big-O 的事。

## 1. Set：数学集合

**两无**：无重复（add 已有元素静默忽略）、无顺序（没有下标、没有位置关系）。本质是"**二元成员判定器**"——`contains()` 只答 true/false，不答几次。

```cpp
std::set<std::string> s;
s.insert("dragon"); s.insert("Pegasus");
s.insert("dragon");                    // 静默忽略，size 仍为 2
s.count("dragon");                     // 1 = 在（std 用 count 或 find 判存在）
for (const auto& x : s) std::cout << x << " ";   // ✅ range-for 可以（vector 可以它也可以）
// for (int i = 0; i < s.size(); i++) s[i]      // ❌ 没有下标！
```

**遍历顺序**：std::set 按排序序输出。🕷 注意是 **ASCIIbetical**：大写字母的 ASCII 码小，所以 `"Pegasus"` 排在 `"dragon"` **前面**——不是你习惯的字典序。

## 2. Map：键值关联

每个键唯一，映射到恰一个值；**重复赋键 = 覆盖旧值**（不是追加）。

```cpp
std::map<std::string, int> score;
score["alice"] = 90;                   // 插入
score["alice"] = 95;                   // 覆盖！现在 map 里只有一个 alice=95
score.count("bob");                    // 0（std 的 containsKey 等价物）
for (const auto& [key, val] : score)   // C++17 结构化绑定，遍历得到 (键, 值) 对
    std::cout << key << " -> " << val << "\n";
```

命名习惯（官方建议）：变量名体现"键→值"方向，如 `isbnToTitle`、`wordToCount`。

## 3. 🕷 本讲四大坑（官方重点强调，考试常客）

### 坑 1：`map[key]` 查不存在的键 = **偷偷插入默认值**

```cpp
std::map<std::string, int> m;          // 空
if (m["ghost"] == 0) { ... }           // 这行执行完，m 里多了 "ghost"→0 ！
std::cout << m.size();                 // 1 —— 查询产生了副作用
```

`[]` 在键缺失时**插入默认值并返回它**（0、空串……）。只查不改的正确姿势：

```cpp
if (m.count("ghost")) { ... }                    // 先判存在
auto it = m.find("ghost"); if (it != m.end())    // 或迭代器式
```

### 坑 2：从 map 里取**容器型**值必须用引用

```cpp
std::map<std::string, std::vector<std::string>> name;
// ❌ std::vector<std::string> v = name["Julie"];   拿到的是副本！
//    v.push_back("X") 改的是副本，map 里永远是空 vector
std::vector<std::string>& v = name["Julie"];       // ✅ 引用直通 map 内部
v.push_back("X");                                   // 真的进去了
```

这是模块 01"传值 vs 传引用"在容器身上的复发——**拷贝语义无处不在**。

### 坑 3：`m[key]++` 合法，`m.get(key)++` 类似写法无效

计数的标准姿势就是 `wordToCount[word]++`（缺省 0 起步自增）。官方环境里 `get()` 返回右值不能自增；std 环境干脆没有 get，统一 `[]`（但记住坑 1：计数场景"缺省即 0"恰好是你要的，两坑对冲）。

### 坑 4：13 位 ISBN 存 int 会溢出

int 上限约 21 亿（10 位数）；13 位 ISBN 是万亿级——**当标识符用的长数字一律存 string**（要算术才用数值类型）。

**一键多值**（如名字 → 多个姓氏）：`Map<K, Vector<V>>`，配合坑 2 的引用：`map[key].add(...)` 链式追加。

## 4. 三大应用场景（官方演示）

1. **去重（deduplication）**：开题挑战"每个字符串只打印一次"。朴素解 = 嵌套循环查前缀 O(n²)；set 版 = 边走边 `contains`，优雅且快。变体：检测有无重复、每个重复只报一次（两个 set）、vector 去重（清空后从 set 回填）
2. **查找表（lookup table）**：ISBN→书名、学号→姓名。官方顺带指出：字段多了就该打包成**自定义类型**（`BookInfo`），别开一堆平行 map（OOP 模块的前菜）
3. **频次统计**：`Map<string, int>` 数文件里每个词出现几次（官方用《德古拉》滤出出现超 100 次的词）——**这是文本处理/日志分析的原型算法**

## 5. 有序 vs 无序（Set/Map vs Hash 版）

`std::set/map`：内部有序（红黑树——stage3 模块 7 的伏笔），查找 O(log n)，遍历输出排序。`std::unordered_set/map`：哈希表（stage2 模块 13 的伏笔），查找平均 O(1) 更快、但无序。课程规模差异可忽略；工程/HPC 里大数据量优先 unordered（迭代序无关时）。

## 6. 常见误区清单

1. `m[key]` 查询偷偷插入——判存在用 `count`/`find`
2. 从 map 取容器值忘了 `&`——改了副本
3. 以为 set 有下标——它连"第几个"的概念都没有
4. 排序当字典序——ASCIIbetical 大写在前
5. 13 位数字硬塞 int——溢出成负数
6. 想一键多值直接赋多次——被覆盖；要 `map<K, vector<V>>`

## 7. 与超算比赛的联系

- HelloHPC 第 6 题（魔咒筛选器）的**去重+校验+分类**流水线就是 set/map 的工业级应用
- Graph500 的 visited 标记、邻居去重——大规模图处理的基石操作
- 频次统计 = 日志分析的原子操作（stage1 模块 04 的 `sort | uniq -c` 是 shell 版，这里是程序版，同一思想两种表达）
- unordered_map 的哈希性能在大数据量下是真金白银的差距——"有序 vs 无序怎么选"是 HPC 工程日常判断
