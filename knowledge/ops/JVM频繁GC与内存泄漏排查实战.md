# JVM 频繁 GC 与内存泄漏排查实战

> 目标：通过一次完整线上事故，理解如何从 `top` 发现 JVM 异常，再通过 `jstat`、`jcmd`、Heap Dump、MAT 一步步定位到“对象为什么回收不掉”。
>
> 这篇笔记保留完整模拟 Terminal / MAT 输入输出，并在每一步解释为什么这么查。

---

# 一、事故场景

线上某 Java 服务出现：

```text
服务：payment-service
环境：production / Kubernetes

现象：
- 接口 RT 周期性毛刺
- CPU 每隔一段时间冲高
- 监控显示 GC Pause 增多
- JVM 内存占用持续偏高
```

已经进入异常 Pod。

---

# 二、第一步：`top` 看机器和 Java 进程

输入：

```bash
top
```

模拟输出：

```text
top - 11:23:18 up 18 days,  4:12,  0 users,  load average: 3.42, 2.96, 2.51
Tasks:  54 total,   2 running,  52 sleeping,   0 stopped,   0 zombie

%Cpu(s): 41.8 us,  5.7 sy,  0.0 ni, 51.9 id,  0.2 wa,  0.0 hi,  0.4 si,  0.0 st

MiB Mem :   8192.0 total,    394.8 free,   6128.7 used,   1668.5 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   1472.3 avail Mem

    PID USER      PR  NI      VIRT      RES    SHR S  %CPU  %MEM     TIME+ COMMAND
    624 app       20   0     11.4g     5.6g  29184 S  286.4  70.1   931:28 java
      1 app       20   0      6.8m     3.8m   3100 S    0.0   0.0     0:01 tini
    713 app       20   0     11912     4108   3224 R    0.3   0.0     0:00 top
```

## 这一步怎么看？

关键线索：

```text
Java CPU = 286.4%
RES = 5.6G
机器总内存 = 8G
```

说明 Java 进程：

```text
既占了较多 CPU
又占了大量内存
```

但注意：

> `top` 只能让你开始怀疑 JVM / GC，不能证明 GC 有问题。

因为 `top` 看不到：

- Young GC 次数
- Full GC 次数
- Eden / Survivor / Old 使用率
- GC 总耗时

所以下一步必须看 JVM GC 指标。

---

# 三、`jstat -gcutil`：看 GC 是否频繁

输入：

```bash
jstat -gcutil 624 1000
```

含义：

```text
624
= Java PID

1000
= 每 1000ms 输出一次
```

模拟输出：

```text
  S0     S1      E       O      M     CCS    YGC     YGCT    FGC    FGCT     CGC    CGCT     GCT
  0.00  72.14   88.27   91.83  96.42  93.11   842   34.921     7    8.412      26    1.834   45.167
  0.00  74.02   96.51   92.06  96.42  93.11   842   34.921     7    8.412      26    1.834   45.167
  0.00  69.38   21.73   92.41  96.42  93.11   843   34.987     7    8.412      26    1.834   45.233
  0.00  71.62   76.94   92.88  96.42  93.11   843   34.987     7    8.412      26    1.834   45.233
  0.00  68.91   18.55   93.26  96.42  93.11   844   35.058     7    8.412      26    1.834   45.304
  0.00  70.48   84.37   93.71  96.42  93.11   844   35.058     7    8.412      27    1.902   45.372
  0.00  67.85   23.11   94.16  96.42  93.11   845   35.131     7    8.412      27    1.902   45.445
  0.00  66.92   91.44   94.73  96.42  93.11   845   35.131     7    8.412      27    1.902   45.445
  0.00  12.08    7.26   67.31  96.42  93.11   846   35.204     8    9.694      27    1.902   46.800
  0.00  65.17   72.89   68.02  96.42  93.11   846   35.204     8    9.694      27    1.902   46.800
```

---

# 四、怎么看 `jstat` 输出？

## 1. Young GC 很频繁

观察：

