# 模块 04 参考答案

## A1

- `^`：行首锚点（必须是行开头）
- `user `：字面文本"user "（含空格）
- `([a-z]+)`：捕获组，小写字母一个以上（用户名）
- ` port `：字面文本
- `[0-9]+`：数字一个以上（端口号）
- `$`：行尾锚点（后面不能再有内容）

匹配形如 `user alice port 22`（用户名纯小写、行内无多余内容）的整行。

## A2

a) `uniq` 只合并**相邻**的重复行——不排序时相同项分散各处，各自计 1。所以 `sort | uniq -c` 是固定搭配。
b) `-n` 按数值比较（否则 `9` > `10`，字典序逐字符比）；`-k1,1` 只用第 1 列作为排序键（防止次列影响顺序）。

## A3

不加 `-E`（基础正则 BRE）：`(`、`|`、`?` 被当**字面字符**，`(invalid |authenticating )?` 匹配的是"问号前面那一串字面括号竖线的零次或一次"——几乎什么都匹配不上。加 `-E`（扩展正则 ERE）后才是"可选的两种失败类型"分组。**结论：sed/grep 写正则一律默认带 `-E`**。

## A4

- awk `$1`：当前行的**第 1 列**（域）；bash `$1`：脚本/函数的**第 1 个参数**。同符号不同世界
- `NR`：当前行号（Number of Records）；`NF`：当前行的列数（Number of Fields）

## A5

`paste -sd+` 把一列（多行）用 `+` 连接成**一行算式文本**，如 `1+2+3`；`bc` 是任意精度计算器，收到这行文本求值输出总和。体会：Unix 工具之间传的永远是"文本"，把数据摆成算式形状就能让计算器吃下去。

## B1

a) 提取用户名（注意 `[preauth]` 的方括号要转义，且它可选）：
```bash
sed -E 's/.*Disconnected from (invalid |authenticating )?user (.*) [^ ]+ port [0-9]+( \[preauth\])?$/\2/' ssh.log
```
输出：root / admin / leo / admin / root / test

b) 统计：
```bash
sed -E 's/.*Disconnected from (invalid |authenticating )?user (.*) [^ ]+ port [0-9]+( \[preauth\])?$/\2/' ssh.log | sort | uniq -c
#  2 admin
#  2 root
#  1 leo
#  1 test
```
admin 与 root 并列最常被猜（真实世界里就是这俩最招攻击者）。

c) 取第一：
```bash
... | sort -nr | head -n1 | awk '{print $2}'
```
`-r` 倒序 + `head -n1` 取最多；`awk '{print $2}'` 把计数列丢掉只留用户名（并列时 sort 稳定序下 admin 在前）。

## B2

a) `grep -c 's$' /usr/share/dict/words`（`-c` 直接输出匹配行数）
b) 参考两种：
```bash
cut -c1 /usr/share/dict/words | sort | uniq -c | sort -nr | head -5
# 或
awk '{print substr($0,1,1)}' /usr/share/dict/words | sort | uniq -c | sort -nr | head -5
```
`cut -c1` 取每行第 1 字符；`substr($0,1,1)` 是 awk 等价物。词表基本全是小写开头，s/t/a/p/c 通常霸榜。

## B3

a) `awk '{print $2}'`
b) `awk '$1 >= 1024 {print $2}'` → docs notes.txt a.c
c) `awk 'BEGIN {s=0} {s+=$1} END {print s}'` → 7680
（`BEGIN` 初始化计数器、每行累加、`END` 输出——awk 统计的标准三段式）

## C1

两步：① shell 解析到 `>` 时**立即打开并清空** input.txt（`>` 的语义是截断写入，模块 01）——此刻文件已变空；② sed 开始读它，读到的是空文件，什么也不输出。原内容在 sed 开跑前就没了，**不是 sed 删的，是 shell 的 `>` 干的**。

正确做法：
```bash
sed 's/a/b/' input.txt > tmp.txt && mv tmp.txt input.txt   # 临时文件 + 原子替换
sed -i 's/a/b/' input.txt                                   # sed 原地（内部同样是临时文件）
```

## C2

regexone 通过后自然掌握：字符/类匹配、`*` `+` `?` 重复、`{m,n}`、分组捕获、条件分支。建议连同笔记里的大正则对照食用。

## C3

- `sed -E 's/.*from //'`：`.*` 贪婪，匹配到**最后一个** "from"，输出 `bob extra tail`（前面全被吞掉）
- `perl -pe 's/.*?from //'`：`*?` 非贪婪，匹配到**第一个** "from"，输出 `real user bob extra tail`
结论：**提取"分隔符之后的内容"，分隔符可能多次出现时必须非贪婪**（或把锚点和捕获组写得足够精确）。
