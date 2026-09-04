# 模块 10 练习：大杂烩

> 分层：A 概念 / B 动手 / C 挑战。🕷 = 高频易错。官方本讲无练习，以下为本仓库设计。

## A 概念题

**A1** 什么是守护进程？命名惯例是什么？举两个例子（其中一个要求来自你已学的模块）。
*考察点：后台常驻进程；d 后缀；sshd/tmux服务器。*

**A2** 🕷 为什么"网盘同步/RAID 不算备份"？它们和真备份保护的对象有什么不同？
*考察点：同步传播删除/损坏/勒索；备份要版本化+异地。*

**A3** `gh api user --jq '.login'` 里 jq 做了什么？为什么说"token 即身份"，这要求你对 token 采取什么态度？
*考察点：JSON 提取；最小权限与保管。*

**A4** `rm -- -r` 和 `rm -r` 的区别？`--` 这个惯例解决什么问题？
*考察点：停止解析 flag；处理手滑创建的怪名文件。*

**A5** FUSE 把什么从内核搬到了用户空间？sshfs 挂载后，你对挂载点的 `cat` 操作实际经历了什么？
*考察点：用户态文件系统；本地操作经 SSH 转发到远端。*

## B 动手题

**B1（认识你机器上的 daemon）**：
```bash
systemctl status | head -15        # 正在运行的服务一览
systemctl status ssh               # 找 sshd（WSL 里可能叫 ssh/sshd）
ps aux | grep -E 'sshd|systemd' | head -5
```
回答：你的 WSL 里有哪些以 d 结尾的进程？它们各自大概在干什么？
*考察点：daemon 就在你身边。*

**B2（curl/jq 实战，用 gh 的 API 通道）**：
```bash
gh api user --jq '.login, .public_repos'
gh api repos/zouningcan/hpc-roadmap --jq '.stargazers_count, .forks_count'
gh api repos/zouningcan/hpc-roadmap/commits --jq '.[] | .sha[0:7]' | head -3
```
第三条如果把 `.[]` 换成 `.[0]`，输出差在哪？（.[] = 遍历数组）
*考察点：jq 的数组遍历与字段提取。*

**B3（`--` 与 `-` 实验）**：
```bash
cd ~/ex && touch -- -r -weird
ls                  # 预测：怎么列出 -weird？
ls -- -weird        # 现在呢？
rm -- -r            # 删掉名叫 -r 的文件（不是递归！）
ls
```
*考察点：`--` 停止解析的现场体感。*

**B4（cron 周期任务）**：
```bash
crontab -e
# 加入一行（每分钟把时间戳写进日志）：
# * * * * * date >> /tmp/cron-test.log
# 等两分钟后：
cat /tmp/cron-test.log
crontab -r          # 用完删掉这个定时任务
```
五个字段分别是什么单位？"周期任务别写成 daemon"的理由？
*考察点：crontab 五字段；复杂度匹配。*

## C 挑战题

**C1（备份策略设计）** 为你的 `~/hpc` 学习仓库设计一套备份方案，要求满足 3-2-1 原则，并说明：为什么"推到 GitHub"只完成了其中一部分？（提示：Git 远端保护什么、不保护什么——比如你手滑 force push 之后）
*考察点：3-2-1；同步/托管 ≠ 完整备份（force push 会覆写远端）。*

**C2（sshfs 或替代实验）** 有条件的话（WSL 需另装）：`sshfs user@host:/remote/path ~/mnt` 挂载后 `ls ~/mnt` 看远端文件，用完 `fusermount -u ~/mnt` 卸载。做不到就写：sshfs 与 `scp/rsync` 的本质区别是什么（文件级拷贝 vs 协议级挂载）？
*考察点：挂载语义——远端文件系统"长在"本地目录树上。*

**C3（systemd 服务实战）** 把一个简单程序变成系统服务：写 `/etc/systemd/system/hello.service`（用 python3 -m http.server 8000 当 ExecStart），`systemctl daemon-reload && systemctl start hello`，浏览器/curl 验证 8000 端口，看 `systemctl status hello`，最后 stop + disable。贴出 status 输出里的关键行（Active、CGroup）。
*考察点：service 单元三段结构 → 实际运行。*