```text
YGC
842
842
843
843
844
844
845
845
846
```

短短几秒：

```text
842 → 846
```

同时 Eden：

```text
88%
96%
↓ Young GC
21%

76%
↓ Young GC
18%

84%
↓ Young GC
23%
```

状态流转：

```text
Eden 很快被填满
    ↓
触发 Young GC
    ↓
Eden 被清理
    ↓
很快再次填满
    ↓
再次 Young GC
```

当前判断：

> 程序对象分配速度很快，导致 Young GC 频繁。

但 Young GC 频繁本身不一定是内存泄漏，还要继续看 Old 区。

---

## 2. Old 区持续上涨

观察：

```text
O:
91.83%
92.06%
92.41%
92.88%
93.26%
93.71%
94.16%
94.73%
```

Old 区一路升高。

然后：

```text
FGC:
7 → 8
```

同时 Old：

```text
94.73%
   ↓ Full GC
67.31%
```

说明：

```text
Old 区越来越满
    ↓
接近 95%
    ↓
触发 Full GC
    ↓
Old 降到 67%
```

所以 JVM 已经出现明显老年代压力。

---

## 3. 为什么 Full GC 后 Old 仍然 67% 很可疑？

如果 Full GC 后：

```text
Old 94% → 10%
```

说明大量对象其实可以正常回收。

但现在是：

```text
94% → 67%
```

说明 Full GC 以后还有大量对象仍然存活。

当前怀疑：

```text
方向 1：
业务确实存在大量长期存活对象

方向 2：
本应释放的对象仍被引用
→ 内存泄漏
```

尤其如果后续变成：

```text
67%
→ 75%
→ 85%
→ 95%
→ Full GC
→ 70%
→ 80%
→ 95%
→ Full GC
```

这种循环锯齿，就非常危险。

---

# 五、`jstat` 常见字段怎么记？

```text
S0 / S1
= Survivor 区使用率

E
= Eden 区使用率

O
= Old 老年代使用率

M
= Metaspace 使用率

CCS
= Compressed Class Space 使用率

YGC
= Young GC 次数

YGCT
= Young GC 总耗时

FGC
= Full GC 次数

FGCT
= Full GC 总耗时

GCT
= GC 总耗时
```

排查时最常盯：

```text
E
O
YGC
FGC
YGCT
FGCT
```

---

# 六、下一步：到底是什么对象占着内存？

输入：

```bash
jcmd 624 GC.class_histogram
```

模拟输出：

```text
624:
 num     #instances         #bytes  class name (module)
-------------------------------------------------------
   1:       1856421      356432832  [B (java.base@17)
   2:       1325814      212130240  java.lang.String (java.base@17)
   3:        842716      101125920  com.sf.payment.model.PaymentRecord
   4:        801294       96155280  com.sf.payment.cache.PaymentContext
   5:        723118       57849440  java.util.HashMap$Node
   6:        318482       50957120  java.util.HashMap
   7:        269184       43069440  java.util.ArrayList
   8:        184293       29486880  java.util.concurrent.CompletableFuture
   9:         86241       18628056  java.lang.Long
  10:         51844       12442560  com.sf.payment.dto.PaymentRequest
  11:         36492        9341952  java.util.concurrent.ConcurrentHashMap$Node
  ...

Total       6492814     1219847424
```

这里最值得关注的业务类：

```text
PaymentRecord
842,716 个
≈ 101 MB

PaymentContext
801,294 个
≈ 96 MB
```

但注意：

> 一次 histogram 只能告诉你“现在活着这么多对象”，不能直接证明这些对象泄漏。

---

# 七、为什么要重复跑 histogram？

下一步应该隔一段时间再跑：

```bash
jcmd 624 GC.class_histogram
```

如果看到：

```text
PaymentRecord
842,716
    ↓
1,120,000

PaymentContext
801,294
    ↓
1,080,000
```

而且经历 GC 后仍持续上涨，就说明：

> 这些对象很可能被长生命周期引用持有，没有被正常回收。

