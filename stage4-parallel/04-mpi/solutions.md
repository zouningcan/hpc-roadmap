# 模块 04 答案：MPI

## A 概念题

**A1** 同一份程序 = 同一个可执行文件；干不同的活 = 每个 rank 执行到的分支不同。分支的舵是 `MPI_Comm_rank` 拿到的编号——`if (rank==0) 做甲 else 做乙`。SPMD 的精髓：**程序唯一、数据各异、分支分活**。四行不同输出就是四个进程各自执行了同一行 printf，rank/size 的值不同。

**A2** 三要素：**源 rank、tag、通信器**（另加数据类型/个数要一致，否则数据解释错乱）。能对上的只有 a（源 0 ✓、tag 7 ✓）；b 的 tag=8 对不上；c 的类型 MPI_FLOAT 与收方 MPI_INT 不符（多数实现收不匹配类型直接报错）。考点：tag 是"信道号"，类型是"语言"，两者都算匹配的一部分。

**A3** 图见 notes 第 3 节。归属：Bcast 全员同份；Scatter 全员各一块；Gather 仅 root；Allreduce 全员同值。
Allreduce 强在**专用算法**：Reduce+Bcast 是两步（先归到 root 再群发），root 两次当瓶颈；Allreduce 的环形实现让流量均摊到所有进程、且两阶段可流水——进程越多优势越大。库提供它是为了让"人人要结果"这个高频场景不走弯路。

**A4** 对偶表：

| | OpenMP reduction | MPI Allreduce |
|---|---|---|
| 私有副本在哪 | 每个线程栈上的副本 | 每个进程自己的内存 |
| 谁合并 | 运行时在并行区结束时合并（共享内存直接读各副本） | 各进程收发消息互相合并（网络传输） |
| 合并的物理成本 | 共享内存写，近零 | 真实的消息/网络流量 |

概念同源（归约），物理载体不同（共享内存 vs 消息）——这就是两套并行世界的对偶轴心。

**A5** 集体性 = 所有组内进程**必须都调用**同一集合操作（顺序、参数一致）。8 个调 1 个忘：8 个进程永远阻塞在 Bcast 上等第 9 个——程序挂死，`mpirun` 也不报错，只能 Ctrl-C。OpenMP 对应情形：某线程没进 parallel 区域只是"没参与"，其余线程照常分完活正常结束——**MPI 的集体性是硬约束，OpenMP 的工作共享是软分工**。

## B 动手题

**B1** 4 行；顺序不固定（4 个独立进程谁抢到终端谁先打印）。`| sort` 后有序且方便 diff/对比——暴露的性质：**并行程序的输出/执行顺序是不确定的（non-determinism）**，除同步点外不应依赖进程间相对顺序。这也解释了并行调试为何困难。

**B2** 参考实现：
```c
int main(int argc, char **argv) {
    MPI_Init(&argc, &argv);
    int rank; MPI_Comm_rank(MPI_COMM_WORLD, &rank);
    int buf = 0; double t0 = MPI_Wtime();
    for (int i = 0; i < 100; i++) {
        if (rank == 0) {                       // 0 先发后收：双方交错，无死锁
            MPI_Send(&buf, 1, MPI_INT, 1, 0, MPI_COMM_WORLD);
            MPI_Recv(&buf, 1, MPI_INT, 1, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
        } else {
            MPI_Recv(&buf, 1, MPI_INT, 0, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
            buf++;                             // 乒乓：回来时数值+1，可验证走了两程
            MPI_Send(&buf, 1, MPI_INT, 0, 0, MPI_COMM_WORLD);
        }
    }
    if (rank == 0) printf("100 rounds in %.3f ms, final=%d\n",
                          (MPI_Wtime()-t0)*1e3, buf);
    MPI_Finalize();
    return 0;
}
```
配对原则：rank0 发→收、rank1 收→发，任何时刻一方 Send 必有另一方 Recv 在等。

**B3** 版本 1 **死锁**：0 号和 1 号同时 Send 后互相等对方 Recv（循环等待，饿死在第一条消息上）。修复：任一侧交换成先 Recv 后 Send。版本 2 **不死锁**：0 先 Recv、1 先 Send——1 的 Send 立即被 0 的 Recv 接住，随后 1 Recv、0 Send，天然交错。规律：**按 rank 奇偶交错收发方向**是 stencil 类应用的标准防死锁写法。

