# 模块 08 参考答案

## A1

目标、依赖、规则。核心价值：**增量构建**——通过比较时间戳判断哪些目标过期，没变的绝不重跑；且依赖是传递的，改一处只重建受影响的部分。

## A2

- `%`：模式通配符，在目标和依赖里匹配同一段字符串（`plot-foo.png` 与 `foo.dat` 的 `foo` 对应）
- `$*`：`%` 匹配到的那部分（`foo`）
- `$@`：目标名（`plot-foo.png`）
- `make plot-foo.png` 执行：`./plot.py -i foo.dat -o plot-foo.png`

## A3

`.PHONY` 声明目标是"动作"而非文件。不写时，若目录里恰好存在名为 `clean` 的文件，make 检查目标 `clean` 与依赖（无）的新旧关系后认为"clean 已是最新"，**拒绝执行**——clean 失灵。

## A4

修 bug → 补丁 +1：`2.31.5`；兼容性新功能 → 次 +1：`2.32.0`；不兼容 → 主 +1：`3.0.0`。

## A5

锁文件把">=x.y 的范围要求"钉死成精确版本，保证**任何人在任何机器上构建出相同环境**。不锁版本：评测机今天装到的依赖版本与你开发机不同（新版本行为变化），你的代码在评测机上以你从没见过的方式失败——且你无法复现调试。

## B1 预测与解释

1. `make`：报错 `No rule to make target 'data.dat'`（plot-data.png 依赖 data.dat，而它不存在也没有规则能生成）
2. `touch data.dat && make`：先跑 `plot-data.png` 规则（`== run plot.py for data` + 生成文件），再跑 `paper.pdf` 规则——**依赖先就位**
3. 再次 `make`：`make: Nothing to be done for 'all'`（或 up to date）——一切最新，零工作
4. `touch paper.tex && make`：**plot.py 不跑**（data.dat/plot.py 没变，plot-data.png 仍新），只重建 paper.pdf——增量性
5. `touch plot.py && make`：plot-data.png 过期（plot.py 比它新了）→ 画图重跑 → pdf 也跟着重建——**依赖的传递性**

## B2

报 `Makefile:N: *** missing separator. Stop.`——make 认为该行不是规则命令（命令行必须以 Tab 开头）。历史遗留设计，无法配置更改；编辑器里开"显示空白字符"防手滑。

## B3

注释 `.PHONY` 且 `touch clean` 后：`make clean` 输出 `"clean" is up to date` 之类并**不执行 rm**——make 把目标 clean 和那个同名文件搞混了。恢复 `.PHONY` 后正常。结论：动作型目标一律 phony。

## C1

pre-commit 钩子退出非 0 = 拦截提交。测试路径：制造构建错误 → `git commit` 显示钩子输出且提交失败 → 修复 → 提交通过。要点：`.git/hooks/` 下的文件要 `chmod +x`；钩子不在版本控制里（要团队共享得用 pre-commit 框架或把钩子放进仓库再软链）。

## C2

标准 YAML 见 notes 第 3 节。红绿验证：push 一个 `foo=$(ls)` 之类的脚本 → Actions 页面出现红叉、日志指出 SC2045 → 修复重新 push → 绿勾。这就是 CI 的最小完整闭环。

## C3 参考流程：

```python
import numpy as np
A = np.random.rand(512, 512); B = np.random.rand(512, 512)
ref = A @ B                       # 朴素参考实现（可信但慢）
mine = my_matmul(A, B)            # 优化版
assert np.allclose(mine, ref, atol=1e-9)   # 容差比较（浮点不可能严格相等）
```

三要素：**随机输入**（防打表、覆盖边界）、**可信参考实现**（慢但对）、**容差比较**（浮点语义）。比赛 verify 与此同构——这就是"回归测试"思想在 HPC 的日常形态。
