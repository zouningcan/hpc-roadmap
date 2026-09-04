# 模块 06 参考答案

## A1

**快照**。blob 存文件内容（纯字节，**不含文件名**）；tree 存目录结构（名字 → blob/tree 的映射）；commit 存父提交+作者+信息+指向顶层树的指针。**文件名存在 tree 里**——同一个内容在两个路径下共享同一个 blob。

## A2

引用（reference）是**人类可读的、可变的提交指针**，内容就是一个提交哈希。`main` 是分支引用（指向该分支最新提交）；`HEAD` 通常指向当前分支（间接指向你现在站在哪个提交上）。提交不可变，"前进" = 新提交 + 挪引用。

## A3

暂存区让你**挑选下次快照包含哪些改动**。没有它：工作区里所有未提交改动会被迫打包进同一次提交——修了一半的另一个功能、临时调试 print 全混进历史。有了它：add 挑选 → 分主题多次提交，历史干净可读。

## A4

- `reset`：**改写/移动历史**（把分支引用硬挪到目标提交，`--hard` 连工作区一起清）——像那几次提交没发生过
- `revert`：**追加一个反向提交**抵消目标提交的改动——历史只增不改
- 绝对不能 reset 的场合：**已经推送到共享分支的提交**（别人基于它工作，改写会造成历史分叉灾难）。此时只能 revert。

## A5

`fetch` 只把远端的对象和引用下载到本地（更新 `origin/main`），**不动你的工作区和本地分支**；`pull` = fetch + merge（直接合并进当前分支）。谨慎者偏好 fetch：先 `git diff main origin/main` 看清远端发生了什么，再决定怎么合。

## A6

至少两种：

1. **静默假成功**（实战案例①）：传输层被 `url.insteadOf` 重写到只读镜像，push"成功"实则被吞。排查：`cat .git/config` 看 remote 原文、`gh api repos/.../branches` 或网页核对
2. **推错了分支/远端**：本地在别的分支上，或 remote 指向别处；`git status` / `git branch -vv` 一眼看出"Your branch is ahead of/has no upstream"
3. （加分）认证降级：cred helper 返回了无效凭据但服务端 200 之类的极端情况——永远用"远端侧证据"（网页/API/ls-remote）确认推送结果，不迷信退出码

## B1

以实际仓库为准。示例判据：`git log --oneline -- stage1-tools/06-git/notes.md` 应显示本模块文件只在本轮提交中出现；`git blame` 的输出格式为 `哈希 (作者 日期 时间 行号) 内容`——它回答"这行最后由谁在哪次提交改的"，正好是 A6 里"谁写的"这类考古问题。

## B2 预测

- `git status`：README.md 显示 **Changes to be committed**（已暂存），drills/README.md 显示 **Changes not staged**（仅工作区）
- `git diff`（工作区 vs 暂存区）：只剩 drills/README.md 的差异（README 的工作区与暂存区一致了）
- `git diff --staged`（暂存区 vs 上次提交）：只有 README.md 的差异
- 两次 restore 后回到 clean working tree

## B3

- `git checkout main` 后 **test.txt 消失**——切换分支 = 把工作区换成目标分支的快照，该文件只在 experiment 的提交里
- 切回 experiment 它又出现（同理）
- merge 时 main 没有分叉 → fast-forward，`main` 直接指到新提交
- 冲突实验：同一行两边各改成不同内容，merge 报 CONFLICT，文件内出现 `<<<<<<< HEAD ... ======= ... >>>>>>> branch` 标记；编辑保留想要的版本、删标记、`git add 文件`、`git commit` 完成合并

## B4

stash 把工作区（与暂存区）改动收进栈式抽屉，工作区瞬间回到 HEAD 干净状态；`stash list` 显示 `stash@{0}`；`pop` 取出并从栈删除（`apply` 是取出但保留）。典型用途：改到一半要切分支修 bug。

## B5

1. `git commit --amend -m "正确的消息"` 生成**替换版**提交（哈希变了——所以已推送的提交不要 amend）
2. `.gitignore` 提交后，`touch a.log` 的 a.log 不再出现在 untracked——忽略规则只对**未跟踪**文件生效；已经被跟踪的文件要先 `git rm --cached` 移出

## C1

Learn Git Branching 的可视化提交树会直接展示：checkout -b 如何长出新指针、merge 如何产生双父提交、rebase 如何把提交"摘下重放"。通关标准：你能不看文字预测每条命令后树的形状。

## C2

关键命令序列：
```bash
git bisect start HEAD <good哈希>
git bisect run ./test.sh
# 结束后 git bisect reset 回到原分支
```
原理：Git 每次跳到区间中点提交、checkout 过去、跑你的脚本（退出码 0=good 非0=bad），对数次数定位首个 bad 提交。10 个提交只需 ~3 次测试；1000 个也只要 ~10 次。

## C3

`~/.gitconfig`（global）与 `.git/config`（仓库级）同名配置**仓库级优先**。但实战案例的教训是：`url.<x>.insteadOf` 这类**多值键**，仓库级加一条空值**盖不掉** global 的那条——两值并存，global 依旧生效（当时靠"更长前缀自映射"绕过）。结论：配置分层不复杂，**多值键的覆盖语义**是坑点。
