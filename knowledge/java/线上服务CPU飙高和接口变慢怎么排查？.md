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
