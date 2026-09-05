# 模块 04：数据整理（Data Wrangling）

> **来源**：MIT《The Missing Semester》2020 第 4 讲
> 中文：https://missing-semester-cn.github.io/2020/data-wrangling/ ｜ 英文：https://missing.csail.mit.edu/2020/data-wrangling/
> 本讲是前三讲（管道、grep、find）的**集大成**：把"一堆原始日志"加工成"一个答案"的完整流水线。

## 0. 一句话定位

数据整理 = **让数据变成你想要的形状**：过滤（grep）、变形（sed）、按列加工（awk）、统计（sort/uniq）——全部用管道串起来。超算比赛分析 HPL 跑分日志、统计赛题输出，用的就是这套。

## 1. 场景与思维框架：管道是逐步雕刻出来的

目标：从系统日志找出谁在尝试登录服务器。观察官方讲义的演进过程——**每一步只解决一个问题**：

```bash
ssh myserver journalctl                                          # 全量日志，没法看
ssh myserver journalctl | grep sshd                              # 只要 sshd 相关
ssh myserver 'journalctl | grep sshd | grep "Disconnected from"' | less
```

**省流量技巧**：引号让过滤在**远端**完成——本地只收到过滤后的结果（引号挡住本地 bash，整个管道原样传给远端执行——模块 01 两层模型的应用）。结果可 `> ssh.log` 落盘，避免反复网络传输。

## 2. 正则表达式（regular expression）

字符级匹配语言，grep/sed/awk 共用。速查：

| 模式 | 含义 | 模式 | 含义 |
|------|------|------|------|
| `.` | 任意单字符（换行除外） | `*` / `+` | 前一元素 0+ / 1+ 次 |
| `[abc]` | 字符集合（`[a-z]` 范围、`[^a]` 取反） | `(cat\|dog)` | 分支"或" |
| `^` / `$` | 行首 / 行尾锚点 | `( )` | **捕获组**，替换时用 `\1`、`\2` 引用 |

**两个大坑**：

1. **基础 vs 扩展正则（BRE/ERE）**：`(`、`|`、`+` 在 BRE 里要写 `\(`、`\|`、`\+`。**记住 sed 加 `-E` 用扩展正则**，少一半转义地狱；grep 加 `-E` 同理。
2. **贪婪匹配（greedy）**：`.*` 会吞掉尽可能多的字符。日志里有用户名本身叫 "Disconnected from user" 时，`.*Disconnected from` 会匹配过头。sed 不支持非贪婪 `*?`，需要时用 `perl -pe 's/.*?Disconnected from //'`。

## 3. sed：流编辑器（stream editor）

最常用的就一个命令——替换 `s/正则/替换/`：

```bash
sed -E 's/Disconnected from //'      # 删掉每行的这段前缀
```

官方压轴大正则（从登录日志提取用户名），逐段拆解：

```
sed -E 's/.*Disconnected from (invalid |authenticating )?user (.*) [^ ]+ port [0-9]+( \[preauth\])?$/\2/'
        └─┬─┘└──────┬──────┘└──────────┬─────────┘└─┬─┘└──┬──┘└─┬─┘└────┬─────┘└┬┘└─┬─┘
      前缀吞掉     可选的失败类型        user 关键字   用户名  空格 IP  port 端口号 可选preauth 行尾  只留第2组
```

- `[^ ]+` = 一段"非空格"文本（IP 地址），`[0-9]+` = 数字串
- 替换只写 `\2` = **丢弃其他部分，只留第 2 个捕获组（用户名）**
- **架构思想**：用 `^...$` 锚住整行 + 分组要保留的部分，是"提取字段"的标准套路

**原地修改的陷阱**：`sed 's/a/b/' file > file` 会先把 file 截空（`>` 由 shell 打开并清空，模块 01 的知识！），sed 还没读完就没了。正确做法：输出到临时文件再 `mv`，或 `sed -i`（原地，注意它其实也是生成临时文件再替换）。

## 4. awk：恰好擅长文本的编程语言

按**列**看世界：每行按分隔符（默认空格）拆成域，`$1`…`$n` 是第几列，`$0` 是整行，`NR` 行号、`NF` 列数。

```bash
awk '{print $2}'                          # 打印第二列
awk '$1 == 1 && $2 ~ /^c[^ ]*e$/ {print $2}'   # 带条件：第1列==1 且 第2列匹配正则
```

