# 模块 08：元编程（构建系统 / 依赖管理 / CI / 测试）

> **来源**：MIT《The Missing Semester》2020 第 8 讲
> 中文：https://missing-semester-cn.github.io/2020/metaprogramming/ ｜ 英文：https://missing.csail.mit.edu/2020/metaprogramming/
> 这里的"元编程"指**管理代码的流程**：怎么构建、怎么管依赖、怎么自动测试——不是"写生成代码的代码"。

## 0. 一句话定位

写出代码只是一半，另一半是让代码**可复现地构建、被自动测试、依赖不漂移**。HelloHPC 多道题要求你交 `build` 脚本/`compile.sh`——本模块就是它的正式名称。

## 1. 构建系统：make

**三要素：目标（target）、依赖（dependencies）、规则（recipe）**。核心价值：**依赖未变就不重建**——增量构建。

```make
paper.pdf: paper.tex plot-data.png
	pdflatex paper.tex

plot-%.png: %.dat plot.py
	./plot.py -i $*.dat -o $@
```

逐行解剖：

- 第一条规则：目标 `paper.pdf`，依赖 `paper.tex` 和 `plot-data.png`——**任一依赖比目标新**才重新执行下面的命令（缩进必须是 **Tab**，不是空格——历史遗留的头号 Makefile 坑）
- `make` 裸跑 = 构建第一条规则（默认目标）；`make plot-data.png` 指定目标
- **模式规则**：`%` 是通配符，`plot-foo.png` 自动匹配 `foo.dat`；`$@` = 目标名，`$*` = `%` 匹配的部分，`$<` = 第一个依赖
- 依赖是**传递的**：改了 `plot.py` → `plot-data.png` 过期 → `paper.pdf` 过期 → 一路重建；只改 `paper.tex` 则只重跑 pdflatex，不重跑画图

**伪目标（phony target）**：`clean` 这种不对应文件的动作目标：
```make
.PHONY: clean
clean:
	rm -f *.pdf *.png *.aux
```
不标 `.PHONY` 的隐患：目录里恰好有个叫 `clean` 的文件时，make 认为"clean 已是最新"而拒绝执行。

现代替代：CMake（C/C++ 大型项目标准，WRF 用它）、ninja、Bazel——但 make 的**依赖思想**是所有构建系统的共同祖先。

## 2. 依赖管理

- 各语言的仓库：apt（Ubuntu 系统包）、PyPI（pip）、npm（Node）、Cargo（Rust）
- **语义化版本号** `主.次.补丁`：
  - API 没变、修 bug → 补丁 +1
  - 向后兼容地**加**功能 → 次 +1
  - **不兼容**的改动 → 主 +1（Python 2→3 就是主版本号警告的现实案例）
- **锁文件**（lock file）：把">=1.2"钉死成"=1.2.5"——保证别人构建出和你完全一样的环境（复现性）
- **vendoring**：把依赖源码直接拷进项目——彻底掌控，代价是体积与更新负担

HPC 现实：赛题常要求"评测环境无网络"，依赖要么 vendoring 要么写进 build 脚本自己编译（HelloHPC 第 10 题 WRF 连 4 个依赖都从源码编）。

## 3. 持续集成（CI）

定义：**代码变动时自动运行的东西**。典型形态：仓库里放一个配置文件（GitHub Actions 是 `.github/workflows/*.yml`），声明触发规则（最常见：每次 push 跑测试），CI 系统起虚拟机执行并记录结果——失败通知你，通过亮绿徽标。

```yaml
# .github/workflows/lint.yml —— 每次 push 自动 shellcheck 所有脚本
on: [push]
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: sudo apt install -y shellcheck
      - run: shellcheck **/*.sh
```

## 4. 测试词汇表

| 术语 | 含义 |
|------|------|
| 测试套件（test suite） | 所有测试的总称 |
| 单元测试（unit test） | 微观：测单个函数/模块 |
| 集成测试（integration test） | 宏观：各组件拼起来能不能协同 |
| 回归测试（regression test） | 防"修过的 bug 复活"——每个修好的 bug 都该留下一个测试 |
| 模拟（mock） | 用假实现替换外部依赖（网络/磁盘），让测试可控可重复 |

比赛场景：优化前后**必须验证输出一致**（每题都有 verify/哈希校验）——本质就是回归测试：优化是"改代码"，正确性门槛靠"跑同一组算例比对结果"。

## 5. 常见误区清单

1. Makefile 的命令行缩进是 **Tab**——"missing separator" 报错九成是这个
2. 忘记 `.PHONY`，遇到同名文件时 clean 失灵
3. 依赖列表不全（头文件没写进依赖）→ 改了头文件不重编 → 玄学不一致
4. 没有锁文件的"能跑就行"——三个月后队友/评测机装不到同版本依赖
5. CI 只在"绿"的时候看——它红着的时候被无视，等于没有

## 6. 与超算比赛的联系

- HelloHPC 第 5 题交 `compile.sh`、第 6 题交 `build`、第 10 题从源码编 WRF——**构建脚本是评分的一部分**
- 优化循环 = 改代码 → 重编译 → 跑 verify → 对比性能：没有增量构建（make 思想），每次全量重来会浪费宝贵的比赛时间
- 评测无网络：依赖管理（vendoring/自编译）是硬约束不是风格问题
