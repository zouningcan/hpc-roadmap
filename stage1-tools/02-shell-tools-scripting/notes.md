# 模块 02：Shell 工具与脚本

> **来源**：MIT《The Missing Semester》2020 第 2 讲
> 中文：https://missing-semester-cn.github.io/2020/shell-tools/ ｜ 英文：https://missing.csail.mit.edu/2020/shell-tools/
>
> **结构修正说明**：本仓库模块划分对齐官方 11 讲。实战教学中曾把 tmux / ssh / dotfiles / 信号（SIGHUP）打包进"第 2 讲"，但官方它们属于**第 5 讲《命令行环境》**，将在模块 05 展开。本讲专注：bash 脚本、glob 与 find、代码查找、用法查询、**函数与脚本的差异（source）**。
> 标注 🕷 的条目来自真实学员错题。

## 0. 一句话定位

从"用 shell 对话"升级到"让 shell 替你干活"：写 bash 脚本做自动化、在百万文件里找到目标、读懂别人的脚本。这些是超算队每天用的基本功。

## 1. Bash 脚本基础

### 变量：一个空格引发的血案

```bash
foo=bar     # ✅ 等号两边【不能有空格】
foo = bar   # ❌ bash 空格切词 → "运行程序 foo，参数 = 和 bar" → foo: command not found
```

🕷 错误样本：把报错理解成"格式不对"。**报错内容是 `foo: command not found`——它在找一个叫 foo 的程序**。机制：切词规则（模块 01）的必然推论。

### 引号三连

```bash
foo=bar
echo $foo     # bar（但值含空格时会被拆词——word splitting）
echo "$foo"   # bar（双引号：展开变量，保持整词）——推荐写法
echo '$foo'   # $foo（单引号：纯字面量）
```

### 函数：发明一个你自己的命令

```bash
mcd () {
    mkdir -p "$1"
    cd "$1"
}
# source mcd.sh 加载后：mcd mydir 建目录并进入
```

语法纪律（防 bug 疫苗）：
- 调用**不写括号**：`mcd mydir` ✅，`mcd(mydir)` ❌（和 C 相反）
- `name () {` 每个空格都留；**`{` 是保留字（reserved word）不是符号**——🕷 `{echo` 会被解析成"一个名叫 `{echo` 的命令"，不开块
- `}` 独占一行
- 写完必跑 `bash -n 脚本` 做语法体检（不执行，零成本抓语法错）

### 特殊参数表（背下来）

| 符号 | 含义 |
|------|------|
| `$0` | 脚本/程序名 |
| `$1`…`$9` | 第 1–9 个参数 |
| `$#` | 参数个数 |
| `$@` | 所有参数（遍历用） |
| `$?` | **上一条命令的退出码（exit code）** |
| `$$` | 当前 shell 的 PID |
| `!!` | 完整的上一条命令（`sudo !!` 救场） |
| `$_` | 上一条命令的最后一个参数 |

🕷 高频错：把 `$?` 和 `$0/$1` 混为一谈。前者是**结果报告**，后者是**传入参数**，两家人。

### 退出码：0 是成功，非 0 是失败

和直觉相反。设计逻辑：失败有很多种（1 一般错误、127 找不到命令、130 被 Ctrl+C 杀死），成功只有一种，配 0。

```bash
ls /不存在 ; echo $?    # 2（失败）
ls ; echo $?            # 0（成功）

false || echo "失败兜底"   # ||：前一条【失败】才执行
true && echo "成功后续"    # &&：前一条【成功】才执行
echo a ; echo b           # ; ：无条件顺序执行
cd /data || exit 1        # 脚本经典写法：进不去就立刻退出
```

人话翻译：`&&`="成功了，然后"；`||`="不然的话就"（兜底）；`;`="干完这件，反正接着干"。

### 命令替换

```bash
echo "今天是 $(date)"              # $( ) 替换为命令输出
diff <(ls foo) <(ls bar)          # <( ) 替换为临时文件名，可当文件用
```

### shellcheck：bash 的语法检查器

写完脚本跑一遍 `shellcheck 脚本`，能抓出 90% 的坑（漏引号、误用变量）。`sudo apt install shellcheck`。

## 2. 函数与脚本的差异：source vs ./

**本质区别一句话：有没有新开一个进程。**

| 方式 | 发生什么 | 后果 |
|------|----------|------|
| `bash f.sh` / `./f.sh` | **fork 一个子 shell** 去跑文件；函数/cd/变量写进子进程内存；跑完退出 | 定义随子进程**火化**，你的终端毫无变化 |
| `source f.sh` | **不生新进程**；你的 bash 亲自逐行执行，等同亲手打字 | 函数、cd、变量落在**你自己的 shell** 里 |

C++ 类比：`./` 是**按值传递**（改副本），`source` 是**在 this 上调方法**（改本体）。

**验证实验（必做）**：
```bash
cat > /tmp/cdtest.sh <<'EOF'
cd /tmp
EOF
chmod +x /tmp/cdtest.sh
cd ~
/tmp/cdtest.sh           # 分身版
pwd                      # 还在 ~（分身的搬家与你无关）
source /tmp/cdtest.sh    # 本体版
pwd                      # /tmp（你真的搬家了）
```

**为什么这么设计**：默认隔离是安全底线——你跑的脚本可能是任何人写的，若默认能改你的终端（偷换函数、覆盖 PATH），终端随时被无形劫持。**source 是你亲手签的授权书**："这个文件我看过，允许它改我"。这解释了 `~/.bashrc` 的机制：shell 每次启动**自动 source** 它，所以里面放的 export/别名对每个新终端生效。

