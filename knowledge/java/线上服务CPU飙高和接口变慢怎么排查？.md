# 线上服务 CPU 飙高和接口变慢怎么排查？

## 面试回答

Java 服务线上出现 CPU 飙高、接口 RT 上升或者整体服务变慢时，我一般不会一上来就盯某一段业务代码，而是按照“机器/进程 → JVM/Java 应用 → Linux 系统调用/内核”的顺序逐层下钻。

先用 `top` / `htop` 判断机器整体资源情况，并确认是不是当前 Java 进程导致 CPU 升高；然后优先进入 Arthas，通过 `dashboard` 看 JVM 总体状态，通过 `thread` 找高 CPU 或阻塞线程，通过 `trace` 判断某个接口具体慢在哪一层，通过 `profiler` 生成火焰图看整体 CPU 热点。

如果 Arthas 已经能定位到具体业务方法、锁、GC、DB/RPC 调用等问题，就直接进入对应问题排查；如果 Arthas 只能看到现象，或者怀疑系统调用、调度、I/O、内核锁竞争，则继续下钻到 `strace`、`perf`、`ftrace`。

## 核心排查链路

```text
Java 服务出问题
    ↓
先 top / htop 看机器和进程
    ↓
确认是不是 Java 进程的问题
    ↓
进入 Arthas
    ↓
dashboard
    ↓
先看 JVM / CPU / 内存 / GC / 线程总体状态
    ↓
thread
    ↓
找高 CPU 线程 / BLOCKED 线程 / 死锁
    ↓
trace
    ↓
看具体接口或方法内部到底是哪一步慢
    ↓
profiler
    ↓
如果问题是整体 CPU 热点，生成火焰图定位热点调用链
    ↓
如果已经定位
    ↓
结束，进入具体问题修复

如果定位不到，或者怀疑 Linux / 内核
    ↓
strace
    ↓
看系统调用分布，判断 futex / read / write / network / file 等方向
    ↓
perf
    ↓
看 CPU 热点函数、完整调用栈、硬件计数器
    ↓
ftrace
    ↓
深入内核，看调度、futex、I/O、内核函数调用链
```

## top 中 CPU 参数怎么看

执行：

```bash
top
```

经常会看到：

```text
%Cpu(s): 88.2 us, 7.1 sy, 0.0 ni, 4.1 id, 0.6 wa, 0.0 hi, 0.0 si, 0.0 st
```

这一整行是在说明：**CPU 的全部时间分别花在了哪里。**

通常：

```text
us + sy + ni + id + wa + hi + si + st ≈ 100%
```

由于显示时有四舍五入，手算可能差 0.1% 左右。

### us：user

表示 CPU 花在**用户态程序**上的时间比例。

对 Java 服务可以先粗略理解为：CPU 正在执行 Java/JVM 等用户态代码。

例如：

```text
us = 90%
```

说明 CPU 大量时间在执行应用程序，可能需要排查：

- 业务代码计算量大
- 死循环
- 自旋
- 序列化/反序列化
- 正则计算
- JVM 自身高 CPU 工作等

### sy：system

表示 CPU 花在**Linux 内核态**上的时间比例。

Java 程序进行系统调用，例如网络读写、线程调度、锁等待等，都可能进入内核。

```text
sy 很高
```

可能需要关注：

- 大量系统调用
- 网络包处理
- 上下文切换
- 锁/调度
- 内核相关开销

### id：idle

表示 CPU 的**空闲时间比例**。

这是判断 CPU 忙不忙最直观的字段。

```text
id = 80%
→ CPU 很闲

id = 4%
→ CPU 基本被打满
```

因此排查 CPU 时，可以先看 `id`：

```text
id 很低
→ CPU 确实很忙
```

### wa：iowait

表示 CPU 因为任务在等待 I/O，而暂时没有事情可做的时间比例。

例如线程正在等磁盘读取完成：

```text
Java
↓
读文件
↓
磁盘慢
↓
线程等待 I/O
```

此时可能出现：

```text
wa 很高
```

这不代表 CPU 算力不够，而更可能说明磁盘/存储 I/O 存在问题。

### ni / hi / si / st

初学阶段先知道含义即可：

- `ni`：调整过 nice 优先级的用户进程消耗的 CPU
- `hi`：硬件中断消耗的 CPU
- `si`：软件中断消耗的 CPU
- `st`：虚拟化环境中，被宿主机抢走的 CPU 时间

### CPU 参数快速判断

```text
先看 id
↓
CPU 到底忙不忙？

id 很低
↓
CPU 真忙

再看 us / sy / wa
↓
到底忙在哪里？
```

典型情况：

```text
us 90
sy 5
id 5

→ 应用程序 CPU 消耗高
→ Java 服务优先进入 Arthas
```

```text
us 20
sy 70
id 10

→ 内核态开销异常高
→ 可能继续查 strace / perf
```

```text
us 10
sy 5
wa 70
id 15

→ 大量时间在等待 I/O
→ 优先查磁盘 / iostat
```

## top 中进程参数怎么看

例如：

```text
PID   USER   %CPU   %MEM    VIRT    RES    COMMAND
18231 app    365.7   18.2   6.2g    2.9g   java
```

### PID

进程 ID，可以理解为进程的“身份证号”。

后续很多排查命令都需要 PID：

