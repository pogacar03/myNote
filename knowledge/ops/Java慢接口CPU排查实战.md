# Java 慢接口 CPU 排查实战

> 目标：不要只背命令，而是按照一次真实事故的状态流转去学会“为什么下一步这么查”。
>
> 这篇笔记保留完整模拟 Terminal 输入输出，并在每一步补上判断依据。

---

# 一、事故场景

线上告警：

```text
服务：order-service
环境：production / Kubernetes
实例数：6

接口：
POST /api/order/submit

正常：
AVG RT ≈ 120ms
P99 ≈ 350ms

现在：
AVG RT ≈ 1.8s
P99 ≈ 5.6s

QPS：
平时 ≈ 800
现在 ≈ 850

错误率：
0.3%

告警：
部分用户反馈下单一直转圈
```

已经连接到一个异常 Pod：

```text
Connected to pod: order-service-7c8659bd68-k2x9m

Linux order-service-7c8659bd68-k2x9m 5.15.0
Java: OpenJDK 17
PID: unknown

[prod@order-service-7c8659bd68-k2x9m ~]$ █
```

---

# 二、第一步：`top` 看机器和进程

输入：

```bash
top
```

模拟输出：

```text
top - 00:53:41 up 37 days,  6:21,  0 users,  load average: 5.82, 5.41, 4.96
Tasks:  46 total,   2 running,  44 sleeping,   0 stopped,   0 zombie

%Cpu(s): 72.8 us,  6.4 sy,  0.0 ni, 20.1 id,  0.1 wa,  0.0 hi,  0.6 si,  0.0 st
MiB Mem :   8192.0 total,    512.6 free,   4587.2 used,   3092.2 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   2874.4 avail Mem

    PID USER      PR  NI      VIRT      RES    SHR S  %CPU  %MEM     TIME+ COMMAND
    217 app       20   0     10.8g     3.1g  28440 S  468.7  38.8   1823:41 java
      1 app       20   0      6.8m     3.9m   3180 S    0.0   0.0      0:01 tini
    302 app       20   0     11912     4060   3200 R    0.3   0.0      0:00 top
```

## 这一步怎么看？

关键看两层：

### 1. 整机 CPU

```text
us = 72.8%
sy = 6.4%
id = 20.1%
wa = 0.1%
```

这里说明：

- `us` 很高：CPU 大量消耗在用户态程序计算
- `wa` 很低：不像是磁盘 I/O 等待
- `id` 只剩 20%左右：CPU 明显繁忙

### 2. 哪个进程吃 CPU

```text
217 java 468.7%
```

说明 Java 进程 217 正在大量消耗 CPU。

> `%CPU` 可以超过 100%，因为多核机器上，一个进程可以同时占用多个核。

此时下一步的目标是：

> 找到 Java 进程 217 里面，到底是哪几个线程在吃 CPU。

---

# 三、传统链路：`top -H -p PID`

输入：

```bash
top -H -p 217
```

模拟输出：

```text
top - 00:54:22 up 37 days,  6:22,  0 users,  load average: 5.93, 5.47, 5.01
Threads: 186 total,   7 running, 179 sleeping,   0 stopped,   0 zombie

%Cpu(s): 73.4 us,  6.1 sy,  0.0 ni, 19.7 id,  0.1 wa,  0.0 hi,  0.7 si,  0.0 st

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
    486 app       20   0   10.8g   3.1g  28440 R  96.8  38.8  18:42.13 http-nio-8080-e
    491 app       20   0   10.8g   3.1g  28440 R  94.5  38.8  17:58.71 http-nio-8080-e
    503 app       20   0   10.8g   3.1g  28440 R  92.7  38.8  16:51.44 http-nio-8080-e
    509 app       20   0   10.8g   3.1g  28440 R  89.9  38.8  16:12.09 http-nio-8080-e
    518 app       20   0   10.8g   3.1g  28440 R  87.2  38.8  15:43.65 http-nio-8080-e
    527 app       20   0   10.8g   3.1g  28440 R   5.2  38.8   2:04.31 http-nio-8080-e
    229 app       20   0   10.8g   3.1g  28440 S   1.1  38.8   8:27.62 VM Thread
    235 app       20   0   10.8g   3.1g  28440 S   0.7  38.8   4:11.08 GC Thread#0
```

## 这一步怎么看？

