# 模块 06 练习：版本控制（Git）

> 分层：A 概念 / B 动手 / C 挑战。🕷 = 高频易错。
> B 卷直接在 **hpc-roadmap 仓库**上做（先在 WSL 里 `git clone git@github.com:zouningcan/hpc-roadmap.git ~/hpc` 拿一份自己的工作副本），也可以另建玩具仓库。**做完把自己的实验仓库 force 回原状不影响主仓库。**

## A 概念题

**A1** Git 保存的是"差异"还是"快照"？blob、tree、commit 三种对象各存什么？文件名存在哪种对象里？
*考察点：数据模型；blob 不含文件名（名字在 tree 里）是经典细节。*

**A2** "仓库的本质 = 对象 + 引用"。引用是什么、和提交什么关系？`main` 和 `HEAD` 分别是什么？
*考察点：引用是可变的提交指针；HEAD 是当前位置。*

**A3** 暂存区（staging area）解决什么问题？没有它会发生什么？
*考察点：挑选"下次快照装什么"；无它则工作区所有改动被迫一次性全提交。*

**A4** `git reset` 和 `git revert` 的本质区别？什么时候绝对不能用 reset？
*考察点：改写历史 vs 追加反向提交；已推送到共享分支后。*

**A5** `git fetch` 和 `git pull` 的区别？为什么谨慎的人偏好 fetch？
*考察点：pull = fetch + merge；fetch 不动工作区，先看清远端变化再决定。*

**A6** 🕷 你 `git push` 后终端显示退出码 0、没有任何报错，但 GitHub 网页上什么都没有。列出至少 2 种可能原因（提示：本模块实战案例 + `git status` 会告诉你什么）。
*考察点：静默假成功的真实案例；push 后本地 status/branch -vv 的确认习惯。*

## B 动手题

**B1（在自己的克隆上读懂历史）**：
```bash
git clone git@github.com:zouningcan/hpc-roadmap.git ~/hpc && cd ~/hpc
git log --all --graph --decorate --oneline    # 画出 DAG
git log --oneline -- stage1-tools/06-git/notes.md   # 这个文件经过几次提交？
git blame stage1-tools/README.md | head -5    # 谁在哪次提交写的这几行？
```
回答：提交哈希前 7 位、你的仓库有几代提交、README 的某行最后修改于哪个提交。
*考察点：log 的过滤（按路径）、blame 的用法。*

**B2（暂存区实验）**：
```bash
echo "第一处改动" >> README.md
echo "第二处改动" >> drills/README.md
git status                # 两个文件都是 modified
git add README.md         # 只暂存一个
git status                # 预测：两者各显示在哪个区？
git diff                  # 预测：还剩哪个文件的差异？
git diff --staged         # 预测：显示哪个文件的差异？
git restore drills/README.md   # 丢弃未暂存的改动
git restore --staged README.md # 取消暂存
git restore README.md           # 也丢弃
git status                # 回到干净
```
*考察点：三区状态流转；restore 的两种用法。*

**B3（分支与合并全流程）**：
```bash
git checkout -b experiment
echo "实验内容" > test.txt && git add test.txt && git commit -m "add test"
git checkout main          # test.txt 还在吗？为什么？
git checkout experiment    # 又回来了？
git merge experiment       # 在 main 上合并（fast-forward）
git log --oneline -3
```
然后制造一次**冲突**：在两个分支上改同一行，merge，观察 `<<<<<<<` 标记，手工解决后 add + commit。
*考察点：分支切换时工作区跟着快照走；fast-forward；冲突解决全流程。*

**B4（stash 三步舞）**：
```bash
echo "改到一半的东西" >> README.md
git stash                  # 收进抽屉
git status                 # 干净了
git stash list             # 看抽屉
git stash pop              # 取回来
git diff                   # 改动还在
```
*考察点：stash 的使用场景（切分支前收摊）。*

**B5（amend 与 gitignore）**：
1. 提交一个故意写错的消息，用 `git commit --amend -m` 改好
2. `echo "*.log" > .gitignore` 并提交；再 `touch a.log`，`git status` 确认它**不再出现**在未跟踪列表
*考察点：amend 修最近提交；gitignore 生效验证。*

## C 挑战题

**C1（Learn Git Branching）** 完成 https://learngitbranching.js.org 的 "Main: Introduction Sequence" 全部关卡（约 8 关）。这是建立分支/引用直觉的最佳工具，比文字描述快十倍。

**C2（bisect 找坏蛋）** 自造场景：
```bash
git init ~/bisect-lab && cd ~/bisect-lab
# 依次提交 10 个版本：good..good..bad（在第 6 个提交里把输出从 "ok" 改成 "broken"）
# 写一个检测脚本 test.sh：grep -q broken out.txt 则退出码 1，否则 0
git bisect start HEAD <第一个good的哈希>
git bisect run ./test.sh     # 自动二分
```
验证它定位到的正是第 6 个提交。
*考察点：bisect + 自动化脚本的组合——性能回退排查的同款方法。*

**C3（别名与全局配置）** 给 `~/.gitconfig` 加：
```
[alias]
    graph = log --all --graph --decorate --oneline
    st = status
```
然后 `git graph` 应直接出图。再回答：`~/.gitconfig` 和仓库里 `.git/config` 谁优先？（结合实战案例：正是 global 层的 url 重写盖住了仓库配置的意图）