**单向门**：子进程只能回传两样东西——stdout/stderr 的内容 + 一个 8 位退出码（`$?` 之所以存在）。除此之外永远无法写回父进程内存（操作系统用 MMU 强制隔离）。

## 3. 通配符（globbing）：`*` `?` `{}`

```bash
cp *.txt backup/            # shell 先展开成文件名列表再传给 cp
touch file{1..3}.txt        # 花括号展开：一次建 3 个文件
cp code/{a,b}.py src/       # = cp code/a.py code/b.py src/
convert image.{png,jpg}     # = convert image.png image.jpg
```

**关键认知**：`*` 是 **shell 的**展开符——shell 在命令出门前就把 `*.txt` 换成了具体文件名列表，程序自己从未见过 `*`。（模块 01 两层模型的直接推论。）

## 4. 查找文件：find

**结构：`find 从哪找 怎么筛 找到后干嘛`（默认打印）**——find 自己遍历整棵目录树逐个匹配。

```bash
find . -name src -type d             # 名叫 src 的目录
find . -path '*/test/*.py' -type f   # 路径匹配
find ~ -type f -mtime -1             # 最近 24h 修改过的普通文件
find . -size +500k -size -10M -name '*.tar.gz'
find . -name '*.tmp' -exec rm {} \;  # 找到即删；{} 是占位符
```

条件组合：**直接并列 = 且**；`-o` = 或；`!` = 非；括号 `\(...\)` 分组。

`-mtime` 单位是**天**（`-1` = 24h 内；`+1` = 超过 1 天）；分钟级用 `-mmin -60`。

🕷🕷 **必考：`-name` 的模式必须加引号**。不加引号时 bash 先展开 `*.txt`，三种下场：

| 当前目录匹配到几个 | bash 的动作 | 后果 |
|---|---|---|
| 0 个 | 原样传字面量 | **碰巧正确**（运气） |
| 1 个 | 换成那个文件名 | **语义偷换，静默漏结果**（如漏掉子目录的 notes.txt）——最阴险 |
| ≥2 个 | 换成多个词 | find 报 `paths must precede expression`（吵闹但好发现） |

三种里只有加引号永远正确。现代 GNU find 检测到此错会提示 `possible unquoted pattern after predicate -name?`——看到这行 = 条件反射补引号。

## 5. 查找代码与历史命令

```bash
grep -C 5 "main" file.c     # -C 上下文行数
grep -v pattern             # 反选
grep -Rn "MPI_Init" src/    # -R 递归 -n 行号
rg "MPI_Init" -t py         # ripgrep：更快，自动忽略 .git 与二进制
```

超算比赛拿到几万行赛题代码，第一步永远是 `rg` 找入口函数。

历史：`history | grep find`；**Ctrl+R 关键词反向搜索**（最常用）；配 fzf 可模糊搜索。

## 6. 查看命令用法

```bash
cmd --help      # 最快
man cmd         # 完整手册（q 退出，/ 搜索）
tldr cmd        # 社区例句版（强烈建议安装）
type cmd        # 告诉你它是别名/函数/内置/磁盘程序（比 which 可靠）
```

内置命令（builtin，如 `cd`——它必须改变 shell 自身状态所以不能是外部程序）查帮助用 `help cd`，`man cd` 查不到。

## 7. 实战法医案例：一个空格引发的两条报错

真实事故还原。源文件：

```bash
back(){
 [ -f "$1" ] || {echo "文件不存在: $1";return 1;}
 cp "$1" "$1.bak"
}
```

source 时报两条错：`cp: cannot stat ''` + `line 4: syntax error near unexpected token }`。

**因果链**：`{echo` 缺空格 → `{` 没被认成开块（保留字必须独立成词）→ 但行尾的 `}` 被认成**关块** → **函数体在第 2 行就被提前关闭** → 第 3 行 `cp` 变成顶层命令，source 时真的执行了（$1 为空 → `cp "" ""` → cannot stat）→ 第 4 行的 `}` 无家可归 → 语法错。

**方法论**：① 报错会撒谎但不全撒——"cp 执行了"+"line 4 多余 }"两条证据足以反推文件结构 ≠ 预期；② `cat -n` 看现场（行号对齐报错）、`cat -A` 显形隐藏字符（`^M` = Windows CRLF 行尾，同样导致"看着正常的行报语法错"）；③ `bash -n` 语法体检；④ 3 行的文件不值得刑侦，重建 + 全套验证更快。

## 8. 常见误区清单

1. `foo = bar` 报错机制：切词后找程序 foo
2. `$?` 为 0 是**成功**
3. `find -name "*.py"` 的引号是语义的一部分，不是风格
4. `{` `}` 是保留字，必须独立成词
5. 函数定义后要 source 才能用；**换终端/重连后函数消失**，要重新 source
6. Windows 记事本编辑脚本会引入 CRLF，症状是"正常行报语法错"

## 9. 与超算比赛的联系

- 赛题代码主要用 bash 脚本驱动：编译、跑算例、收集结果（Graph500 题就是"写一个 shell 脚本完成全流程"）
- `find + grep/rg` 在几万行代码里定位优化点
- HelloHPC 第 5 题（Calculation）要求**自己写 compile.sh**——脚本就是题目的一部分
