# 模块 08 练习：元编程（构建/依赖/CI/测试）

> 分层：A 概念 / B 动手 / C 挑战。🕷 = 高频易错。

## A 概念题

**A1** make 的三要素是什么？它的核心价值（相对手敲命令）是什么？
*考察点：目标/依赖/规则；增量构建。*

**A2** 解释这个模式规则里每个符号：`plot-%.png: %.dat plot.py` 配合 `./plot.py -i $*.dat -o $@`——`%`、`$*`、`$@` 各代表什么？`make plot-foo.png` 会执行什么命令？
*考察点：模式规则与自动变量。*

**A3** `.PHONY` 是干什么的？不写它，什么时候 `make clean` 会失灵？
*考察点：伪目标与文件名冲突。*

**A4** 语义化版本 `2.31.4`：修了个 bug、加了个新功能（兼容旧 API）、改了不兼容的 API，版本号分别怎么变？
*考察点：主/次/补丁的语义。*

**A5** 锁文件（lock file）解决什么问题？为什么"能跑就行、不锁版本"在比赛评测机上是个雷？
*考察点：复现性构建。*

## B 动手题

**B1（Makefile 增量构建全流程，官方演示复现）**：
```bash
mkdir -p ~/ex/make && cd ~/ex/make
cat > plot.py <<'EOF'
import sys
open(sys.argv[sys.argv.index('-o')+1],'w').write('# fake png from '+sys.argv[sys.argv.index('-i')+1])
EOF
cat > Makefile <<'EOF'
paper.pdf: paper.tex plot-data.png
	@echo "== build paper.pdf"
	@touch paper.pdf

plot-%.png: %.dat plot.py
	@echo "== run plot.py for $*"
	./plot.py -i $*.dat -o $@
EOF
```
按顺序执行并**预测每一步 make 的行为**：
```bash
make                    # 1. 预测：会发生什么？（提示：data.dat 存在吗）
touch data.dat && make  # 2. 预测：先跑哪条规则，再跑哪条？
make                    # 3. 预测：输出什么？
touch paper.tex && make # 4. 预测：plot.py 还会跑吗？为什么？
touch plot.py && make   # 5. 预测：这次呢？
```
*考察点：依赖传递与时间戳比较——make 的一切行为都从这两条推出。*

**B2（🕷 Tab 之坑）** 把 Makefile 里一条命令的缩进换成 4 个空格，`make` 报什么？这个报错名为什么？
*考察点：missing separator；Makefile 九成事故的源头。*

**B3（clean 伪目标）** 给上面的 Makefile 加：
```make
.PHONY: clean
clean:
	rm -f *.pdf *.png
```
执行 `make clean` 确认清理。然后实验：`touch clean`（造一个同名文件）+ 把 `.PHONY` 行注释掉，再 `make clean`——观察失灵现象，解释原因。
*考察点：伪目标防冲突的机制。*

## C 挑战题

**C1（官方练习 3：pre-commit 钩子）** 在任意 git 仓库配 pre-commit 钩子：提交前自动跑 `make`（或 shellcheck），失败则**拒绝提交**：
```bash
# .git/hooks/pre-commit （记得 chmod +x）
#!/bin/bash
make || { echo "构建失败，拒绝提交"; exit 1; }
```
测试：故意在脚本里制造一个错误，`git commit` 应被拦截。
*考察点：git hooks——把质量门禁搬进本地提交环节。*

**C2（官方练习 4/5：GitHub Actions）** 给自己的 hpc-roadmap 仓库加一个 workflow：每次 push 对所有 `.sh` 文件跑 shellcheck（notes 第 3 节的 YAML 模板可用）。push 一个含错误的脚本验证它变红，修复后变绿。
*考察点：CI 的完整闭环（触发→执行→反馈）。*

**C3（回归测试思维）** 你把矩阵乘优化了 3 倍，怎么证明它"没算错"？设计一个最小验证流程（提示：随机矩阵 + 与朴素实现对比 + 比较容差）。写下来——这和你以后每道比赛赛题要做的 verify 是同一件事。