可以发现：

```text
486  96.8%
491  94.5%
503  92.7%
509  89.9%
518  87.2%
```

而且这些线程都是：

```text
http-nio-8080-exec-*
```

所以当前判断是：

> 大量 CPU 是业务 HTTP 请求线程消耗的，不像是 GC 线程导致的。

接下来要回答：

> 线程 486 到底在执行哪段 Java 代码？

---

# 四、把线程 ID 转成十六进制

Linux `top -H` 里看到的线程 ID 是十进制：

```text
486
```

输入：

```bash
printf "%x\n" 486
```

输出：

```text
1e6
```

## 为什么要转十六进制？

因为 `jstack` 输出里通常用十六进制显示 JVM 线程对应的操作系统线程 ID：

```text
nid=0x1e6
```

所以映射关系是：

```text
top -H
线程 ID = 486
    ↓
十六进制 = 1e6
    ↓
jstack 中找 nid=0x1e6
```

---

# 五、`nid` 是什么？

`nid` 一般理解为：

> **Native Thread ID**

即：

> JVM 线程对应到操作系统原生线程的线程 ID。

所以 `nid` 的作用，就是帮你把：

```text
Linux 线程
```

和：

```text
Java 线程
```

对应起来。

---

# 六、`jstack`：看高 CPU 线程正在执行什么

输入：

```bash
jstack 217 | grep -A 30 "nid=0x1e6"
```

模拟输出：

```text
"http-nio-8080-exec-17" #87 daemon prio=5 os_prio=0 cpu=1128432.44ms elapsed=2876.31s tid=0x00007f8c4812a800 nid=0x1e6 runnable
   java.lang.Thread.State: RUNNABLE
        at java.util.regex.Pattern$CharPropertyGreedy.match(Pattern.java:4290)
        at java.util.regex.Pattern$Branch.match(Pattern.java:4734)
        at java.util.regex.Pattern$GroupHead.match(Pattern.java:4789)
        at java.util.regex.Pattern$Loop.match(Pattern.java:4898)
        at java.util.regex.Pattern$GroupTail.match(Pattern.java:4820)
        at java.util.regex.Pattern$BranchConn.match(Pattern.java:4698)
        at java.util.regex.Pattern$CharProperty.match(Pattern.java:3931)
        at java.util.regex.Matcher.search(Matcher.java:1728)
        at java.util.regex.Matcher.find(Matcher.java:745)
        at com.sf.order.util.AddressParser.normalize(AddressParser.java:84)
        at com.sf.order.service.impl.OrderServiceImpl.validateAddress(OrderServiceImpl.java:312)
        at com.sf.order.service.impl.OrderServiceImpl.submit(OrderServiceImpl.java:167)
        at com.sf.order.controller.OrderController.submit(OrderController.java:98)
        ...
```

## 这一步能判断什么？

状态流转：

```text
高 CPU 线程 486
    ↓
映射到 nid=0x1e6
    ↓
线程状态 RUNNABLE
    ↓
大量栈帧都在 java.util.regex
    ↓
落到 AddressParser.normalize(AddressParser.java:84)
```

当前最强怀疑：

> 这个请求线程正在 `AddressParser.normalize()` 里的正则匹配上大量消耗 CPU。

但一次 `jstack` 只能算一张快照，还不能因为抽中一次就直接拍板。

---

# 七、再抽一个高 CPU 线程验证

第二个高 CPU 线程：

```text
491
```

先转十六进制：

```bash
printf "%x\n" 491
```

输出：

```text
1eb
```

再查：

```bash
jstack 217 | grep -A 30 "nid=0x1eb"
```

模拟输出：

```text
"http-nio-8080-exec-22" #92 daemon prio=5 os_prio=0 cpu=1082143.71ms elapsed=2889.56s tid=0x00007f8c4813f800 nid=0x1eb runnable
   java.lang.Thread.State: RUNNABLE
        at java.util.regex.Pattern$Branch.match(Pattern.java:4734)
        at java.util.regex.Pattern$Loop.match(Pattern.java:4898)
        at java.util.regex.Pattern$GroupTail.match(Pattern.java:4820)
        at java.util.regex.Pattern$BranchConn.match(Pattern.java:4698)
        at java.util.regex.Pattern$CharProperty.match(Pattern.java:3931)
        at java.util.regex.Matcher.search(Matcher.java:1728)
        at java.util.regex.Matcher.find(Matcher.java:745)
        at com.sf.order.util.AddressParser.normalize(AddressParser.java:84)
        at com.sf.order.service.impl.OrderServiceImpl.validateAddress(OrderServiceImpl.java:312)
        at com.sf.order.service.impl.OrderServiceImpl.submit(OrderServiceImpl.java:167)
        at com.sf.order.controller.OrderController.submit(OrderController.java:98)
        ...
```

