# 模块 06：版本控制（Git）

> **来源**：MIT《The Missing Semester》2020 第 6 讲
> 中文：https://missing-semester-cn.github.io/2020/version-control/ ｜ 英文：https://missing.csail.mit.edu/2020/version-control/
> 进阶读物：Pro Git（官方免费）、Learn Git Branching（交互式）、Oh Shit Git!?!（事故救援）
> 本模块的实验场就是**你自己的 hpc-roadmap 仓库**——每个命令都能在真实历史上跑。

## 0. 一句话定位与学习路径

Git 是事实标准的版本控制系统（VCS）：给项目拍快照、记录"谁在何时为何改了什么"、多分支并行不互相踩。**Git ≠ GitHub**（前者是工具，后者是托管平台）。XKCD 有幅漫画："我先用 git 保存一下，然后炸了"——因为它抽象泄漏严重，所以本模块按官方路径**自底向上**学：先懂数据模型，命令就不再是咒语。

## 1. 数据模型（先懂这个，命令才有意义）

**快照而非差异**：每个提交保存整个目录树的完整快照（内部有共享优化，概念上如此）。三种对象：

| 对象 | 是什么 | 内容 |
|------|--------|------|
| **blob** | 文件内容 | 一组字节（不含文件名！） |
| **tree** | 目录 | 名字 → blob 或 tree 的映射 |
| **commit** | 一次快照 | 父提交（可多个）、作者、信息、指向顶层树 |

**寻址**：每个对象按内容的 **SHA-1 哈希**命名——内容变则哈希变，天然防篡改、可去重。亲手看一眼（在任意仓库）：

```bash
git cat-file -p HEAD^{tree}   # 看当前提交的顶层树（名字→哈希的映射）
git cat-file -p <某个blob哈希>  # 看文件原始内容
```

**引用（reference）**：人类可读的、**可变**的提交指针——`main` 就是一个引用。`HEAD` 是"你现在的位置"（通常指向当前分支）。

**历史是 DAG（有向无环图）**：提交不可变；"修改历史" = 创建新提交 + 挪动引用。合并提交有两个父辈——这就是分支图上的分叉与汇合。

**仓库的本质 = 对象库 + 引用集合**。懂了这句，所有命令都是"增删查对象和引用"的操作。

## 2. 暂存区（staging area）

Git 的独门设计：**下次快照装什么，由你说了算**。工作区（你正在改的）→ `git add` 挑选 → 暂存区（下次提交的内容）→ `git commit` 拍板。

用途：改了两个无关功能，分两次提交；bug 修复进提交、调试 print 不进。`git add -p` 可以按**代码块**挑（进阶必备）。

## 3. 基础命令

```bash
git init                      # 建仓库（数据都在 .git/ 目录里）
git clone <url>               # 复制远端仓库
git status                    # 三区状态总览（最该常敲的命令）
git add <file>                # 工作区 → 暂存区
git commit -m "消息"           # 暂存区 → 新提交
git log --all --graph --decorate --oneline   # 可视化 DAG 历史
git diff                      # 工作区 vs 暂存区
git diff --staged             # 暂存区 vs 上次提交
git diff <rev1> <rev2>        # 两个提交之间
git checkout <revision>       # 移动 HEAD（切分支/看历史版本）
```

## 4. 分支与合并

```bash
git branch            # 列分支；git branch <名> 新建
git checkout -b <名>   # 新建并切换（= branch + checkout）
git merge <分支>       # 把该分支合并进当前分支
```

- 无分叉时合并是 **fast-forward**（直接挪指针）
- 两边都改了同一处 → **冲突（conflict）**：文件里出现 `<<<<<<<`/`=======`/`>>>>>>>` 标记，手工编辑解决后 `add` + `commit`；`git mergetool` 可调图形工具
- `git rebase <分支>`：把当前分支的提交"摘下来"重放到新基线上——历史更线性，但改写过去（团队共享分支慎用）

## 5. 远程操作

