# 模块 04 练习：MPI

> 分层：A 概念 / B 动手 / C 挑战。
> 环境准备（一次性）：`sudo apt install openmpi-bin libopenmpi-dev`——WSL 单机即可跑全部 B 卷。

## A 概念题

**A1** 解释 SPMD：四个进程跑同一份 hello.c，为什么打印出四行不同的话？"同一份程序"和"干同一份活"的区别是什么？靠什么机制分开各人的活？
*考察点：SPMD 本质；rank 的分活角色。*

**A2** 写出 MPI 消息匹配的三要素，并判断下面哪对消息能对上：
```
rank1: MPI_Recv(buf, n, MPI_INT, 0, tag=7, WORLD)
rank0: a) MPI_Send(buf, n, MPI_INT, 1, tag=7, WORLD)
       b) MPI_Send(buf, n, MPI_INT, 1, tag=8, WORLD)
       c) MPI_Send(buf, n, MPI_FLOAT, 1, tag=7, WORLD)
```
*考察点：源/tag/类型；消息匹配规则。*

**A3** 画出四个进程时 `MPI_Bcast`、`MPI_Scatter`、`MPI_Gather`、`MPI_Allreduce(SUM)` 的数据流向图（各一张小图，标注结果在谁手里）。并回答：`Allreduce` 比 `Reduce`+`Bcast` 强在哪？（一句话：为什么库要专门提供它）
*考察点：集合通信家族；All 前缀语义；专用算法的意义。*

**A4** OpenMP 的 `reduction(+:s)` 和 MPI 的 `MPI_Allreduce(..., MPI_SUM, ...)` 是对偶关系。指出两者的"私有副本"分别存在哪里、由谁合并。
*考察点：线程私有副本 vs 进程私有内存；共享内存合并 vs 消息传递合并。*

**A5** 集合通信的"集体性"指什么？9 个 rank 的程序里 8 个调了 `MPI_Bcast`、1 个忘调，会发生什么？这和 OpenMP 里一个线程忘进 parallel 区域的后果有何不同？
*考察点：集体阻塞；MPI 与 OpenMP 的容错差异。*

## B 动手题

**B1（首次点火）**：
```bash
mkdir -p ~/ex/mpi && cd ~/ex/mpi
cat > hello.c <<'EOF'
#include <mpi.h>
#include <stdio.h>
int main(int argc, char **argv) {
    MPI_Init(&argc, &argv);
    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    MPI_Comm_size(MPI_COMM_WORLD, &size);
    printf("hello from rank %d of %d\n", rank, size);
    MPI_Finalize();
    return 0;
}
EOF
mpicc hello.c -o hello
mpirun -np 4 ./hello          # 预测：几行？顺序固定吗？
mpirun -np 4 ./hello | sort   # 预测：这 why 有意义？
```
回答：输出顺序为什么每次可能不同？这暴露了并行程序的什么性质？
*考察点：进程独立性；输出交错与不确定性。*

**B2（乒乓：点对点入门）**：写 pingpong.c——rank 0 与 rank 1 互发一个 int 各 100 次（0→1→0 为一轮），最后一轮的消息带毫秒耗时。提示：双方都要 Recv 和 Send；想想谁先发才不死锁。
*考察点：Send/Recv 配对；死锁规避；计时。*

**B3（死锁门诊）**：以下两段代码哪些会死锁？为什么？给出各自的修复：
```c
// 版本1                       // 版本2
if (rank==0) {                 if (rank==0) {
  Send(→1); Recv(←1);            Recv(←1); Send(→1);
} else {                       } else {
  Send(→0); Recv(←0);            Recv(←0); Send(→0);
}                              }
```
*考察点：死锁的循环等待结构；配对交错原则。*

**B4（并行求 π）**：把 notes 第 4 节的 π 程序补全跑通（`mpirun -np 4`），然后：
1. 记录 1/2/4 进程耗时，算加速比
2. 把循环改为"连续切块"（rank i 算 [i*n/size, (i+1)*n/size) 段），再测——两种分法结果有差异吗？为什么？
3. 把 `MPI_Reduce` 错改成"每个 rank 都打印 local"再总加——逻辑上哪一步自欺欺人了？
*考察点：并行积分范式；分块方式与求和顺序（浮点结合律再现）；结果归约的正确姿势。*

## C 挑战题

**C1（ring 传环，mpitutorial 经典）**：N 个进程围成环，一个数从 rank 0 出发，沿 0→1→2…→N-1→0 走一整圈回到起点。要求：消息只用相邻传递（`MPI_Send(→(rank+1)%size)`），正确处理"先收后发"或"非阻塞先发再收"两种写法，两种都写出来并说明各自为何不死锁。
*考察点：环形通信结构；阻塞/非阻塞两种姿势的时序分析。*

**C2（考试预演：WRF 边界交换，stencil+MPI）**：一维热传导链均分给 size 个进程，每进程管一段；每轮迭代后，各进程要把自己的边界值发给左右邻居（rank 0 没有左邻，rank size-1 没有右邻，跳过即可）。实现迭代 100 轮，含：
1. 边界交换的收发顺序设计（偶数 rank 先发后收、奇数 rank 先收后发——为什么这样不会死锁？）
2. 每轮之后的全局同步点该用什么？
3. （口答）如果再加 OpenMP 把每段内的循环并行化，Slurm 脚本该怎么写 `-n/-c`？（hybrid 预演）
*考察点：halo/边界交换范式——WRF 真实通信模式的极简版；hybrid 概念落地。*

**C3（口答/伪代码：Allreduce 的环形算法）**：不用库，用点对点通信实现"4 进程各自有数 x_i，求 Σx_i 人手一份"。写出环形双阶段算法的伪代码（阶段1：每步把"自己攒的部分和"传给右邻、收左邻的，共 size-1 步；阶段2：把总和传一整圈）。估算它比"全发 root 加完再群发"好在哪。
*考察点：ring allreduce 思想——NCCL/深度学习训练的底层同款；流量均摊分析。*
