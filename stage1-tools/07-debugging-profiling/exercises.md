# 模块 07 练习：调试及性能分析

> 分层：A 概念 / B 动手 / C 挑战。🕷 = 高频易错。
> B 卷需要 WSL；perf 相关若权限受限（容器/WSL1），标注即可跳过，改做 cProfile 部分。

## A 概念题

**A1** `time` 输出 real/user/sys 三个时间。a) 各是什么？b) real 远大于 user+sys 说明什么？c) 什么情况下 user+sys 会大于 real？
*考察点：三钟分解的诊断逻辑（等 IO vs 计算密集 vs 并行）。*

**A2** 追踪型（cProfile）和采样型（perf）剖析器的区别？各自的适用场景？
*考察点：开销与信息的取舍。*

**A3** 为什么"没 profile 就优化"是 HPC 第一戒律？用一句话说出 profile 循环的正确顺序。
*考察点：测量先于动手；record→找热点→优化→验证。*

**A4** strace 能回答什么 pdb 很难回答的问题？举一个典型场景。
*考察点：系统调用层的可见性（卡在哪个 syscall）。*

**A5** 🕷 你的程序输出结果错了。列出你的调试动作顺序（至少 4 步，含"读完整报错"和"最小复现"这两个我们实战反复验证过的动作）。
*考察点：调试方法论，而非背工具。*

## B 动手题

**B1（pdb 修 bug）**：
```bash
mkdir -p ~/ex/debug && cd ~/ex/debug
cat > buggy.py <<'EOF'
def bubble_sort(arr):
    for i in range(len(arr)):
        for j in range(len(arr) - 1):
            if arr[j] > arr[j + 1]:
                arr[j], arr[j + 1] = arr[j + 1], arr[j]
    return arr

print(bubble_sort([3, 1, 2, 5, 4]))
EOF
python3 buggy.py
```
输出看起来"也排好了"但算法低效（内层多扫了很多）。用 pdb 走一遍：
```bash
python3 -m pdb buggy.py
# b 3 （在内层循环行下断点）→ c → 到断点后 p i, j, arr → n 单步 → q 退出
```
贴出你在断点处 `p i, j, arr` 的输出，并回答：内层循环的边界应该是多少才能少做无用比较？
*考察点：pdb 断点/单步/查变量三板斧；顺带看出循环边界问题。*

**B2（shellcheck 实战）**：
```bash
cat > m3u.sh <<'EOF'
for f in $(ls *.mp3)
do
  echo "playing $f"
done
EOF
shellcheck m3u.sh 2>/dev/null || sudo apt install -y shellcheck
shellcheck m3u.sh
```
它报了什么？这个脚本的隐藏 bug 是什么（提示：含空格的文件名）？
*考察点：静态分析的价值；`$(ls)` 解析 ls 输出是经典反模式。*

**B3（time 分解实验）**：
```bash
time curl -s https://missing.csail.mit.edu > /dev/null    # 网络型负载
time grep -r "main" /usr/include > /dev/null              # CPU+IO 型负载
```
两次的 real/user/sys 形态有什么不同？各属于哪类瓶颈？
*考察点：读懂数字形态而不是背定义。*

**B4（cProfile 找热点）**：
```bash
cat > fib.py <<'EOF'
import sys
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)
print(fib(int(sys.argv[1]) if len(sys.argv) > 1 else 30))
EOF
python3 -m cProfile -s tottime fib.py 35
```
哪个函数是热点、被调用了多少次？然后加 `@functools.lru_cache` 装饰 fib，重新 profile——调用次数变成了多少？
*考察点：cProfile 的输出读法（tottime/callcount）；缓存对递归的降维打击。*

## C 挑战题

**C1（hyperfine 对比）** 用 hyperfine（`sudo apt install hyperfine`）对比 `fd -e md` 与 `find . -iname '*.md'`（fd 需另装）或 `grep -r vs rg`：
```bash
hyperfine --warmup 3 'rg pattern /usr/include' 'grep -r pattern /usr/include'
```
报告快多少倍，并解释 --warmup 3 为什么必要（联系"缓存冷热"）。
*考察点：科学基准测量的姿势。*

**C2（官方练习：lsof + kill）** 起一个服务找出并终结它：
```bash
python3 -m http.server 4444 &
lsof | grep LISTEN | grep 4444     # 找到 PID
kill <PID>
```
如果 lsof 输出太多，怎么精确过滤？（进阶：`ss -tlnp` 一条搞定）
*考察点：端口占用排查——服务器排障日常。*

**C3（官方练习：stress + 绑核观察）**：
```bash
stress -c 3 &            # htop 里观察 3 个满载核
taskset --cpu-list 0,2 stress -c 3 &   # 强制挤到 0 和 2 两个核
htop                      # 观察区别；taskset 在 HelloHPC 第5题里就是指定核心跑分用的
```
*考察点：CPU 亲和性的可视理解。*
