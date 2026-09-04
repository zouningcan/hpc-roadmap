# 模块 04（stage2）参考答案

## A1

无重复（插入已有元素静默忽略）、无顺序（不保插入序、无位置关系）。不支持下标因为**"第 i 个"这个概念不存在**——集合只关心成员资格。本质是**二元成员判定器**：contains 只答在/不在，不答几次。

## A2

size = **2**。`m["ghost"]` 判等时已把 "ghost"→0 插入（条件为真），随后 `m["phantom"] = 1` 又插一个。
只查不插：`m.count("ghost")`（返回 0/1）或 `auto it = m.find("ghost"); it != m.end()`。

## A3

**乙生效**。甲拿到的是 map 内 vector 的**副本**（值拷贝——模块 01 的规则对容器一样适用），push_back 落在副本上，函数结束即丢弃，map 里永远是空 vector；乙用**引用**直通 map 内部，改动落地。（官方 Exam Prep #2 原题同款。）

## A4

map 里只有 `alice → 95`（第二次赋值**覆盖**，一键一值）。一键多值用 `map<string, vector<string>>`：`m["alice"].push_back(...)` 链式追加。

## A5

`"Pegasus"` 先出。ASCIIbetical：'P'(80) < 'd'(100)，大写字母 ASCII 码整体小于小写——排序按字符码不按人类字典序。

## A6

有序版遍历输出排序、操作 O(log n)（树结构）；无序版哈希桶、平均 O(1) 但遍历乱序。大数据量 + 不在乎序 → `unordered_set/unordered_map`。

## B1 参考

```cpp
std::set<std::string> seen;
for (const auto& w : words) {
    if (!seen.count(w)) { std::cout << w << "\n"; seen.insert(w); }
}
// 重复词各报一次：
std::set<std::string> seen, reported;
for (const auto& w : words) {
    if (seen.count(w) && !reported.count(w)) { std::cout << w << "\n"; reported.insert(w); }
    seen.insert(w);
}
// 原地去重：
std::set<std::string> uniq(words.begin(), words.end());
words.assign(uniq.begin(), uniq.end());
```
嵌套版对第 i 个词要回看前 i-1 个，总比较 ~n²/2 次；set 版 n 次查询。

## B2 预期输出

```
0
0 0
0 1
```
第三行：`ages["alice"] > 18` 里的 `[]` **把 alice→0 插进去了**（0 > 18 为假），size 从 0 变 1——查询产生副作用。count 版则全程 size=0。

## B3 参考

```cpp
std::map<std::string, int> freq;
std::string w;
while (in >> w) freq[w]++;               // 缺省 0 起步自增（坑1恰好变正用）
for (const auto& [word, n] : freq)
    if (n >= 50) std::cout << word << " " << n << "\n";
```
`map[key]++` 是频次统计的世界标准惯性；结构化绑定 `[word, n]` 让遍历可读。

## B4 参考

a) **string**——13 位 ISBN 超出 int 范围（int ≈ ±21 亿，10 位）；它是标识符不是算术量，"长数字当名字用就存字符串"。
b) 平行 map（isbnToTitle + isbnToAuthor）查询要同步维护两份，坏味道。更好：
```cpp
struct Book { std::string title, author; };
std::map<std::string, Book> isbnToBook;
```
数据打包成类型——这正是下一站 OOP 的动机。

## C1 参考

```cpp
std::map<int, std::set<std::string>> byCount;
for (const auto& [word, n] : freq) byCount[n].insert(word);
// 次数最多的前 3：byCount 反向遍历（map 升序，取最后三个键）
```
值必须是 set：**多个词共享同一频次**（撞车）——反转键值时旧值的新键会重复，set 自动收拢且去重。

## C2 参考

```cpp
std::set<int> setUnion(const std::set<int>& a, const std::set<int>& b) {
    std::set<int> out = a;
    out.insert(b.begin(), b.end());            // 逐个插入版（红黑树去重）
    return out;
}
std::set<int> setIntersect(const std::set<int>& a, const std::set<int>& b) {
    std::set<int> out;
    std::set_intersection(a.begin(), a.end(), b.begin(), b.end(),
                          std::inserter(out, out.end()));   // 或手写双指针
    return out;
}
// 手写双指针骨架：
// 两个迭代器同步前进：相等收进结果、双方前进；小的那方前进
```
双指针版 O(n+m) 一趟扫完——**前提是两边有序**，这正是有序 set 送的便利（也预演了归并排序的合并步）。

## C3 参考现象

vector 每次查询平均扫一半（O(n)），set 走树高（O(log n)）——百万级数据下实测差距通常在**数十到上百倍**。随 n 增大：vector 查询时间线性涨，set 几乎不动（log 增长极缓）。下一讲 Big-O 会把"飞快"翻译成正式语言。
