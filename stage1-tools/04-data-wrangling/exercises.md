# 模块 04 练习：数据整理

> 分层：A 概念 / B 动手 / C 挑战。🕷 = 易错高发点。
> B 卷全部基于自造数据集，无服务器也能做。先预测再执行。

## A 概念题

**A1** 解释正则 `^user ([a-z]+) port [0-9]+$` 的每一段含义。它匹配什么格式的行？
*考察点：锚点、字符类、重复、空格的字面匹配。*

**A2** a) 为什么 `uniq -c` 之前必须 `sort`？b) `sort -nk1,1` 的 `-n` 和 `-k1,1` 各干什么？
*考察点：uniq 只合并相邻；数字序 vs 字典序（`9` 和 `10` 谁大）；按列排序。*

**A3** `sed 's/a/b/'` 不加 `-E` 时，`(invalid |authenticating )?` 这段会怎样？加 `-E` 后呢？
*考察点：BRE 与 ERE 的区别。*

**A4** awk 里 `$1` 和 bash 脚本里的 `$1` 分别指什么？awk 的 `NR` 和 `NF` 呢？
*考察点：同符号不同语境（列 vs 参数）；行号与列数。*

**A5** 一列数字求和，`paste -sd+ | bc -l` 是怎么工作的？（paste 干了什么，bc 收到了什么）
*考察点：工具组合的理解——把"列"变成"算式文本"。*

## B 动手题

**B1（自造数据集 + 官方流水线复现）**：
```bash
mkdir -p ~/ex/wrangle && cd ~/ex/wrangle
cat > ssh.log <<'EOF'
Jan 12 03:14:07 host sshd[102]: Disconnected from authenticating user root 10.0.0.5 port 4242 [preauth]
Jan 12 03:14:09 host sshd[103]: Disconnected from invalid user admin 10.0.0.9 port 5150 [preauth]
Jan 12 03:15:01 host sshd[110]: Disconnected from authenticating user leo 192.168.1.3 port 2222
Jan 12 03:15:33 host sshd[115]: Disconnected from invalid user admin 10.0.0.9 port 5151 [preauth]
Jan 12 03:16:02 host sshd[120]: Disconnected from authenticating user root 10.0.0.5 port 4300
Jan 12 03:16:44 host sshd[125]: Disconnected from invalid user test 172.16.0.8 port 3306 [preauth]
EOF
```
a) 用 sed 提取每行的**用户名**（root/admin/leo/test），输出六行用户名
b) 统计每个用户名出现次数（谁被攻击者最爱猜？）
c) 按次数从多到少排序，只留排名第一的用户名
*考察点：官方大正则的本地复现；sort|uniq -c；-k1,1 与 -r 的组合。*

**B2（字典词频，官方练习 2 简化版）** 用 `/usr/share/dict/words`（WSL 没有就 `sudo apt install wamerican` 或换任意词表文件）：
a) 找出所有以 `s` 结尾的词的**数量**
b) 统计词表中**首字母**的出现频率，按多到少排序取前 5
*考察点：锚点 `^`；`cut -c1` 或 `awk '{print substr($0,1,1)}'`；统计三件套。*

**B3（awk 三连）** 对下面数据（粘贴成 ls -l 风格）：
```
4096 docs
1024 notes.txt
2048 a.c
512 main.py
```
a) 打印文件名（第 2 列）
b) 只打印大小 ≥ 1024 的文件名
c) 用 BEGIN/END 算所有大小的总和
*考察点：域访问、条件块、BEGIN/END 计数器模式。*

## C 挑战题

**C1（官方练习 3）** 解释 `sed 's/a/b/' input.txt > input.txt` 为什么会毁掉文件（分两步说明谁在什么时候清空了它），并给出两种正确做法。
*考察点：`>` 的打开即截断；sed -i / 临时文件+mv。*

**C2（官方练习 1）** 完成 https://regexone.com 的交互式正则教程（约 15 节），截图或记录最后一关的通过状态。这是正则入门公认最佳路径。

**C3（非贪婪实战）** 造一行 `key=value key2=value2 disconnected from real user bob extra tail`，用 `.*` 贪婪提取和 `perl -pe 's/.*?from //` 非贪婪提取各跑一遍，对比输出差异并解释。
*考察点：贪婪匹配何时咬人。*
