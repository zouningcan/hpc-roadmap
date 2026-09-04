# 模块 10：大杂烩（Potpourri）

> **来源**：MIT《The Missing Semester》2020 第 10 讲
> 中文：https://missing-semester-cn.github.io/2020/potpourri/ ｜ 英文：https://missing.csail.mit.edu/2020/potpourri/
> 收纳箱式一讲。其中 **daemon、API+jq、备份原则**三样对 HPC 实用度最高；其余（键盘重映射、窗口管理器等）按需取用。官方本讲无练习，练习为本仓库设计。

## 0. 一句话定位

前面十讲装不下的实用知识：后台服务怎么管、远程文件怎么"变本地"、API 怎么在命令行调、备份的正确姿势。

## 1. 守护进程（daemon）与 systemd

**daemon = 后台常驻进程**，名字约定以 `d` 结尾：`sshd`（SSH 服务）、`systemd` 本身、tmux 的服务器进程（模块 05 你已经亲手养过一个"类 daemon"）。

Linux 标准管理器是 **systemd**，交互入口 `systemctl`：

```bash
systemctl status          # 列出运行中的服务
systemctl start/stop/restart sshd
systemctl enable sshd     # 开机自启 / disable 关掉
```

一个服务单元文件（`/etc/systemd/system/myapp.service`）的解剖：

```ini
[Unit]
Description=我的应用
After=network.target          # 依赖顺序：网络就绪后再启动

[Service]
User=foo                      # 用哪个身份跑（别用 root 跑一切——最小权限）
ExecStart=python3 app.py      # 启动命令
Restart=on-failure            # 挂了自动拉起

[Install]
WantedBy=multi-user.target    # 哪种启动级别下启用
```

**周期性任务用 cron，不要为此写 daemon**：`crontab -e` 里一行 `分 时 日 月 周 命令`。

## 2. FUSE：用户态文件系统

正常只有内核能实现文件系统；**FUSE** 把这事搬到用户空间——普通程序就能定义"文件操作的行为"。

- **sshfs**：把远程目录挂载到本地路径，`ls`/`cat` 操作自动经 SSH 转发——远程文件"变成本地文件"
- rclone（挂 S3/网盘）、gocryptfs（挂载后才是明文的加密盘）、borgbackup（把去重压缩的备份挂载成目录浏览）

HPC 用法：集群上的大数据集挂到本地编辑器里看，不用来回 scp。

## 3. 备份的正确姿势

🕷 **同步 ≠ 备份**：Dropbox/网盘同步、RAID 镜像会把**删除、损坏、勒索加密**忠实地传播到每一份副本——它们保护硬件故障，不保护"你自己的错误"或攻击者。

好备份的三特征：**版本化**（能回到昨天/上周）、**去重**（省空间才能多版本）、**可验证的恢复**（没演练过 restore 的备份等于没有）。云上的数据（网盘、在线文档）也该有离线副本。

经典 **3-2-1 原则**：3 份数据、2 种介质、1 份异地。工具：rsync（增量同步）、borg/restic（去重+加密+版本化）、时间机器类方案。

## 4. 命令行调 API

API = 结构化的 URL。`curl` 发请求，**`jq`** 解析 JSON（模块 04 提过，这里是主场）：

```bash
curl -s 'https://api.weather.gov/points/42.36,-71.09' | jq '.properties.forecast'
# 手边有 gh 的话（更好用的认证+JSON 一条龙）：
gh api user --jq '.login'
gh api repos/zouningcan/hpc-roadmap/commits --jq '.[0] | .sha[0:7] + " " + .commit.message'
```

认证靠 **token**——"token 即身份"：拿到它的人就拥有它的全部权限（呼应模块 09 的最小权限案例）。

## 5. 通用 CLI flag 惯例（跨工具通用的"方言"）

| 惯例 | 含义 |
|------|------|
| `--help` / `-h` | 帮助 |
| `--version` / `-V` | 版本 |
| `-v` / `-vvv` | 详细输出，**可叠加**（更更更详细） |
| `--quiet` / `-q` | 安静模式 |
| `-` | 代表 stdin/stdout（如 `curl -s url \| tar -xz -C dir -` 里流式解压——不落盘） |
| `--` | **从此停止解析 flag**：`rm -- -r` 删的是名叫 `-r` 的文件，不是递归删 |

`--` 是救命符：手滑建了 `-rf` 这种名字的文件时，没有它你几乎删不掉。

## 6. 键盘重映射（Vim 用户的刚需）

Caps Lock → ESC/Ctrl（模块 03 强烈建议过）；更进一步：单击/长按不同含义（点=Esc，按住=Ctrl）、按键盘独立规则。工具：karabiner-elements（mac）、xmodmap/AutoHotkey（Win/Linux）、QMK（固件层）。

## 7. 其余速览

- **平铺窗口管理器**：键盘驱动的窗口布局，理念同 tmux 面板（i3/Sway）
- **VPN**：把对 ISP 的信任转移给 VPN 商；HTTPS 已加密大部分流量；劣质 VPN 比没有更糟
- **Markdown**：`*斜体*`、`**粗体**`、`# 标题`、`` `代码```、`[名](链接)`——本仓库全部文档都是它
- **Docker/VM/云**：`vagrant up`、容器化、租云主机（比赛训练没集群时租一台带 GPU 的）
- **Jupyter**：交互式笔记本（集群上跑时配合模块 05 的端口转发）
- **GitHub 工作流**：提 issue 报问题 vs fork→branch→PR 贡献代码

## 8. 常见误区清单

1. 网盘同步/RAID 当备份用——删除和勒索会同步传播
2. 从不演练 restore 的备份——纸面安全
3. 用 root 跑一切服务——最小权限（Service 文件里 User= 的意义）
4. 忘了 `-` 能当文件参数——大文件流式处理不落盘
5. 见到 `-rf` 名字的文件束手无策——`--` 停止解析

## 9. 与超算比赛的联系

- 集群上的监控/服务（Grafana、exporter）全是 daemon——`systemctl status` 是排障第一步
- `gh api + jq` 组合处理一切 JSON 输出（比赛平台 API、CI 状态）
- sshfs 把集群的赛题数据挂到本地，用顺手的编辑器看
- 训练环境：没集群时租云 GPU 主机 + tmux + 一切本课程技能
