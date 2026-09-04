# 模块 11 练习：stage1 综合复习（结业考）

> 本模块练习 = **stage1 全阶段结业考**。A 卷快速问答（每题 30 秒内答出为达标）；B 卷综合动手（串联多模块）；C 卷开放题。
> 全部不查资料作答，答完对照 solutions.md，错的知识点回对应模块重学。

## A 快速问答（12 题，对应 notes 第 12 节清单）

**A1** bash 层和程序层各处理哪些符号？各举两个。
**A2** `find . -name '*.py'` 去掉引号后，glob 匹配到 0/1/多个文件分别什么后果？
**A3** `{echo` 为什么不是"开块"？`}` 独占一行是什么纪律？
**A4** 用"动词+名词"拆解：`ci(`、`d$`、`3dw`。
**A5** sed 提取日志字段的标准套路是什么结构（两个关键词）？
**A6** 写出断线杀任务链条（五环节）和 tmux 的解法原理。
**A7** source 与 `./` 的本质区别（一句话+C++类比）。
**A8** reset 和 revert 的区别？什么场合绝对禁用前者？
**A9** real=10s、user+sys=0.5s，瓶颈大概在哪类资源？
**A10** make 靠什么机制跳过不需要的重建？
**A11** 为什么网盘同步/RAID 不算备份？好备份三特征？
**A12** 四个随机词典词（66 bit）vs 8 位随机字母数字（48 bit），哪个抗离线爆破？

## B 综合动手（多模块串联）

**B1（01+02+04 串联：一条管道的自我修养）** 在你的 hpc 仓库里：
```bash
cd ~/hpc
# 1) 找出所有 exercises.md（模块02）
find . -name 'exercises.md'
# 2) 统计每个文件多少行（模块01 管道 + wc）
find . -name 'exercises.md' | xargs wc -l
# 3) 只留行数最多的前三（模块04 统计链）
find . -name 'exercises.md' | xargs wc -l | sort -nr | head -4
# 4) 把结果写进 /tmp/top3.txt 再用 Vim 打开看看（模块03）
```
逐步预测每条的输出形态再执行。第 3 步为什么是 `head -4` 不是 `head -3`？
*考察点：find/xargs/wc/sort 的组合拳 + 读懂 wc 的 total 行。*

**B2（05+06 串联：git 远端体检）**：
```bash
cd ~/hpc
git fetch                          # 只取不并
git status                         # 本地 vs 远端的状态
git log --oneline -3 origin/main   # 远端最新三条（不动本地）
git branch -vv                     # 本地分支与上游的绑定关系
```
解释 `branch -vv` 输出里 `[origin/main]` 和 `ahead 1` 这类标注的含义。
*考察点：fetch 的非破坏性；upstream 概念。*

**B3（02+07 串联：profile 一个 bash 循环）**：
```bash
cat > slow.sh <<'EOF'
total=0
for i in $(seq 1 10000); do
    total=$((total + i))
done
echo $total
EOF
time bash slow.sh
```
把 10000 改成 50000 再 time——耗时增长接近几倍？这符合"循环开销线性"的直觉吗？再换 Python 写同款循环对比（官方 Q&A 第 2 题的 bash 性能边界体验）。
*考察点：time 测量；bash 循环 vs Python 的量级差异。*

## C 开放题

**C1（给自己写一份"环境说明书"）** 用 Markdown（模块 10）写 `~/hpc/my-env.md`，内容包括：你的 WSL 发行版、git 版本、gh 账号、SSH 密钥位置与指纹、本仓库的 remote 地址、常用的三个 alias。这份文档本身就该进 git 管理。
*考察点：把自己环境的"关键事实"文档化——以后每台新机器/集群都值得一份。*

**C2（stage1 复盘）** 用 200 字回答：这 11 个模块里，哪三个知识点对你做"超算比赛"最重要？为什么？（没有标准答案，但要求具体到模块编号和机制，不许泛泛而谈）
