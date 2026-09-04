# 模块 02 参考答案

## A1

输出依次：`bar`、`bar`、`$foo`。
前两条差异在 **word splitting**：无引号的 `$foo` 若值含空格会被拆成多个词；双引号展开变量且保持整词。第三条单引号纯字面量。

`foo = bar` 机制：bash 按空格切成 `foo`、`=`、`bar` 三个词 → 第一个词当程序名 → 找不到叫 foo 的程序 → `foo: command not found`。报错内容本身就证明它在"找程序"。

## A2

a) `$?` 存**上一条命令的退出码**；**0 = 成功**，非 0 = 失败（1 一般错误 / 127 找不到命令 / 130 被 Ctrl+C）。
b) 不是一家子：`$0 $1 $@ $#` 是**传进来的参数**，`$?` 是**结果报告**。

## A3

| 命令 | ls 执行吗 | 原因 |
|------|-----------|------|
| `touch a && ls` | ❌ | && 需要前一条**成功** |
| `touch a ; ls` | ✅ | ; 无条件 |
| `touch a \|\| ls` | ✅ | \|\| 恰好在**失败**时触发（兜底） |

## A4

当前目录有一个 a.py：bash 把 `*.py` 展开成 `a.py`，实际执行 `find . -name a.py`——**不报错，但语义被偷换成"精确匹配名为 a.py 的文件"，子目录的 b.py 被静默漏掉**。
三态：0 个匹配=原样传（碰巧正确）；1 个=语义偷换（最危险）；≥2 个=`paths must precede expression` 报错（好发现）。

## A5

本质：**`./` 会新开一个子进程去跑，source 不开新进程、由当前 shell 亲自逐行执行**。
默认隔离是安全设计：脚本可能是任何人写的，若默认能改父 shell（偷换函数/改 PATH/移动目录），终端随时被无形劫持；进程内存由操作系统强制隔离，子进程**物理上无法**写父进程内存。
子进程能留下的两样东西：**打印到 stdout/stderr 的内容** + **一个 8 位退出码**。

## A6

- 函数：定义后**加载进当前 shell 内存**（source），调用快、能改变 shell 状态（cd、定义别名）；但随 shell 生死，换终端要重新加载
- 脚本：独立文件，`./` 执行时**另起子进程**，适合独立任务；改不了你的 shell 状态
- `mcd` 必须 source：cd 的效果要发生在**你自己的 shell** 里；算数据的脚本不需要影响你，`./` 跑完拿结果即可
- 想让函数每个终端都有：把定义（或 source 行）写进 `~/.bashrc`（每次开 shell 自动 source）

## B1

1. `bash -n` 不执行只查语法，零成本抓出 `{echo`、多余 `}` 这类错——本题历史事故里它 0.1 秒就能抓住真凶
2. `type backup` 输出 `backup is a function ...`，证明函数已住进当前 shell（名字+身份双确认）
3. 路径一：`ls` 出现 `test.txt.bak`，内容 123
4. 路径二：打印 `文件不存在: 不存在.txt`，`return 1` 让调用方能用 `$?` 探测失败
5. 新终端里 `backup test.txt` → `command not found`：**函数只住进当时 source 它的那个 shell，新窗口是全新进程，内存里没有它**。解决：把定义写进 `~/.bashrc`

## B2

a) `find ~ -type f -mtime -1`
b) `find /data -name '*.log' -size +100M`
c) 现场结果：
   - `find . -name '*.txt'` → 2 个（`./test.txt`、`./sub/notes.txt`）
   - `find . -name *.txt` → 只有 1 个 `./test.txt`（bash 把 `*.txt` 换成了当前目录仅有的 `test.txt`，语义偷换成精确匹配）
   - `touch another.txt` 后 → `find . -name '*.txt'` 3 个；`find . -name *.txt` **报错** `paths must precede expression`（展开成两个词）

## B3

第一个 `pwd`：`/home/用户名`（分身的 cd 与你无关）；第二个：`/tmp`（本体亲自 cd）。
一句话：`./` 是子进程执行、效果随子进程销毁；source 是当前 shell 亲自执行、效果落在本体。

## B4 参考实现

```bash
marco () {
    MARCO_DIR="$(pwd)"
    echo "已记录: $MARCO_DIR"
}
polo () {
    cd "$MARCO_DIR"
}
```
要点：用**变量**记住 `$(pwd)` 的输出（命令替换）；调用 `$( )` 时变量在当前 shell 里持久存在。

## C1 参考

```bash
count=0
until ./flaky_script; do
    count=$((count + 1))
done
echo "运行了 $((count + 1)) 次后失败"
```
或 while + `$?`：
```bash
count=0
while true; do
    ./flaky_script
    if [ $? -ne 0 ]; then break; fi
    count=$((count + 1))
done
echo "$((count + 1)) 次"
```
要点：`until` 在命令失败（退出码非 0）时继续循环；`$(( ))` 算术展开。

## C2 参考

```bash
find ~ -name '*.html' -print0 | xargs -0 zip backup.zip
# 或
find ~ -name '*.html' -exec zip backup.zip {} +
```
管道直传会碎的原因：`find` 默认输出用**换行**分隔，而 `xargs` 默认用**空格/换行**切分——文件名含空格时被拆成多个参数。`-print0`/`-0` 改用 **NUL 字符**（文件名中不可能出现的字节）分隔，绝对安全。

## C3 参考（至少含两个方向）

1. `cat -n 脚本` —— 行号对齐报错，确认"第 2 行"到底是哪行（肉眼数容易错）
2. `cat -A 脚本` —— 显形隐藏字符：行尾 `^M$` = CRLF（Windows 编辑器引入），`\r` 让 bash 把 `}` 解析成 `}\r` 这类怪词；顺手检查 `{`、`}` 是否被粘住了（`{echo`）
3. `bash -n 脚本` —— 独立语法体检，不执行就能复现
4. 修复：CRLF 用 `sed -i 's/\r$//' 脚本`；保留字粘录用编辑器补空格
两个怀疑方向 = **内容错**（结构/空格）与**编码错**（CRLF）。