所以：

```text
一次 histogram
= 看对象分布

多次 histogram 对比
= 看对象增长趋势
```

---

# 八、Heap Dump：进入真正的内存泄漏定位

如果确认对象持续增长，需要进一步回答：

> 到底是谁一直引用这些对象？

模拟输入：

```bash
jmap -dump:live,format=b,file=/tmp/heap.hprof 624
```

模拟输出：

```text
Dumping heap to /tmp/heap.hprof ...
Heap dump file created [4876249132 bytes in 6.842 secs]
```

生成：

```text
/tmp/heap.hprof
≈ 4.8GB
```

---

# 九、生产环境为什么不能随便 `jmap -dump:live`？

这是一个非常重要的线上注意点。

`live` 的含义是：

> 只导出仍然存活的对象。

为了确定哪些对象是 live，对 JVM 来说通常意味着需要进行 GC / 堆遍历。

大堆导出还会产生：

```text
STW 风险
+ CPU 开销
+ 磁盘 IO
+ 大量磁盘空间占用
```

所以线上应该先评估：

- 是否可以在副本 / 从节点上操作
- 磁盘空间是否足够
- 是否允许停顿
- 是否优先使用已有 OOM 自动 Dump

> 不要在生产高峰期看到内存高就无脑 `jmap -dump:live`。

---

# 十、MAT：为什么先看 `Dominator Tree`？

Heap Dump 放到 Eclipse MAT 后，常见入口：

```text
Leak Suspects Report
Dominator Tree
Histogram
Top Consumers
```

当前问题是：

> 谁持有大量对象，导致它们释放不了？

所以先看：

```text
Dominator Tree
```

模拟输出：

```text
Class Name                                              | Shallow Heap | Retained Heap | Percentage
----------------------------------------------------------------------------------------------------------------
com.sf.payment.cache.PaymentContextCache                |        64 B  |    2.31 GB    |   48.7%
  java.util.concurrent.ConcurrentHashMap                |        64 B  |    2.28 GB    |   48.1%
    java.util.concurrent.ConcurrentHashMap$Node[]       |    8.00 MB   |    2.24 GB    |   47.3%

com.sf.payment.model.PaymentRecord                      |   101.2 MB   |   684.5 MB    |   14.4%

java.lang.String                                        |   212.1 MB   |   436.8 MB    |    9.2%

byte[]                                                  |   356.4 MB   |   356.4 MB    |    7.5%

java.util.concurrent.CompletableFuture                  |    29.5 MB   |   121.7 MB    |    2.6%
```

最刺眼的是：

```text
PaymentContextCache
Retained Heap = 2.31 GB
≈ 整个堆 48.7%
```

---

# 十一、Shallow Heap 和 Retained Heap 的区别

## Shallow Heap

> 对象自己本身占多少内存。

例如：

```text
PaymentContextCache 自己
可能只有几十字节
```

## Retained Heap

> 如果这个对象被回收，连带可以一起释放多少内存。

例如：

```text
PaymentContextCache
    ↓
ConcurrentHashMap
    ↓
80 万个 PaymentContext
    ↓
PaymentRecord / String / byte[]
```

所以：

```text
Shallow Heap 很小
但 Retained Heap 可以非常大
```

这正是 MAT 中定位“大对象持有者”的关键。

---

# 十二、`Path To GC Roots`：为什么这些对象回收不掉？

接着看：

```text
Path To GC Roots
```

通常选择：

```text
exclude weak/soft references
```

模拟结果：

```text
GC Root
└── java.lang.Class
    └── com.sf.payment.cache.PaymentContextCache
        └── static CACHE
            └── java.util.concurrent.ConcurrentHashMap
                └── table[]
                    └── ConcurrentHashMap$Node
                        └── value
                            └── com.sf.payment.cache.PaymentContext
                                └── paymentRecord
                                    └── com.sf.payment.model.PaymentRecord
```

证据链已经很清楚：

