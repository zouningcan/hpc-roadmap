# 模块 04（stage2）练习：Set 与 Map

> 分层：A 概念 / B 动手 / C 挑战。🕷 = 高频易错。

## A 概念题

**A1** Set 的"两无"是什么？为什么 set 不支持下标访问？它本质上是"什么判定器"？
*考察点：无重复/无顺序；没有位置概念；二元成员判定。*

**A2** 🕷 下面代码执行后 map 的 size 是多少？为什么？
```cpp
std::map<std::string, int> m;          // 空
if (m["ghost"] == 0) m["phantom"] = 1;
```
只查不插的正确写法（两种）？
*考察点：[] 缺键即插入默认值；count/find。*

**A3** 🕷 想给 `map<string, vector<string>>` 里的 "Julie" 追加一个姓氏，下面两段哪个生效？为什么？
```cpp
// 甲
std::vector<std::string> v = m["Julie"]; v.push_back("Smith");
// 乙
std::vector<std::string>& v = m["Julie"]; v.push_back("Smith");
```
*考察点：取容器值必须引用；拷贝语义复发。*

**A4** `score["alice"] = 90;` 之后再 `score["alice"] = 95;`——map 里是什么？想"一键多值"正确的容器形状是什么？
*考察点：覆盖语义；map<K, vector<V>>。*

**A5** `std::set<std::string>` 里同时放入 `"Pegasus"` 和 `"dragon"`，遍历谁先出？这叫什么序？
*考察点：ASCIIbetical——大写 ASCII 小。*

**A6** `std::set` 和 `std::unordered_set` 的两个核心区别？大数据量、不在乎遍历序时选哪个？
*考察点：有序 O(log n) vs 哈希平均 O(1)。*

## B 动手题

**B1（去重三连，官方开题挑战）** 给定 `vector<string> words`（含大量重复）：
1. 打印每个词恰好一次（set 法），并数一数"你原来的嵌套循环版本要做多少次比较"（O(n²) 的体感）
2. 找出所有出现次数 ≥ 2 的词，每个只报一次（提示：两个 set——见过的 + 报过的）
3. 把 words 本身去重（清空重灌法）
*考察点：set 化解 O(n²) 的模式；双 set 状态机。*

**B2（🕷 坑 1 现场复现）**：
```cpp
std::map<std::string, int> ages;      // 空的
std::cout << ages.size() << "\n";     // 0
bool hasBob = ages.count("bob");
std::cout << hasBob << " " << ages.size() << "\n";      // 预测
bool older = ages["alice"] > 18;
std::cout << older << " " << ages.size() << "\n";       // 预测！
```
逐行预测再运行。哪一行产生了副作用？
*考察点：count 不插入 vs [] 插入。*

**B3（频次统计——官方德古拉案例的本地版）** 从文件读词统计频次：
```cpp
#include <fstream>
#include <map>
#include <sstream>
// 骨架：
std::ifstream in("/usr/share/common-licenses/GPL-3" /*或任何文本文件*/);
std::map<std::string, int> freq;
std::string word;
while (in >> word) freq[word]++;
// 然后：1) 打印全部 词条数、2) 滤出出现 ≥ 50 次的词（官方同款操作）
```
（`while (in >> word)` 按空白切词——暂不清洗标点。）
*考察点：map 计数的标准惯性；范围过滤。*

**B4（查找表与自定义类型的分界）** 实现 `isbnToTitle` 查询：给定 3 本书（13 位 ISBN！），支持按 ISBN 查书名。a) ISBN 用什么类型存，为什么？b) 再加"查作者"——你会开第二个平行 map 还是想办法把 title/author 放一起？写 b 的更好版本（struct Book）。
*考察点：坑 4 溢出；平行 map 的坏味道 → 自定义类型的动机。*

## C 挑战题

**C1（官方备考题：频次表反转）** 把 B3 的 `map<string,int>`（词→次数）反转成 `map<int, set<string>>`（次数→词集合），然后打印"出现次数最多的前 3 个次数"及各自的词数。反转键值时"值会撞车"怎么办——为什么值的类型必须是 set？
*考察点：值变键的碰撞处理；map<K, set<V>> 的组合。*

**C2（给 std::set 补上官方的集合运算符）** Stanford 库有 `+`（并）、`*`（交）、`-`（差），std 没有。写三个函数：
```cpp
std::set<int> setUnion(const std::set<int>& a, const std::set<int>& b);
std::set<int> setIntersect(const std::set<int>& a, const std::set<int>& b);
std::set<int> setDifference(const std::set<int>& a, const std::set<int>& b);
```
并集尝试两种实现（逐个 insert vs 双指针归并），说明有序 set 给了双指针版什么便利。
*考察点：集合语义的落地；利用"有序"性质的归并技巧（stage3 归并排序的前菜）。*

**C3（性能体感：vector 扫 vs set 查）** 造 100 万个随机 int 存进 vector 和 set 各一份，然后做 10 万次 `contains` 随机查询，分别计时。差距多少？随数据量增长两者各怎么变化？（对照讲义"为什么说 set/map 飞快"——下一讲 Big-O 会给出正式语言）
*考察点：O(n) 扫描 vs O(log n) 查找的实测；为模块 05 铺垫直觉。*
