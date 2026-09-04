# 模块 05 练习：命令行环境

> 分层：A 概念 / B 动手 / C 挑战。🕷 = 实战教学中真实错过的知识点。
> B 卷全程可在 WSL 完成，无需真实服务器。

## A 概念题

**A1** 🕷 写出"没用 tmux 时断网，任务死亡"的完整五环节链条（必须包含那个信号的名字），并说明：为什么说"没有任何东西重启了"？
*考察点：SIGHUP 传播链；进程树陪葬。*

**A2** a) SIGTERM 和 SIGKILL 的本质区别？b) 为什么默认建议先发 SIGTERM？c) 什么样的程序"杀不死"，只能 SIGKILL？
*考察点：可捕获性；优雅退出的意义。*

**A3** tmux 的"客户端/服务器"是什么意思？"tmux 服务器"是实体还是比喻？断线杀死的分别是哪个进程？用什么命令可以亲眼看到它？
*考察点：client/server 角色划分；ps 验证。*

**A4** 三个保活方法 tmux / nohup / Slurm sbatch，各自"逃离 SIGHUP"的方式有什么不同？各自的适用场景？
*考察点：三种方法的机制差异（换爹 / 装死 / 换机器）。*

**A5** 改完 `~/.bashrc` 后如何让它立刻生效？这个动作的本质是什么（模块 02 的知识）？
*考察点：source 语义在配置文件上的应用。*

## B 动手题

**B1（信号实验）**：
```bash
sleep 10000 &
jobs                          # 看到任务号
kill -TERM %1                 # 预测：任务会怎样？
sleep 10000 &
kill -STOP %2 ; jobs          # 预测：状态变成什么？
kill -CONT %2 ; jobs          # 预测：又变成什么？
pgrep -af sleep               # 确认进程状态
pkill sleep                   # 清场
```
*考察点：TERM/STOP/CONT 的手感；jobs 任务号引用。*

**B2（tmux 全流程，🕷 本模块核心实验）**：
```bash
tmux new -s work              # 建会话
# 在里面跑：top（或 while true; do date; sleep 1; done）
# 按 Ctrl+b 再按 d —— 分离
ps aux | grep tmux            # 亲眼确认 tmux 服务器进程还在
tmux ls
tmux attach -t work           # 接回，输出还在滚
# 再练：Ctrl+b % 分屏 / Ctrl+b 方向键切面板 / Ctrl+b c 新窗口 / Ctrl+b z 全屏切换
tmux kill-session -t work     # 收工
```
贴出 `ps aux | grep tmux` 里服务器那一行，并用一句话说明它为什么断线不死。
*考察点：三板斧；服务器进程的实体性。*

**B3（作业控制）**：
```bash
sleep 600                     # 前台占用终端
# 按 Ctrl-Z（预测：发生了什么？提示符回来了吗？）
bg                            # 放后台继续
jobs
fg                            # 拉回前台，再 Ctrl-C 结束
```
*考察点：Ctrl-Z 暂停（SIGTSTP）与 fg/bg 的配合。*

**B4（别名与持久化）**：
1. 定义 `alias ll='ls -lh'`，验证生效
2. 把它写进 `~/.bashrc`（`echo "alias ll='ls -lh'" >> ~/.bashrc`），`source ~/.bashrc`
3. **开一个新终端窗口**，敲 `ll`——解释为什么新窗口里也生效（机制层回答）
*考察点：别名生命周期；.bashrc 与新 shell 的关系。*

## C 挑战题

**C1（官方练习：pidwait 函数）** 编写函数 `pidwait`：接受一个 PID，等待该进程结束后返回（提示：`kill -0 PID` 在进程不存在时返回非 0——利用退出码轮询，配 sleep 避免空转）。
*考察点：`kill -0` 的探测语义；退出码思维；模块 02 函数语法的综合运用。*

**C2（官方练习：找出最常用命令做别名）** 用一条管道统计你最常用的 10 个命令：
```bash
history | awk '{print $2}' | sort | uniq -c | sort -nr | head -10
```
（解释每一段在干什么——模块 04 的复习）然后给第一名建别名。
*考察点：跨模块综合（history + awk + 统计三件套）。*

**C3（SSH 配置实战）** 手上暂时没有服务器也能做：
1. `ssh-keygen -t ed25519` 生成密钥对（一路回车），查看 `~/.ssh/` 下多了哪两个文件、谁是公钥谁是私钥、**哪个可以给别人哪个打死不能给**
2. 写一份 `~/.ssh/config` 模板（Host 名、HostName、User、Port 四项），说明每项作用
3. 解释 `ssh host -L 9999:localhost:8888` 干了什么、本机浏览器该访问哪个地址
*考察点：密钥对的方向性；config 结构；端口转发方向。*
