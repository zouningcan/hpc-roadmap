# 模块 04：MPI 消息传递并行

> **来源**：[LLNL MPI 教程](https://hpc-tutorials.llnl.gov/mpi/)（Blaise Barney 经典）；[mpitutorial.com](https://mpitutorial.com/tutorials/)（有中文版，示例代码全）
> 配套先修：本模块的"集合通信"一节与 `03-openmp` 的 reduction 构成对偶；`01-slurm` 的 sbatch 是跑 MPI 的载体
> 环境提示：WSL 可装 OpenMPI（`sudo apt install openmpi-bin libopenmpi-dev`）全程实测，无需真集群

## 0. 一句话定位

MPI（Message Passing Interface，消息传递接口）= **多个进程各管各的内存，靠显式收发消息协作**。跨节点并行的唯一标准——Top500 上所有 MPI 应用横跨数十万核，WRF（HelloHPC 第 10 题）就是这么跑的。

## 1. 编程模型：SPMD 与通信器

**SPMD**（Single Program Multiple Data）：所有进程运行**同一份程序**，靠"我是谁"决定干哪份活——这是 MPI 的基本姿势：

```c
#include <mpi.h>
int main(int argc, char **argv) {
    MPI_Init(&argc, &argv);                      // MPI 世界开机
    int rank, size;
    MPI_Comm_rank(MPI_COMM_WORLD, &rank);        // 我是几号（0 起）
    MPI_Comm_size(MPI_COMM_WORLD, &size);        // 一共几个进程
    printf("hello from rank %d of %d\n", rank, size);
    MPI_Finalize();                              // 关机
    return 0;
}
```

- **通信器（communicator）**：一组互相能发消息的进程的"群"。`MPI_COMM_WORLD` 是默认全员大群——所有 MPI 程序的世界人口
- **rank**：进程在群里的编号（0 号常当"班长"/root）
- **size**：群的总人数

编译运行（WSL 实测可行）：

```bash
mpicc hello.c -o hello          # mpicc = 帮你链接 MPI 库的 gcc 包装
mpirun -np 4 ./hello            # 起四个进程跑（-np = number of processes）
# WSL 单机即可；超算上则写进 sbatch 脚本由 Slurm 起进程（模块 01 的 -n 128 就是它）
```

## 2. 点对点通信：Send / Recv

```c
// 0 号进程发，1 号进程收
if (rank == 0) MPI_Send(&data, 1, MPI_INT, 1, tag=0, MPI_COMM_WORLD);
if (rank == 1) MPI_Recv(&data, 1, MPI_INT, 0, tag=0, MPI_COMM_WORLD, MPI_STATUS_IGNORE);
```

六要素记法：**缓冲区、个数、类型、发给谁（rank）、标签（tag）、在哪个群**。tag 是同一对进程间多条消息的"信道号"——先发的 3 号消息不会被先收走。

**消息匹配三规则**：收方的（源、tag、通信器）都要对上才收。

🕷 **死锁四重奏**（MPI 新手第一大坑）：

```c
// 死锁版：双方都先 Send
// 0 号: Send→1 ; 1 号: Send→0    —— 两个 Send 都在等对方 Recv 才肯完成
// 0 号: Recv←1 ; 1 号: Recv←0    —— 安全
```

原则：**成对交错**（一收一发错开），或用非阻塞通信（`MPI_Isend/Irecv` + `MPI_Wait`）打破循环等待。

## 3. 集合通信：群发系统的标准动作

与 `01-slurm`/笔记里"集合通信"概念正式接轨——**整组进程共同参与的通信原语**：

```
Bcast(root=0)            Scatter(root=0)        Gather(root=0)        Allreduce(+)
P0: [ABCD] →全份ABCD     P0: [A|B|C|D]→A,B,C,D   A,B,C,D → P0:[ABCD]   1,2,3,4 → 人人10
```

| 原语 | 语义 | 结果归属 |
|------|------|----------|
| `MPI_Bcast` | 一对全（广播） | 全员同份数据 |
| `MPI_Scatter` | 一切多份，人一块 | 全员各一块 |
| `MPI_Gather` | 多收一，凑整 | 仅 root |
| `MPI_Allgather` | 交换后各持全份 | 全员 |
| `MPI_Reduce` | 归约合并（+/max/min…） | 仅 root |
| `MPI_Allreduce` | 归约 + 广播结果 | **全员（出镜率之王）** |
| `MPI_Barrier` | 集合点名（纯同步） | — |

记法钥匙：**All 前缀 = 结果人人有份**。

```c
// 求全局和——OpenMP reduction 的 MPI 表兄（对偶关系：线程私有副本 ↔ 进程私有内存）
double local = compute_my_share(rank), total;
MPI_Allreduce(&local, &total, 1, MPI_DOUBLE, MPI_SUM, MPI_COMM_WORLD);
// 此时所有 rank 手里的 total 都相等——不需要再手动广播
```

**为什么不手写循环**：库里的 Allreduce 用环形（ring）/树形（tree）算法把流量摊到全网（见"集合通信"笔记），进程一多，手写的"全发 root"会被 root 的网卡堵死。**用库，别造轮子**。

## 4. 经典范式：并行求 π（把整条链串起来）

积分 `π = 4∫₀¹ dx/(1+x²)`，把区间切成 size 份，每个 rank 算自己那一段，最后归约：

```c
long n = 1000000000;
double h = 1.0 / n, local = 0.0;
for (long i = rank; i < n; i += size)          // 交叉分块：rank 拿 i%size==rank 的份额
    local += 4.0 / (1.0 + (i+0.5)*h*(i+0.5)*h);
local *= h;
MPI_Reduce(&local, &pi, 1, MPI_DOUBLE, MPI_SUM, 0, MPI_COMM_WORLD);
if (rank == 0) printf("pi ≈ %.12f\n", pi);
```

三个要点：① 数据怎么分（`i += size` 交叉分 vs 连续切块——交叉分负载更均）；② 各算各的零通信；③ 一次 Reduce 收官。**能不通信就不通信，通信集中在收尾一次**——MPI 编程的美德。

## 5. 进阶速览（比赛用得到的四件）

- **非阻塞通信**：`MPI_Isend/Irecv` 立即返回，计算与通信**重叠**（发着消息的同时继续算）——WRF 每个时间步给邻居发边界数据时就是这么藏通信时间的
- **派生数据类型**：`MPI_Type_create_struct` 让你直接收发结构体/矩阵切片（如按列发数组），不用手动打包
- **进程拓扑**：`MPI_Cart_create` 把一维 rank 编号排成二维网格（stencil 应用给上下左右邻居编号）
- **hybrid MPI+OpenMP**：N 个节点 × 每节点 1 个 MPI 进程 × 节点内 128 OpenMP 线程——通信只发生在节点间（MPI 管跨节点），节点内共享内存零拷贝（OpenMP 管）。WRF/HPL 的真实形态，Slurm 参数对应 `-n 2 -N 2 -c 64`

## 6. 常见误区清单

1. `mpirun -np 4` 挂了先查环境：WSL 首跑要 `mpirun --allow-run-as-root`（root 时）或 `--oversubscribe`（进程数>核数）
2. 死锁排查：先画"谁给谁发、顺序如何"的时序图——九成死锁是双方都 Send
3. 集合通信是**集体动作**：一个 rank 忘调 `MPI_Bcast`，全员永远等它（对不上号=挂死）
4. 以为 rank 0 的数据"自动"人人可见：**没有共享内存**！root 上的数组要 Bcast/Scatter 出去，别人手里才有
5. `MPI_Send` 大数据可能"悄悄变成同步阻塞"（eager 协议只在小于阈值时用）——大消息卡在 Send 等接收方，别迷信"Send 立即返回"
6. OpenMP 思维直译 MPI：想"直接读别人的数组"——不存在的，要么消息传，要么该数据本就该人手一份

## 7. 与超算比赛的联系

- **WRF（HelloHPC 第 10 题）**：MPI 程序，128 核 = mpirun 起 128 个进程；`rsl.error.0000` 里 MPI 初始化失败是常见开局病
- **Graph500/RopeNet（128 核单节点）**：单节点也能用 MPI（共享内存上跑消息），但更优是 hybrid 或纯 OpenMP——选型题
- **HPL**：进程网格 P×Q、按 NUMA 绑进程，全是 MPI 工程学
- **与 Slurm 的分工**：Slurm 决定"哪些核哪些机器归你"，mpirun/srun 在这些资源上起进程——`01-slurm` 里 `srun` 的存在意义
