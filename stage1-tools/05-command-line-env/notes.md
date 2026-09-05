# 模块 05：命令行环境（信号 / tmux / dotfiles / SSH）

> **来源**：MIT《The Missing Semester》2020 第 5 讲
> 中文：https://missing-semester-cn.github.io/2020/command-line/ ｜ 英文：https://missing.csail.mit.edu/2020/command-line/
> 本讲在实战教学中被拆进过"第 2 讲"，现已按官方结构归位。**信号、tmux、ssh 是超算队每天用的三大生存技能**，本模块包含已验证有效的教学案例（🕷）。

## 0. 一句话定位

管好你的"工作环境"：进程出问题用**信号**控制它、断网用 **tmux** 保住任务、配置用 **dotfiles** 版本化、连集群用 **ssh** 免密直通。

## 1. 信号（signals）：进程间的"软件中断"

Shell 通过 UNIX 信号与进程通信——内核替你"拍一下"进程的肩膀，进程暂停当前工作去处理信号。

| 信号 | 触发方式 | 语义 |
|------|----------|------|
| SIGINT | `Ctrl-C` | 中断请求，**可被程序捕获/忽略** |
| SIGQUIT | `Ctrl-\` | 退出（通常带核心转储），可捕获 |
| SIGTERM | `kill -TERM <PID>` | 优雅终止的通用选择，可捕获 |
| **SIGKILL** | `kill -9 <PID>` | **不可捕获、立即结束**（"必杀"，但可能留下孤儿进程与未清理资源） |
| **SIGHUP** | 终端/连接关闭时内核发送 | "挂断"（hangup，电话时代遗产）——见下文死亡链条 |
| SIGSTOP / SIGTSTP | `kill -STOP` / `Ctrl-Z` | 暂停进程 |

**为什么存在"可捕获"？** 程序可以在退出前保存现场、清理临时文件（Python：`signal.signal(signal.SIGINT, handler)`，官方演示里一个忽略 SIGINT 的程序只能用 SIGQUIT 停）。而 SIGKILL 就是不给机会——**先礼后兵的默认顺序：SIGTERM 讲道理，SIGKILL 砸门**。

🕷 **断线杀死任务的完整链条（实战补考过两次的知识点）**：

> 断网 → SSH 连接断 → 集群上的 sshd 发现客户端没了 → sshd 给你的登录 bash 发 **SIGHUP** → bash 死 → 挂在它下面的子进程（你的任务）被清理

没有任何东西"重启"——是你的 shell 被信号杀了，任务因进程树陪葬。重连拿到的是全新空白 shell。**解法有二：让任务免疫 SIGHUP（nohup），或让任务换爹（tmux）**。

🕷 **课堂增量：信号与"停止态"进程的正确模型（学长理论被学生实验推翻后修正）**

给一个 **STOP 态**（SIGSTOP/Ctrl-Z 暂停中）的进程发信号，它会怎样？分两类：

| 信号类别 | 停止态进程的下场 |
|----------|------------------|
| 被**捕获**（程序装了处理器）或被**阻塞**的信号 | 挂起为 pending，等 CONT 恢复后才处理 |
| **未捕获的致命信号**（如 sleep 收到的 SIGTERM——没装处理器，走默认处置=终止） | **立即终止进程，不等 CONT** |

实测：`sleep 10000 &` → `kill -STOP` 后再 `kill -TERM`，进程直接 `Terminated`。"信号会排队等进程醒来"只对前一类成立。

## 2. 作业控制（job control）

```bash
sleep 1000        # 前台跑，终端被占
Ctrl-Z            # 暂停（SIGTSTP），回到提示符
jobs              # 列出本会话任务：[1]+  Stopped  sleep 1000
bg %1             # 让 1 号任务在后台继续
fg %1             # 拉回前台
sleep 2000 &      # & 直接后台启动
kill -TERM %1     # 按任务号杀
pgrep -af sleep   # 按名字找 PID（-a 显示完整命令行 -f 匹配全命令）
pkill sleep       # 按名字杀
```

`nohup`：让命令**忽略 SIGHUP**（名字 = no hangup）；已运行的程序补救用 `disown` 把它移出本 shell 的任务表。

🕷 **课堂增量：`&` 的机制层（fork 但不 wait）**

`sleep 2000 &` 做的事：bash **fork 出子进程后不 wait 它**（不等它结束就打印任务号、交回提示符）。两套地址要分清：

- **任务号 `[1]`**：bash 会话内部的编号，`%1` 语法只在**本 shell** 有效——换个终端就用不了
- **PID**：全系统唯一，`kill 12345`、`pgrep` 用的是它

`Terminated`/`Done` 通知是**异步**的：bash 平时在等你敲命令，子进程死了它不会立刻插嘴，而是在打印**下一个提示符之前**捎带汇报——所以有时杀完敲个回车才看到通知。

**长任务方法谱系（同一目的：逃离 SIGHUP 杀伤半径）**：

| 方法 | 逃法 | 特点 |
|------|------|------|
| tmux | 任务挂到 tmux 服务器名下 | **能回去**：接回界面、实时看、可交互 |
| nohup / disown | 任务忽略 SIGHUP | 发射后不管，输出进日志，只能 tail |
| Slurm sbatch | 任务交给调度器，跑在计算节点 | 集群上的正式姿势（阶段 4 详讲） |

## 3. tmux：终端多路复用器

**三层结构**：会话（session，独立工作区）→ 窗口（window，像浏览器标签页）→ 面板（pane，像 Vim 分屏）。

**架构本质（🕷 实战已验证）**：tmux 是**客户端/服务器**结构，两个进程都跑在集群机器上、通过 socket 通信——**"服务器"是真进程，`ps aux | grep tmux` 能抓到**。你的任务挂在常驻的 tmux 服务器下，不挂在登录 bash 下：

```
sshd ── bash ── tmux客户端（取景器）     ← 断线只杀这一支
                    ⋮
     tmux服务器（常驻进程）── bash ── 你的任务   ← 照跑