第二个高 CPU 线程也落在同一个方法。

证据变强：

```text
Java CPU 高
  ↓
多个 HTTP 线程 CPU 高
  ↓
线程 486 → AddressParser.normalize:84
线程 491 → AddressParser.normalize:84
```

---

# 八、`jstack`、stack、dump 到底是什么意思？

## 1. `jstack`

可以直接理解成：

> 把一个 Java 进程里，当前所有线程正在干什么，一次性打印出来。

例如：

```bash
jstack 217
```

本质上是在对 Java 进程 217 做一次：

```text
thread dump
```

---

## 2. stack 是什么？

`stack` 就是某个线程当前的方法调用链。

例如：

```text
OrderController.submit()
    ↓
OrderServiceImpl.submit()
    ↓
validateAddress()
    ↓
AddressParser.normalize()
    ↓
Matcher.find()
```

这就是这个线程当前的调用栈。

---

## 3. dump 是什么？

`dump` 可以理解成：

> 把程序某一时刻的内部状态完整“倒出来”，形成一个快照。

所以：

```text
thread dump
= 某一瞬间所有线程状态 + 调用栈的快照
```

可以把 `jstack` 想成拍照：

```text
Java 程序持续运行
      ↓
   jstack
      ↓
   咔嚓一下
      ↓
拍下这一瞬间所有线程的位置
```

如果连续几次看到：

```text
第一次：线程 A 在 AddressParser.normalize()
第二次：线程 A 还在 AddressParser.normalize()
第三次：线程 A 还是在 AddressParser.normalize()
```

就说明它不是“刚好路过”，而是很可能长期耗在这里。

---

# 九、发现 Java CPU 高后，是不是可以直接用 Arthas？

可以，而且实际排查中通常更快。

发现：

```text
top
  ↓
Java PID=217 CPU 很高
```

就可以直接：

```bash
java -jar arthas-boot.jar 217
```

然后：

```bash
thread -n 5
```

这一步直接帮你做了：

```text
找高 CPU Java 线程
    ↓
映射 Java 线程
    ↓
直接展示调用栈
```

所以实战推荐链路：

```text
有 Arthas：

top
 → Arthas
 → thread -n 5
 → trace / profiler
 → jad / watch
```

而传统链路：

```text
没有 Arthas / attach 失败：

top
 → top -H -p PID
 → 线程 ID 转十六进制
 → jstack
```

为什么还要学 `top -H + jstack`？

因为面试官很可能会问：

> “线上没有 Arthas 怎么办？”

这时候就要能把底层链路讲出来。

---

# 十、Arthas `thread -n 5`：确认热点线程

进入 Arthas：

```text
[arthas@217]$ thread -n 5
```

模拟输出：

```text
"thread_name" Id=87 cpuUsage=96.43% deltaTime=192ms time=1128432ms RUNNABLE
    at java.util.regex.Pattern$CharPropertyGreedy.match(Pattern.java:4290)
    at java.util.regex.Pattern$Branch.match(Pattern.java:4734)
    at java.util.regex.Pattern$Loop.match(Pattern.java:4898)
    at java.util.regex.Matcher.find(Matcher.java:745)
    at com.sf.order.util.AddressParser.normalize(AddressParser.java:84)
    at com.sf.order.service.impl.OrderServiceImpl.validateAddress(OrderServiceImpl.java:312)
    at com.sf.order.service.impl.OrderServiceImpl.submit(OrderServiceImpl.java:167)
    at com.sf.order.controller.OrderController.submit(OrderController.java:98)

"thread_name" Id=92 cpuUsage=94.87% deltaTime=189ms time=1082143ms RUNNABLE
    at java.util.regex.Pattern$Branch.match(Pattern.java:4734)
    at java.util.regex.Pattern$Loop.match(Pattern.java:4898)
    at java.util.regex.Matcher.find(Matcher.java:745)
    at com.sf.order.util.AddressParser.normalize(AddressParser.java:84)
    at com.sf.order.service.impl.OrderServiceImpl.validateAddress(OrderServiceImpl.java:312)
    at com.sf.order.service.impl.OrderServiceImpl.submit(OrderServiceImpl.java:167)
    at com.sf.order.controller.OrderController.submit(OrderController.java:98)

"thread_name" Id=104 cpuUsage=92.11% deltaTime=184ms time=1011872ms RUNNABLE
    at java.util.regex.Pattern$Loop.match(Pattern.java:4898)
    at java.util.regex.Pattern$GroupTail.match(Pattern.java:4820)
    at java.util.regex.Matcher.find(Matcher.java:745)
    at com.sf.order.util.AddressParser.normalize(AddressParser.java:84)
    at com.sf.order.service.impl.OrderServiceImpl.validateAddress(OrderServiceImpl.java:312)
    at com.sf.order.service.impl.OrderServiceImpl.submit(OrderServiceImpl.java:167)
```