```text
GC Root
   ↓
PaymentContextCache.class
   ↓
static CACHE
   ↓
ConcurrentHashMap
   ↓
PaymentContext
   ↓
PaymentRecord
```

所以：

> Full GC 回收不掉这些 `PaymentContext`，不是 GC 失效，而是它们一直被 `static ConcurrentHashMap` 强引用着。

---

# 十三、最终根因

假设代码类似：

```java
public class PaymentContextCache {

    private static final ConcurrentHashMap<Long, PaymentContext> CACHE
        = new ConcurrentHashMap<>();

    public static void put(Long id, PaymentContext context) {
        CACHE.put(id, context);
    }
}
```

如果没有：

```text
TTL
最大容量
LRU / 淘汰
remove
定期清理
```

状态流转就是：

```text
请求不断创建 PaymentContext
    ↓
不断 put 到 static CACHE
    ↓
CACHE 生命周期 = JVM 生命周期
    ↓
对象一直有强引用
    ↓
GC 无法回收
    ↓
Old 区持续上涨
    ↓
Full GC 越来越频繁
    ↓
STW / CPU 抖动 / 接口 RT 毛刺
    ↓
最终可能 OOM
```

最终根因：

> `PaymentContextCache` 使用无界 `static ConcurrentHashMap` 长期保存业务对象，缺少淘汰或删除机制，导致大量 `PaymentContext` 被 GC Root 强引用，无法被 GC 回收，最终造成老年代持续增长和频繁 Full GC。

---

# 十四、完整排查链路

```text
接口周期性卡顿 / CPU 抖动 / 内存高
        ↓
top
        ↓
发现 Java CPU 和内存占用异常
        ↓
jstat -gcutil PID 1000
        ↓
Young GC 是否频繁？
Old 是否持续增长？
Full GC 是否增加？
        ↓
发现：
YGC 高频
Old 91% → 94%
FGC 7 → 8
Full GC 后 Old 仍 67%
        ↓
怀疑大量长期存活对象
        ↓
jcmd PID GC.class_histogram
        ↓
发现 PaymentContext / PaymentRecord 数量异常
        ↓
重复 histogram
        ↓
确认对象数量持续上涨
        ↓
Heap Dump
        ↓
MAT Dominator Tree
        ↓
PaymentContextCache Retained Heap = 2.31GB
        ↓
Path To GC Roots
        ↓
static CACHE → ConcurrentHashMap → PaymentContext
        ↓
根因：无界静态缓存导致对象无法回收
```

---

# 十五、三类线上排障场景对比

现在已经练过三类典型问题：

```text
场景 1：CPU 高
    ↓
RUNNABLE
    ↓
计算型热点
    ↓
正则灾难性回溯

场景 2：CPU 正常但接口慢
    ↓
WAITING / TIMED_WAITING
    ↓
资源等待
    ↓
连接池耗尽
    ↓
长事务持有数据库连接

场景 3：JVM 频繁 GC
    ↓
Old 持续上涨
    ↓
Full GC 后仍降不下来
    ↓
大量长期存活对象
    ↓
Heap Dump + MAT
    ↓
静态缓存强引用导致内存泄漏
```

可以先记一个最上层判断：

> **线上 Java 服务变慢，先判断线程是在“算”、在“等”，还是 JVM 在“忙着回收内存”。**

---

# 十六、面试一句话回答版

> 如果怀疑 JVM 频繁 GC，我会先用 `top` 看 Java 进程 CPU 和内存，再用 `jstat -gcutil PID 1000` 连续观察 Young GC、Full GC 和 Old 区变化。如果 Old 区持续上涨且 Full GC 后仍降不下来，我会怀疑大量长期存活对象。接着用 `jcmd GC.class_histogram` 找异常类，并通过多次 histogram 看对象是否持续增长；确认后再谨慎生成 Heap Dump，用 MAT 的 Dominator Tree 看 Retained Heap，再通过 Path To GC Roots 找是谁持有这些对象，最终定位到真正的强引用链，而不是简单归因于“GC 有问题”。
