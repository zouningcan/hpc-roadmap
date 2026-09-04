# 模块 02 练习：Shell 工具与脚本

> 分层：A 概念 / B 动手（预测→执行→对照）/ C 挑战。🕷 = 改编自真实学员错题。
> 先独立完成，再看 `solutions.md`。

## A 概念题

**A1** `foo=bar` 时写出三条的输出，并解释前两条为何不同：
```bash
echo $foo
echo "$foo"
echo '$foo'
```
另外：`foo = bar` 为什么报错？**说出机制**（不是"格式不对"）。
*考察点：引号三连；空格切词机制。*

**A2** a) `$?` 里存的是什么？值为 0 代表什么？b) 它和 `$0`、`$1`、`$@` 是一家子吗？
*考察点：退出码 vs 位置参数的区分。*

**A3** 假设 `touch a` 失败了，下面三条里 `ls` 各会不会执行？
```bash
touch a && ls
touch a ; ls
touch a || ls
```
*考察点：&& 成功才走 / ; 无条件 / || 失败兜底。*

**A4** `find . -name *.py`（无引号）在"当前目录恰好有一个 a.py、子目录有 b.py"时会发生什么？三种下场（0 个 / 1 个 / 多个匹配）分别是什么后果？
*考察点：glob 由 shell 展开；语义偷换最危险。*

**A5** 用一句话说清 source 与 `./` 的**本质**区别，并回答：为什么 Unix 默认设计成"执行脚本不影响当前 shell"？子进程能留给父进程的两样东西是什么？
*考察点：有无新进程；进程隔离的安全设计；stdout + 退出码。*

**A6** 官方讲义说"函数与脚本有差异（加载时机、执行环境）"。具体指什么？为什么 `mcd`（要 cd 的函数）必须 source，而一个算数据的脚本用 `./` 跑就行？
*考察点：什么时候用函数、什么时候用脚本。*

## B 动手题

**B1（backup 函数全流程）🕷**：
```bash
mkdir -p ~/ex && cd ~/ex
cat > backup.sh <<'EOF'
backup () {
    [ -f "$1" ] || { echo "文件不存在: $1"; return 1; }
    cp "$1" "$1.bak"
}
EOF
bash -n backup.sh          # 1. 语法体检：为什么这一步不能省？
source backup.sh
type backup                # 2. 这条输出证明什么？
echo 123 > test.txt
backup test.txt            # 3. 路径一：正常备份，ls 验证
backup 不存在.txt           # 4. 路径二：兜底分支应打印什么？
```
然后**开一个新终端窗口**，直接敲 `backup test.txt`：
5. 发生了什么？为什么？怎么让每个新终端都自动有这个函数？
*考察点：函数定义/加载/验证全流程；函数随 shell 生死；.bashrc 的作用。*

**B2（find 三连）**：
```bash
# a) 写命令：家目录下最近 24 小时修改过的所有普通文件
# b) 写命令：/data 下大于 100MB 且扩展名 .log 的文件
# c) 预测+执行（目录里有 test.txt 时）：
cd ~/ex && mkdir -p sub && echo hi > sub/notes.txt
find . -name '*.txt'       # 找到几个？
find . -name *.txt         # 找到几个？为什么少了？
touch another.txt
find . -name *.txt         # 现在呢？
```
*考察点：-mtime/-type/-size 组合；引号三态的现场复现。*

**B3（source vs ./ 对照实验）🕷**：完成 notes 第 2 节的 cdtest 实验，贴出两个 `pwd` 的结果，并用一句话解释差异。

**B4（官方练习：marco / polo）** 编写两个函数：
- `marco`：记录当前所在目录
- `polo`：无论现在在哪，跳回 marco 记录的目录
（提示：变量 + `pwd`；两个函数都要能被调用，思考怎么"记住"一个值。）

## C 挑战题

**C1（官方练习改编：抓失败）** 有一个脚本每次运行有 1% 概率失败（退出码非 0）。写一段 bash 循环：反复运行它直到失败为止，输出"运行了多少次才失败"。
*考察点：`while` + `$?`（或 `until`）的组合；退出码的实战用法。*

**C2（官方练习改编：空格安全打包）** 递归找出家目录下所有 `.html` 文件并打包成 zip，要求文件名含空格也不出错。
*考察点：`find -print0` + `xargs -0`，或 `find ... -exec zip out.zip {} +`。为什么管道直传会碎？*

**C3（法医训练）** 同事给你一个脚本，source 时报：
```
line 2: syntax error near unexpected token `}'
```
但你肉眼看第 2 行"完全正常"。列出你的排查步骤（至少 3 步，包含两个不同的怀疑方向）。
*考察点：cat -n / cat -A（CRLF 显形）/ bash -n / 保留字空格——模块 02 第 7 节的完整复用。*