这时候可以基本确认：

> 多个最耗 CPU 的 Java 请求线程都集中在 `AddressParser.normalize()`。

---

# 十一、`trace`：确认方法内部到底慢在哪里

输入：

```bash
trace com.sf.order.util.AddressParser normalize '#cost > 100'
```

模拟输出：

```text
Press Q or Ctrl+C to abort.
Affect(class count: 1 , method count: 1) cost in 128 ms, listenerId: 3

`---ts=2026-09-09 00:58:41.327;thread_name=http-nio-8080-exec-17;id=87
    `---[1864.317ms] com.sf.order.util.AddressParser:normalize()
        +---[0.021ms] java.lang.String:trim() #79
        +---[0.043ms] java.util.regex.Pattern:matcher() #82
        +---[1863.992ms] java.util.regex.Matcher:find() #84
        `---[0.087ms] java.util.regex.Matcher:replaceAll() #85

`---ts=2026-09-09 00:58:42.104;thread_name=http-nio-8080-exec-22;id=92
    `---[2147.662ms] com.sf.order.util.AddressParser:normalize()
        +---[0.018ms] java.lang.String:trim() #79
        +---[0.037ms] java.util.regex.Pattern:matcher() #82
        +---[2147.401ms] java.util.regex.Matcher:find() #84
        `---[0.091ms] java.util.regex.Matcher:replaceAll() #85
```

## 这一步怎么判断？

可以明确看到：

```text
normalize() 总耗时 ≈ 1.8 ~ 2.1s
        ↓
Matcher.find() ≈ 1.8 ~ 2.1s
```

几乎所有时间都花在：

```java
matcher.find()
```

所以现在定位到了：

```text
AddressParser.java 第 84 行
```

但注意：

> “84 行最耗时”不等于“84 行这句话本身就是最终根因”。

还要继续看：

1. 84 行到底用了什么正则？
2. 什么输入把这个正则打爆了？

---

# 十二、`jad`：看 JVM 当前加载的真实代码

输入：

```bash
jad com.sf.order.util.AddressParser
```

模拟输出：

```java
package com.sf.order.util;

import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class AddressParser {

    private static final Pattern ADDRESS_PATTERN =
        Pattern.compile(
            "^([\\u4e00-\\u9fa5A-Za-z0-9]+\\s*)+" +
            "(省|市|区|县|街道|路|号).*$"
        );

    public static String normalize(String address) {
        if (address == null) {
            return null;
        }

        String value = address.trim();                  // #79

        Matcher matcher =
            ADDRESS_PATTERN.matcher(value);             // #82

        if (matcher.find()) {                           // #84
            return matcher.replaceAll("$1");           // #85
        }

        return value;
    }
}
```

此时看到一个很可疑的正则：

```regex
([\u4e00-\u9fa5A-Za-z0-9]+\s*)+
```

特点：

```text
里面有 +
外面又套了一个 +
```

即嵌套量词。

开始高度怀疑：

> 正则灾难性回溯。

---

# 十三、`watch`：抓真实入参

现在要回答：

> 到底是什么 `address` 输入让 `Matcher.find()` 跑两秒？

输入：

```bash
watch com.sf.order.util.AddressParser normalize 'params[0]' -x 2
```

模拟输出：

