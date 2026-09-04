# 模块 01 练习：课程概览与 Shell

> **使用方法**：先独立完成全部题目，再对照 `solutions.md`。分层设计：
> - **A 概念题**：用自己的话讲清楚才算懂，背定义不算
> - **B 动手题**：一律"先预测 → 再执行 → 对照"，预测错了的就是你心智模型的漏洞
> - **C 挑战题**：迁移到新场景
>
> 标注 🕷 的题目改编自真实学员错题——人会在同一个坑里掉两次，提前打疫苗。
> 动手题需要 Linux 环境（WSL 即可）。

## A 概念题

**A1** 用一句话分别说清内核、shell、终端是什么，三者什么关系。
*考察点：三者分工；错答示例："shell 是内核的一部分"（方向反了）。*

**A2** a) `~` 和 `/` 分别指什么？b) 你在 `/home/leo`，执行 `cd ~/..` 后到哪？
*考察点：家目录 vs 根目录。*

**A3** 逐位解释权限串 `drwxr-xr--`，然后回答：要让"其他人"既能进入我的目录又能列出里面的文件，至少给哪些权限？
*考察点：权限串结构；目录 x=进入、r=列出的特殊语义。*

**A4** 文件 `/root/secret` 权限为 `---------- root root`，你是普通用户（可执行 sudo）。以下哪条能读到内容？为什么？
```bash
A)  sudo cat < /root/secret
B)  sudo cat /root/secret
```
*考察点：重定向由谁执行；权限检查针对打开文件的进程。*

**A5** 把 `ls -l / | tail -n 1` 整条翻译成一句人话，并说明 `|` 在其中扮演的角色。
*考察点：管道语义；能完整描述整条命令而不是只认得符号。*

**A6** 你在终端敲下 `hello` 回车，bash 按什么顺序寻找这个命令？为什么在当前目录放一个 `hello` 脚本后，裸敲 `hello` 依然找不到？
*考察点：命令解析顺序（别名→函数→内置→PATH）；PATH 不搜当前目录的安全设计。*

## B 动手题

**B1（权限实验）** 依次执行，**每条跑之前先写下预测**：
```bash
mkdir /tmp/permtest && cd /tmp/permtest
mkdir box && echo hello > box/inside.txt
sudo chown root:root box
sudo chmod 701 box        # drwx-----x

ls box              # 预测：列出？报错？
cat box/inside.txt  # 预测：能读到 hello 吗？
cd box              # 预测：进得去吗？进去后再 ls 呢？
```
*考察点：x/r/w 在目录上的真实语义；"其他人"视角的构造方法（chown root）。*

**B2（sudo 重定向实验）** `/etc/shadow` 在所有 Linux 上都只有 root 可读，现成的实验材料：
```bash
sudo head /etc/shadow       # 预测再跑
sudo head < /etc/shadow     # 预测再跑（这条大概率报错，解释为什么）
```
*考察点：`<` 由 shell 打开；两条命令唯一的区别是"谁负责打开文件"。*

**B3（shebang 全流程，官方练习改编）**：
```bash
mkdir -p ~/ex/missing && cd ~/ex/missing
touch semester
echo '#!/bin/sh' > semester                                   # 注意单引号
echo 'curl --head --silent https://missing.csail.mit.edu' >> semester
cat semester                # 确认两行都在
./semester                  # 观察报错，用 ls -l 解释原因
chmod +x semester
./semester                  # 成功后回答：shell 怎么知道该用 sh 跑它？
```
*考察点：`#` 开头的词是注释（不引号会写入空行）；x 权限与执行；shebang（#!）的机制——内核读前两字节，用其后路径的解释器执行脚本。*

**B4（两层模型实验）** 预测两条输出差异并解释：
```bash
cd /tmp && touch a1.txt a2.txt
echo a*
echo 'a*'
```
*考察点：glob 由 bash 展开；引号挡住 bash 层。*

## C 挑战题

**C1** 设计一个不超过 3 条命令的实验，向一个完全不懂的人**证明**"`>` 重定向是 shell 干的，不是命令干的"。（提示：想想怎么让一个"根本不存在的命令"也触发 Permission denied。）

**C2（官方练习 10 改编）** Linux 把很多硬件信息暴露在 `/sys` 下。写一条命令查看笔记本电池余量或 CPU 温度（WSL 没有真实硬件可跳过，改为：探索 `/sys/class/` 目录结构并说明你看到了什么设备类）。

**C3** 用"两层模型"分析：为什么 `echo *.txt` 在没有 txt 文件的目录里会打印字面量 `*.txt`，而在有 txt 文件的目录里打印文件名列表？（进阶：查 "nullglob" 这个词，说明 bash 的默认行为是什么。）