```bash
git remote -v                     # 看远端配置
git push <remote> <本地>:<远端>    # 上传对象并更新远端引用
git branch --set-upstream-to=origin/main   # 绑定本地↔远端分支
git fetch                         # 只下载对象和引用，不动你的工作区
git pull                          # = fetch + merge
git clone --depth=1 <url>         # 浅克隆（只拿最新一提交，大仓库提速）
```

🕷 **实战案例（2026-09-04 推送排障三道墙，真实事故入档）**：

1. **全局 `url.insteadOf` 镜像重写**：所有 github.com 的 HTTPS 传输被改写到 ghproxy.net，镜像只支持拉取——push 退出码 0 却**什么都没推上去**（静默假成功）。教训：`git remote -v` 显示的地址也可能被重写，看 `.git/config` 原文才是真相
2. **直连 github.com:443 无限卡死**：出口被墙
3. **解法**：SSH 走 `ssh.github.com:443`（`~/.ssh/config` 配 HostName+Port），remote 用 `git@github.com:...` 形式——SSH 地址不触发 https 重写，且 443 端口的 SSH 通道存活

这个案例集齐了：remote 原理、config 分层（global vs repo）、URL 重写机制、SSH/HTTPS 两种传输协议——**一条事故顶十页文档**。

## 6. 撤销家族（危险等级必背）

| 命令 | 干什么 | 危险度 |
|------|--------|--------|
| `git commit --amend` | 修补上一次提交（内容或信息） | 🟡 改写最近提交，已推送的别 amend |
| `git restore <file>` | 丢弃工作区修改（回到暂存区/上提交状态） | 🔴 没提交的改动直接没了 |
| `git restore --staged <file>` | 把文件移出暂存区（不动内容） | 🟢 安全 |
| `git reset --hard <rev>` | 强制回到某提交（工作区一并清掉） | 🔴🔴 最危险的常用命令 |
| `git revert <rev>` | **生成一个"反向提交"**抵消旧提交 | 🟢 历史只增不改，协作安全 |
| `git stash` / `stash pop` | 把手头改动暂存抽屉/取出 | 🟢 切分支前的标准动作 |
| `git clean -fd` | 删除未跟踪的文件 | 🔴 无回收站语义 |

**reset vs revert 是考点**：reset **改写历史**（像没发生过），revert **追加历史**（新提交抵消旧的）。自己一个人的分支随便 reset；推送到共享分支后只能 revert。

## 7. 进阶速览

- `git bisect`：二分法找"哪个提交引入了 bug"（配合自动测试脚本是杀器）
- `git blame <file>`：每行最后由谁在哪次提交修改
- `git add -p`：按块交互式暂存
- `git rebase -i`：交互式整理历史（合并/重排/改信息）
- `.gitignore`：声明不追踪的文件（编译产物、大文件）；全局版 `git config --global core.excludesfile ~/.gitignore_global`

## 8. GitHub 与 Pull Request

向别人的项目贡献：fork → clone 自己的 fork → 开分支改 → push → 发 **Pull Request**（请求原仓库拉取你的分支）。GitLab/BitBucket 是同类平台。比赛队内协作也是这套：分支开发 → PR 互相 review → 合并。

## 9. 常见误区清单

1. Git ≠ GitHub：前者本地工具，后者托管平台
2. 提交的是**暂存区**不是工作区——忘了 add 就提交了个寂寞
3. `git push` 退出码 0 不等于推上去了（案例①的静默假成功）
4. reset 改历史、revert 加历史——共享分支用后者
5. `--hard` 会清工作区，按之前先 `git status` 和 `git stash`
6. 大文件/密钥一旦提交进历史就一直在历史里——`.gitignore` 要**提前**配

## 10. 与超算比赛的联系

- 赛题代码基线用 git 管理：每次优化一个提交，性能回退时 `git bisect` 秒级定位是哪次改动引入
- 队内分工：一人一分支，PR 合并——比赛仓库和软件工程同款流程
- HelloHPC 要求提交原始源码并接受 Code Review——干净的提交历史本身就是评分印象分