```

**三板斧 + 常用键**（`<C-b>` = 按 Ctrl+b 松开再按）：

```bash
tmux new -s work    # 建会话起名（底部出现状态栏）
tmux ls             # 列会话
tmux attach -t work # 接回
```

| 快捷键 | 功能 | 快捷键 | 功能 |
|--------|------|--------|------|
| `<C-b> d` | 分离（会话照跑） | `<C-b> c` / `<C-b> N` | 新窗口 / 跳第 N 窗 |
| `<C-b> %` / `<C-b> "` | 垂直 / 水平分屏 | `<C-b> p` / `<C-b> n` | 前 / 后窗口 |
| `<C-b> 方向键` | 面板间移动 | `<C-b> z` | 当前面板全屏切换 |
| `<C-b> [` | 回卷查看输出 | `<C-b> ,` | 重命名窗口 |

比赛姿势：连上集群先进 tmux，开四个面板——跑 HPL、跑应用赛题、tail 日志、留着手敲。

## 4. 别名与 dotfiles

```bash
alias ll='ls -lh'      # 等号不空格、值加引号（老规矩）
alias mv='mv -i'       # 覆盖前确认，防手滑
\ls                    # 反斜杠临时绕过别名
```

别名只活在当前 shell——**持久化写进启动文件**。各工具的启动/配置文件都以点开头（dotfiles）：

| 文件 | 管什么 |
|------|--------|
| `~/.bashrc` | bash：别名、函数、PATH（每次开 shell 自动 source——模块 02 讲过机制） |
| `~/.vimrc` | Vim |
| `~/.gitconfig` | Git |
| `~/.ssh/config` | ssh 主机别名 |
| `~/.tmux.conf` | tmux |

**管理方法（官方推荐）**：所有 dotfiles 集中放一个 git 仓库，写脚本用**符号链接**（`ln -s`）链到目标位置。四大好处：易安装、可移植、可同步、变更可追溯。可移植技巧：`if [[ "$(uname)" == "Linux" ]]; then ... fi` 条件加载；`[include] path = ~/.gitconfig_local` 机制让机器差异不进主配置。

## 5. 远程机器：SSH

```bash
ssh user@host              # 登录
ssh user@host ls           # 不登录，直接执行命令并返回
```

**密钥认证（比赛第一天就要做的事）**：

```bash
ssh-keygen -o -a 100 -t ed25519 -f ~/.ssh/id_ed25519   # 生成密钥对
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host          # 装公钥（输最后一次密码）
ssh user@host                                            # 此后免密
```

原理：服务器检查 `~/.ssh/authorized_keys` 里有没有你的公钥。`-a 100` 放慢破解速度、`-t ed25519` 是现代算法。密钥可加 passphrase，配合 `ssh-agent` 避免重复输入。

**`~/.ssh/config`（把长命令变两个字母）**：

```
Host fdu
    HostName 10.0.0.5
    User mathlover
    Port 22
    IdentityFile ~/.ssh/id_ed25519
```

之后 `ssh fdu` 即可，scp/rsync 同样识别。支持通配符（`Host *.mit.edu`）。**别把这个文件公开**。

**端口转发（跑 Jupyter 的标准姿势）**：

```bash
ssh fdu -L 9999:localhost:8888
# 把服务器的 8888"拉"到本机 9999；本机浏览器开 localhost:9999 = 服务器上的 Jupyter
```

**传文件**：`scp` 简单粗暴；`rsync` 增量+断点续传（大模型权重、数据集必备 `--partial`）；花活：`cat 本地文件 | ssh host tee 远程文件`。

延伸：Mosh（弱网漫游）、sshfs（把远端目录挂载成本地路径）。

## 6. 常见误区清单

1. `kill -9` 不是默认首选——先 SIGTERM 讲道理，无响应再 -9
2. tmux 不是"防止断网"，是"任务挂在别的爹下面"；nohup 是"对 SIGHUP 装死"
3. 别名/函数改了 `.bashrc` 不 `source`（或重开终端）不生效
4. 端口转发的方向：`-L 本机端口:远端目标`，记"把远端服务拉到本地"
5. `&` 后台任务断线照样死（还挂在登录 bash 下且会被 SIGHUP 波及）——**长任务认准 tmux/nohup/sbatch**

## 7. 与超算比赛的联系

- 比赛全程：ssh + tmux 是工作底座，断电断网换电脑都不丢现场
- 集群上跑 HPL 六小时：没 tmux 等于裸奔
- `~/.ssh/config` + 密钥：多节点跳转（登录节点→计算节点）省大量时间
- 端口转发：集群上起 Jupyter/TensorBoard/Grafana 监控时必用