**B4**
1. 参考量级（1e9 次迭代、4 核 WSL）：serial ~2-4 s → 2 进程 ~一半 → 4 进程 ~1/4 加速比接近线性（计算密集，π 积分是 compute-bound，比 OpenMP 的 sum_array 干净——每次迭代还有除法）
2. 结果**末位有差异**：连续切块与交叉切块的局部和加入 Reduce 的顺序不同 → 浮点结合律 → 末位漂移。两种都对（误差都在截断误差量级），但按位不一致——与 OpenMP 归约同一课。
3. 自欺点：各 rank 打印的是**各自的局部和**，没有任何一步把它们真正相加——"人手一份局部结果"≠"全局结果"。正确收尾必须是 Reduce/Allreduce（或显式消息传递把局部和送到 0 号相加）。教训：**并行程序的结果必须经过显式的合并通信才存在**。

## C 挑战题

**C1**
```c
// 写法一：阻塞，按奇偶错开（偶数先发后收，奇数先收后发）
if (rank % 2 == 0) { Send(→(rank+1)%size); Recv(←(rank+size-1)%size); }
else               { Recv(←(rank+size-1)%size); Send(→(rank+1)%size); }
// 注意 N 为偶数时首尾衔接处也满足交错；特判 size==1 直接结束。
// 写法二：非阻塞，全体统一"先发后收再等待"
MPI_Request req;
MPI_Isend(&x, 1, MPI_INT, (rank+1)%size, 0, MPI_COMM_WORLD, &req);
MPI_Recv(&x, 1, MPI_INT, (rank+size-1)%size, 0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
MPI_Wait(&req, MPI_STATUS_IGNORE);
```
不死锁原理：写法一靠奇偶交错——任何时刻一半进程在 Send、另一半正好在 Recv；写法二靠 Isend 立即返回不阻塞——先发的消息由 MPI 后台缓冲/推进，Recv 正常接，无循环等待。**非阻塞是"不需要想交错"的通用解**。

**C2**
1. 偶数 rank 先发后收、奇数先收后发：任一"发"必有邻居的"收"配对（我的右邻是奇数，它正在收左邻=我）——全局看收发天然交错，无死锁。收到的边界值写入影子单元（halo cell：`u[0]`/`u[N+1]`），下一轮迭代直接用。
2. `MPI_Barrier(MPI_COMM_WORLD)`——保证所有人都完成本轮交换与迭代后才进下一轮（严格说：边界交换本身收发配对后已隐含邻居间同步；Barrier 是保险的全局对表点，视精度/性能需求取舍）。
3. hybrid：每节点 1 个 MPI 进程、节点内 64 OpenMP 线程 → Slurm `#SBATCH -N 2 -n 2 -c 64`（n=进程数=节点内 MPI 数，c=每进程核数给 OpenMP）。要点：**MPI 管跨节点（边界交换），OpenMP 管节点内（段内循环 parallel for）**，通信量从"每核都发"降到"每节点只发"。

**C3** 伪代码（size=4）：
```
# 阶段1：size-1 步，部分和沿环流动
sum = x[rank]
for step in 1..size-1:
    Send(sum → (rank+1)%size)      # 非阻塞
    Recv(tmp ← (rank-1+size)%size)
    sum += tmp                     # 攒别人的部分和
# 阶段1 结束时：rank r 手里的 sum = 从 r 开始逆时针 size 个 x 的和（全局和的一种"轮转分解"）
# 阶段2：把各自的总和再传 size-1 步，让人人拿到同一份
for step in 1..size-1:
    Send(sum → 右邻); Recv(sum ← 左邻)
```
对比"全发 root"：root 收 3 份发 3 份，root 链路流量 6 份、其他进程闲置；环形算法每进程每阶段只收发 size-1=3 条消息且**流量均匀摊在所有进程和所有链路上**——总时间从"瓶颈链路决定"变成"全网均摊"。NVIDIA NCCL 的 ring allreduce 是同款思想在 GPU 拓扑上的精调版——深度学习数据并行的梯度同步就靠它（CSE234 模块的伏笔）。
