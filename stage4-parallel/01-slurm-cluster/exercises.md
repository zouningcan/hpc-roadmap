# 模块 01 练习：集群与 Slurm

> 分层：A 概念 / B 动手 / C 挑战。无集群账号也能全做（考试本身就是"先会写，再上机"）。

## A 概念题

**A1** a) 登录节点和计算节点的分工各是什么？b) 为什么在登录节点跑重计算会被管理员杀掉（两条理由）？
*考察点：角色划分；共享资源的公地悲剧。*

**A2** a) 集群为什么必须有调度器，它解决什么问题？b) sbatch、srun、salloc 三者的使用场景分别是什么？
*考察点：排队动机；三种提交方式的选择。*

**A3** `#SBATCH -c 4` 这行：a) bash 怎么看待这行？b) Slurm 怎么看待这行？c) 一个单机多线程程序（如 OpenMP）不写这条会怎样？
*考察点：SBATCH 注释的双重身份；"核是申请来的"。*

**A4** module 系统解决什么问题？`module load gcc/11` 底层改了什么？为什么 sbatch 脚本里必须重新 load？
*考察点：环境模块 = 环境变量管理；作业环境不继承交互 shell。*

## B 动手题

**B1（写你的第一份作业脚本）** 场景：Graph500，单机 128 核，限时 40 分钟，需要先 `module load` 编译器再 `make` 再运行 `./graph500_reference_bfs_sssp 18`，输出带作业号。从零写一份完整 sbatch 脚本（8 行 `#SBATCH` 头 + 命令体）。
*考察点：作业脚本全要素（J/p/N/n/c/t/o + module + 命令）。*

**B2（读真实脚本）** 解释下面这份脚本每一行的含义，并找出**两个会出问题的地方**：
```bash
#!/bin/bash
#SBATCH -J wrf
#SBATCH -p kp_interact
#SBATCH -N 1
#SBATCH -n 1
#SBATCH -c 4
#SBATCH -t 01:00:00
#SBATCH -o wrf.log

cd $HOME/wrf/run
./run_wrf.sh
```
（背景：这是正式跑 WRF 算例，需要 128 核、运行约 12 分钟、编译已在别处完成）
*考察点：队列选错（interact vs run）、核数不足、时限、cd 用法。*

**B3（无集群模拟）** 在 WSL 里给自己的机器造一个"假 Slurm"体验：
```bash
# 没有调度器时，人肉模拟一个后台作业的生命周期
nohup bash -c 'for i in $(seq 1 300); do date >> job.log; sleep 1; done' &
jobs
tail -f job.log        # Ctrl-C 退出 tail（作业还在跑）
pgrep -af "date >> job.log" || pgrep -af bash
```
说明：这个 nohup 后台循环和 sbatch 提交的作业，**哪些方面等价**（脱离终端存活、输出进文件），**哪些方面不等价**（资源申请？排队？限额？）
*考察点：sbatch 的本质 = nohup 的"集群豪华版"——资源申请+排队+记账。*

## C 挑战题

**C1（考试读题训练）** 打开本地 `hello-hpc/hello-hpc-1st-main/` 里 WRF 和 Graph500 两题的 readme，回答：
1. 两个队列 `kp_interact` 和 `kp_run` 分别被两题用在什么环节？
2. Graph500 的 128 核上限对应 sbatch 脚本里哪几行的什么组合？
3. 两题的时限各是多少？分别对应 `-t` 的什么值？
4. Graph500 为什么强调"提交的脚本里不要重定向输出"？（想想评分靠什么读结果）
*考察点：从真实题面提取 Slurm 参数——考场上第一步就是读题。*

**C2（连接第 5 讲：考试第 1 题预演）** HelloHPC 第 1 题要求拿远程主机的 ED25519 公钥指纹（SHA256、无填充 Base64）。查资料回答：
1. 拿指纹的命令组合是什么（提示：`ssh-keyscan` + `ssh-keygen -lf`，管道连接）？
2. `-lf` 里 `l` 和 `f` 各什么意思？
3. 这和 `ssh` 首次连接时问你的 `fingerprint` 是同一个东西吗？为什么平台用"报指纹"而不是让你直接贴公钥全文？
*考察点：主机密钥验证（TOFU）、指纹 = 公钥的哈希摘要、防中间人。*