```text
Press Q or Ctrl+C to abort.
Affect(class count: 1 , method count: 1) cost in 92 ms, listenerId: 4

method=com.sf.order.util.AddressParser.normalize location=AtExit
ts=2026-09-09 01:50:46.812; [cost=1876.34ms] result=@String[
广东深圳南山区科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园科技园XYZ
]

method=com.sf.order.util.AddressParser.normalize location=AtExit
ts=2026-09-09 01:50:48.097; [cost=2119.62ms] result=@String[
上海浦东新区张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江张江ABC
]
```

现在证据链完整：

```text
特殊超长地址输入
    ↓
进入嵌套量词正则
    ↓
前面大量字符都能匹配
    ↓
最后条件失败
    ↓
正则引擎不断尝试不同拆分方式
    ↓
大量回溯
    ↓
Matcher.find() 持续占用 CPU
    ↓
多个 HTTP 请求线程同时 RUNNABLE
    ↓
Java 进程 CPU 飙高
    ↓
接口 P99 飙升
```

---

# 十四、为什么是 `RUNNABLE`，不是 `WAITING` / `BLOCKED`？

因为这个问题不是线程在等待：

- 数据库
- Redis
- 网络
- 锁

而是正则引擎一直在做计算。

所以线程持续占用 CPU：

```text
RUNNABLE
```

这和 I/O 慢时常见的等待状态不同。

---

# 十五、最终完整排查链路

```text
接口慢
 ↓
top
 ↓
发现 Java CPU 高
 ↓
有 Arthas？
 ├─ 有
 │   ↓
 │ thread -n 5
 │   ↓
 │ 多个高 CPU 线程集中在同一调用栈
 │   ↓
 │ trace
 │   ↓
 │ 定位到 Matcher.find() #84 耗时 2s
 │   ↓
 │ jad
 │   ↓
 │ 看到嵌套量词正则
 │   ↓
 │ watch
 │   ↓
 │ 抓到触发问题的真实超长输入
 │
 └─ 没有 / attach 失败
     ↓
   top -H -p PID
     ↓
   找高 CPU TID
     ↓
   printf 十进制转十六进制
     ↓
   jstack PID | grep nid
     ↓
   定位热点 Java 调用栈
```

最终根因：

> 特殊长地址输入触发正则灾难性回溯，导致多个 HTTP 请求线程长期处于 RUNNABLE 状态并持续消耗 CPU，最终把 Java 进程 CPU 打高，引发接口 RT / P99 飙升。

---

# 十六、这次实战要真正记住的几个问题

## Q1：发现 Java CPU 高以后，可以直接进 Arthas 吗？

可以。

推荐：

```text
top
 → Arthas
 → thread -n 5
```

`top -H + jstack` 是没有 Arthas 时的底层备用链路。

---

## Q2：`jstack` 是什么意思？

```text
jstack PID
```

就是：

> 对一个 Java 进程做一次 thread dump，打印这一瞬间所有线程的状态和调用栈。

---

## Q3：dump 是什么意思？

> 把程序某一时刻的内部状态“倒出来”，形成快照。

所以 `thread dump` 就是线程快照。

---

## Q4：nid 是什么意思？

> Native Thread ID。

用于把 Java 线程映射到操作系统原生线程。

---

## Q5：为什么要把 486 转成 1e6？

因为：

```text
top -H
```

通常看到十进制线程 ID，而：

```text
jstack
```

里常以十六进制：

```text
nid=0x1e6
```

展示。

---

## Q6：为什么不能看到 `AddressParser.java:84` 就直接宣布根因？

因为那只能说明：

> 84 行是当前耗时热点。

还要继续通过：

```text
jad
```

看代码，通过：

```text
watch
```

看真实输入，最终才能闭环根因。

---

# 十七、一句话背诵版

> 慢接口先用 `top` 判断是不是 Java CPU 问题；如果 CPU 高，有 Arthas 就直接 `thread -n 5` 找高 CPU Java 线程和调用栈，没有 Arthas 就 `top -H` 找线程 ID、转十六进制后用 `jstack` 对应 `nid`。定位热点方法后用 `trace` 看内部耗时，用 `jad` 看线上真实代码，用 `watch` 抓真实入参，最后把“机器现象 → 线程 → 方法 → 代码 → 输入 → 根因”整条证据链闭环。