```bash
top -H -p 18231
jstack 18231
strace -p 18231
perf top -p 18231
```

### %CPU

表示这个进程消耗了多少 CPU。

Linux `top` 中单个多线程进程的 `%CPU` 可以超过 100%。

例如 4 核机器：

```text
1 核 ≈ 100%
2 核 ≈ 200%
4 核 ≈ 400%
```

因此：

```text
Java = 365.7%
```

可以粗略理解成这个 Java 进程正在占用约 3.65 个 CPU 核心。

注意区分：

```text
上面的 us / sy / id / wa ...
→ 描述整机 CPU 时间如何分配，总计约 100%

下面单个进程的 %CPU
→ 多核情况下可以超过 100%
```

### %MEM

表示这个进程占整台机器**物理内存**的比例。

例如：

```text
机器内存 = 16GB
%MEM = 18.2%
```

粗略对应约 2.9GB。

### RES

Resident Set Size，表示进程当前**真正驻留在物理内存 RAM 中的内存大小**。

例如：

```text
RES = 2.9g
```

可以先理解为：

```text
当前真正占用物理内存约 2.9GB
```

排查进程实际物理内存占用时，`RES` 和 `%MEM` 比 `VIRT` 更值得关注。

### VIRT

表示进程拥有的**虚拟地址空间大小**。

例如：

```text
VIRT = 8GB
RES  = 2GB
```

不能理解成这个进程真的吃掉了 8GB 物理内存。

更准确的初学理解是：

```text
虚拟地址空间中规划/映射了约 8GB
但当前真正驻留在物理内存中的约为 2GB
```

因此：

```text
判断真实物理内存占用
优先看 RES / %MEM
不要只看 VIRT
```

## load average 是什么

`top` 第一行常见：

```text
load average: 0.72, 0.85, 0.91
```

三个数字分别表示：

```text
0.72 → 最近 1 分钟
0.85 → 最近 5 分钟
0.91 → 最近 15 分钟
```

`load average` **不是 CPU 使用率**。

可以先粗略理解为：

> 当前有多少任务正在运行、等待 CPU，或者处于部分不可中断等待状态。

因此 load 必须结合 CPU 核数看。

例如 4 核机器：

```text
load = 1
→ 很轻松

load ≈ 4
→ 四个核心基本都有活干

load = 8
→ 4 个核心干活，同时还有大量任务等待
```

例如：

```text
4 核机器
load average: 1.2, 1.0, 0.8
```

整体比较轻松。

而：

```text
4 核机器
load average: 12.0, 9.5, 6.0
```

说明系统负载很高，而且从 15 分钟、5 分钟到最近 1 分钟还在持续上升。

## top 参数速记表

| 字段 | 最简单理解 |
|---|---|
| `us` | CPU 在跑用户态程序 |
| `sy` | CPU 在跑 Linux 内核 |
| `id` | CPU 在休息 |
| `wa` | CPU 因任务等待 I/O 而空闲 |
| `PID` | 进程身份证号 |
| `%CPU` | 这个进程吃了多少 CPU |
| `%MEM` | 占整机物理内存的比例 |
| `RES` | 当前真正驻留在物理内存里的大小 |
| `VIRT` | 进程的虚拟地址空间大小 |
| `load average` | 系统有多少任务正在运行/等待，需结合 CPU 核数判断 |

## 每个工具主要解决什么问题

### top / htop

负责回答：**到底是不是这台机器、这个 Java 进程在吃 CPU？**

先做全局判断，不要一开始就钻进 JVM。

### Arthas dashboard

负责回答：**JVM 当前总体状态怎么样？**

重点看：

- CPU
- JVM 内存
- GC
- 线程数
- Runtime 信息

### Arthas thread

负责回答：**哪个 Java 线程正在消耗 CPU，或者哪个线程在阻塞其他线程？**

常用：

```bash
thread -n 5
thread -b
thread <threadId>
```

### Arthas trace

负责回答：**某个接口为什么慢？时间具体花在哪个子调用？**

例如：

```text
createOrder 2000ms
    ├── queryDB 30ms
    ├── Redis 5ms
    └── callPayment 1950ms
```

这时重点就不是 CPU，而是 `callPayment` 对应的 RPC。

### Arthas profiler

负责回答：**整个 Java 进程的 CPU 时间主要消耗在哪些调用链上？**

适合 CPU 高但单靠 thread 不够直观时，通过火焰图寻找最宽的热点调用栈。

### strace

负责回答：**进程在频繁做什么系统调用？系统调用是不是异常慢？**

例如发现 `futex` 占比异常高，就要怀疑锁竞争；如果 `read/write` 异常，则向 I/O 方向继续排查。

### perf

负责回答：**CPU 时间具体消耗在哪些用户态/内核态函数？**

可以通过 `perf top` 看实时热点，也可以 `perf record` 后生成火焰图。

### ftrace

负责回答：**进入内核之后，到底发生了什么？**

通常是最后下钻工具，用于看内核函数调用树、调度延迟、futex、块设备 I/O 等问题。

## 一句话记忆

```text
Java 应用层优先 Arthas；
Arthas 定位不到，再从 strace → perf → ftrace 向 Linux / 内核层下钻。
```

或者：

```text
top 定进程
Arthas 定 Java 问题
strace 定系统调用方向
perf 定 CPU 热点
ftrace 追内核细节
```
