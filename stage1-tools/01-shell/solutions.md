# 模块 01 参考答案

> 先自己做，再对答案。看懂答案 ≠ 会做——三天后重做一遍 A4 和 B1 效果最好。

## A1

- **内核**：操作系统核心程序，直接管理 CPU/内存/磁盘等硬件资源
- **Shell**：包在内核外面的命令解释器，循环执行"读命令→找程序→运行→给结果"
- **终端**：显示 shell 的窗口程序，只负责显示与键盘输入
- 关系：你在终端里打字 → shell 解释并执行 → 需要硬件操作时通过内核完成

## A2

a) `~` 是家目录（`/home/用户名`），`/` 是根目录（整棵文件系统树的顶点）。
b) `~` 展开为 `/home/leo`，`..` 退一级 → **`/home`**。

## A3

- `d`：目录
- `rwx`：所有者可读、可写、可进入
- `r-x`：同组用户可读、可进入、不可写
- `r--`：其他人只能读，不能进入
- 要"进入 + 列出"：**x（进入）+ r（列出），两个都要**。只给 r 能看到条目但进不去也看不到详情；只给 x 能进去但不能 ls。

## A4

**B 能，A 不能。**
- A 中 `<` 重定向由 **shell（你的身份）** 执行：shell 要打开文件接到 cat 的 stdin，打开动作被权限拒绝——此时 sudo cat 根本没开始跑
- B 中文件路径是 **cat 自己的参数**：sudo 让 cat 以 root 身份运行，root 的 cat 打开文件 → 放行
- 规则：**权限检查针对"正在打开文件的进程"的身份**

## A5

"以长格式列出根目录 `/` 的全部内容，然后只取输出的最后一行。" `|` 把左边 `ls` 的 stdout 接到右边 `tail` 的 stdin，是两个程序之间的"水管"。

## A6

顺序：**别名 → 函数 → 内置命令 → $PATH 里的磁盘程序**（四关全落空才报 command not found）。当前目录不在 PATH 搜索范围——这是防"目录投毒"的安全设计（在共享目录放恶意 `ls` 等你踩）。想跑当前目录的脚本必须显式 `./hello`（带斜杠 = 按路径执行，不走 PATH）。

## B1 预期结果

| 命令 | 结果 | 原因 |
|------|------|------|
| `ls box` | ❌ Permission denied | 你是"其他人"，box 对其他人只有 x 没有 r，没有索引牌 |
| `cat box/inside.txt` | ✅ 打印 hello | 路径上每一段只需要 x（门禁卡能穿越），inside.txt 本身 644 可读 |
| `cd box` | ✅ 进得去 | x 允许进入 |
| 进去后 `ls` | ❌ Permission denied | 还是缺 r |

预测错在"能不能 cd"的，重看 notes 第 4 节门禁卡模型。

## B2 预期结果

- `sudo head /etc/shadow`：✅ 正常输出（root 的 head 自己打开文件）
- `sudo head < /etc/shadow`：❌ `bash: /etc/shadow: Permission denied`（shell 以你的身份打开文件失败，head 还没启动）

## B3

- 第一次 `./semester` 报 Permission denied，`ls -l` 显示 `-rw-r--r--`：所有人都没有 x，内核拒绝执行
- `chmod +x` 加上执行位后成功
- **shebang 机制**：`#!` 叫 shebang（释伴行）。内核执行文件时读头两个字节，发现 `#!` 就把该行剩余部分当解释器路径，实际执行 `/bin/sh ./semester`
- 若第一行没写进去（`echo #!/bin/sh > semester` 不带引号时，`#` 开头的词被当注释吃掉，写入的是空行）——这正是官方练习埋的坑

## B4

- `echo a*` → `a1.txt a2.txt`（bash 出门前把 `a*` 展开成当前目录匹配文件）
- `echo 'a*'` → `a*`（引号挡住 bash，echo 收到字面量并打印）
- echo 本身没变，变的是 bash 动没动手

## C1 参考思路

```bash
nonexistent_cmd > /etc/shadow
# bash: /etc/shadow: Permission denied
```

不存在的命令本应报 "command not found"，但报的却是打开文件的权限错误——证明 shell 在启动命令**之前**就先处理了 `>`（打开文件失败，命令根本没机会运行）。对照：`nonexistent_cmd` 不带重定向时报 command not found。

## C2 参考

`ls /sys/class/` 能看到 `net/`（网卡）、`block/`（块设备）、`thermal/`（温度）、`power_supply/`（电池/电源）等设备类。电池余量：`cat /sys/class/power_supply/BAT0/capacity`（真机上）。要点：sysfs 把内核参数和硬件状态暴露成普通文件，"一切皆文件"的设计。

## C3 参考

bash 的 glob 展开规则：**匹配到 0 个文件时，原样传递字面量**（默认不开 nullglob）。所以无 txt 文件时 `*` 原样传给 echo；有则展开为文件名列表。开 `shopt -s nullglob` 后，匹配为空时该词会被删除（echo 只打印空行）——知道这个开关的存在即可。