结构 = **（可选）模式 { 满足时执行的代码 }**。`BEGIN`/`END` 块在处理前/后各跑一次，适合初始化与汇总：

```bash
awk 'BEGIN { rows = 0 } { rows += $1 } END { print rows }'
```

理论上 awk 能替代 grep 和 sed（它有正则和替换），但"专刀专切"更顺手。

## 5. 统计工具链：sort / uniq / paste / bc

```bash
sort | uniq -c            # 计数的黄金搭档：先排序，相邻去重并计数
sort -nk1,1 | tail -n10   # 按第1列数字序（-n）排，取前10多
sort -r                   # 倒序（取最少用 head）
awk '{print $2}' | paste -sd,    # 把一列合并成逗号分隔的一行
paste -sd+ | bc -l        # 一列数字求和：合并成 1+2+3 形式喂给计算器
```

🕷 高频错：`uniq -c` 不排序直接用——**uniq 只合并相邻重复**，不排序结果是一堆散计数。`sort | uniq -c` 必须连用。

🕷 **课堂增量：sort 平局时的"兜底比较"（实战赌局翻车点）**

按 `-k1,1` 排序时若两行的**键相同**（如 root 和 admin 都出现 2 次），GNU sort 默认不会停手——它退回**整行字典序**做最后裁决（last-resort whole-line comparison）。两个推论：

1. **`-r` 连兜底一起翻转**：倒序不只是把次数倒过来，平局行的整行比较也反了。`sort -nk1,1 -r | head -n1` 遇平局选出的是字典序更靠后的那行——凭直觉赌"保持原顺序"（那其实是稳定排序的行为）会翻车。
2. **想要"只按键排、平局保持原样"：加 `-s`（--stable）**，显式稳定排序才是"严格大于才换位"。

> **head -n1 在平局时会撒谎**：top1 有并列时它只报一行，掩盖并列事实。报告场景的正确姿势是把并列第一名全吐出来：
> ```bash
> awk 'NR==1{m=$1} $1==m{print $2}'    # 以第一行的计数为基准，打出所有并列者
> ```

## 6. 完整流水线回顾（本讲的"毕业作品"）

```bash
cat ssh.log \
  | sed -E 's/.*Disconnected from (invalid |authenticating )?user (.*) [^ ]+ port [0-9]+( \[preauth\])?$/\2/' \
  | sort | uniq -c \
  | sort -nk1,1 | tail -n10 \
  | awk '{print $2}' | paste -sd,
```

四段分工：**sed 变形（提取用户名）→ sort|uniq 计数 → 排序取 top → awk+paste 汇成一行**。以后看到任何数据统计需求，先想这四段怎么排。

## 7. xargs：把输入变成参数

```bash
rustup toolchain list | grep nightly | grep -vE 'nightly-x86' | sed 's/-x86.*//' | xargs rustup toolchain uninstall
```

`xargs` 把 stdin 的每行**变成后面命令的参数**——管道的尽头不再是"打印"而是"执行"。含空格文件名要 `find -print0 | xargs -0`（模块 02 已学）。

## 8. 数据来源与结构化数据

- 获取：`curl`
- HTML → `pup`，**JSON → `jq`**（比赛和工程里 jq 出镜率极高）
- 二进制也能进管道：ffmpeg 截帧 → convert 转灰度 → gzip 压缩 → ssh 远端解压显示（官方演示，体会"一切皆流"）

## 9. 常见误区清单

1. `uniq` 只合并**相邻**行——计数前必须 `sort`
2. sed 不加 `-E` 时 `( )` `|` `+` 都要反斜杠转义，行为和预期不符
3. `.*` 贪婪：提取字段时优先"锚定整行 + 捕获组"而非裸 `.*`
4. `sed ... file > file` 自截断——`>` 由 shell 先清空文件
5. awk 的 `$1` 是"第 1 列"，和 bash 的 `$1`（第 1 参数）撞名不同义——看它在哪个语境

## 10. 与超算比赛的联系

- HPL/HPCG 跑完的日志几百行，`grep 'Gflops' | awk '{print $3}' | paste -sd+ | bc` 一行出总分
- 对比两次优化：`diff <(awk ... run1.log) <(awk ... run2.log)`（模块 02 的进程替换）
- HelloHPC 第 2 题（问答）里"20TB 走千兆网估算几小时"这类题，就是数据整理思维
