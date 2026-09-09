# Java 慢接口实战：CPU 正常但连接池耗尽 / 长事务

> 目标：保留完整模拟 Terminal 输入输出，并且在每一步解释“为什么下一步这么查”。
>
> 这个场景和上一篇“CPU 高 + RUNNABLE + 正则回溯”形成对照：这里 CPU 不高，但接口 P99 很慢，核心是资源等待。

---

# 一、事故场景

线上告警：

```text
服务：inventory-service
环境：production / Kubernetes
实例数：8

接口：
GET /api/inventory/detail?id=xxx

正常：
AVG RT ≈ 80ms
P99 ≈ 220ms

现在：
AVG RT ≈ 1.2s
P99 ≈ 4.8s

QPS：
平时 ≈ 1200
现在 ≈ 1180

错误率：
0.1%

CPU：
监控看起来没有明显升高
```

进入异常 Pod：

```text
Connected to pod: inventory-service-6f7d9f6f87-m4n2k

Linux inventory-service-6f7d9f6f87-m4n2k 5.15.0
Java: OpenJDK 17
PID: unknown

[prod@inventory-service-6f7d9f6f87-m4n2k ~]$ █
```

---

# 二、第一步：`top` 判断是不是 CPU 型问题

输入：

```bash
top
```

模拟输出：

```text
top - 02:00:11 up 23 days,  9:14,  0 users,  load average: 1.18, 1.09, 1.03
Tasks:  51 total,   1 running,  50 sleeping,   0 stopped,   0 zombie

%Cpu(s):  8.7 us,  2.1 sy,  0.0 ni, 88.5 id,  0.3 wa,  0.0 hi,  0.4 si,  0.0 st

MiB Mem :   8192.0 total,    684.2 free,   3910.5 used,   3597.3 buff/cache

    PID USER      PR  NI      VIRT      RES    SHR S  %CPU  %MEM     TIME+ COMMAND
    341 app       20   0     9.6g     2.8g  27840 S   18.3  35.1   623:17 java
      1 app       20   0     6.8m     3.8m   3100 S    0.0   0.0     0:01 tini
```

## 这一步怎么看？

```text
CPU idle = 88.5%
Java CPU = 18.3%
```

说明：

> 这次不是典型 CPU 打满问题。

因此不要直接往“死循环 / 正则回溯 / 热点计算”方向走。

CPU 不高但接口慢，更可能是：

```text
数据库等待
Redis 等待
RPC / HTTP 等待
锁等待
线程池等待
连接池等待
磁盘 / 网络 I/O
```

注意：

> CPU 不高不等于“就是慢 SQL”。慢 SQL只是一个候选。

---

# 三、已知具体慢接口时，优先 `trace`

进入 Arthas：

```text
[arthas@341]$ █
```

这次不优先使用：

```bash
thread -n 5
```

因为 `thread -n 5` 更适合找高 CPU Java 线程。

我们已经知道慢接口是：

```text
GET /api/inventory/detail
```

所以目标是直接回答：

> 这条调用链的 4 秒到底花在哪？

输入：

```bash
trace com.sf.inventory.controller.InventoryController detail '#cost > 100'
```

模拟输出：

```text
Press Q or Ctrl+C to abort.
Affect(class count: 1 , method count: 1) cost in 96 ms, listenerId: 2

`---ts=2026-09-09 09:07:21.418;thread_name=http-nio-8080-exec-31;id=126;
    `---[4268.731ms] com.sf.inventory.controller.InventoryController:detail()
        +---[0.031ms] java.lang.Long:valueOf() #71
        +---[4268.411ms] com.sf.inventory.service.InventoryService:getDetail() #74
        `---[0.102ms] com.sf.common.api.Result:success() #76

`---ts=2026-09-09 09:07:23.906;thread_name=http-nio-8080-exec-42;id=137;
    `---[3917.284ms] com.sf.inventory.controller.InventoryController:detail()
        +---[0.027ms] java.lang.Long:valueOf() #71
        +---[3916.998ms] com.sf.inventory.service.InventoryService:getDetail() #74
        `---[0.094ms] com.sf.common.api.Result:success() #76
```

## 判断

可以看到：

```text
Controller 总耗时 ≈ 4s
    ↓
几乎全部耗在 InventoryService.getDetail() #74
```

但：

> `#74` 只是 Controller 调用 Service 的那一行，不代表最终根因就在这一行。

还要继续往 Service 里面拆。

---

# 四、继续 trace Service

输入：

```bash
trace com.sf.inventory.service.InventoryService getDetail '#cost > 100'
```

模拟输出：

```text
Press Q or Ctrl+C to abort.
Affect(class count: 1 , method count: 1) cost in 84 ms, listenerId: 3

`---ts=2026-09-09 09:07:51.624;thread_name=http-nio-8080-exec-31;id=126;
    `---[4281.447ms] com.sf.inventory.service.InventoryService:getDetail()
        +---[0.112ms] com.sf.inventory.cache.InventoryCache:get() #118
        +---[4267.931ms] com.sf.inventory.mapper.InventoryMapper:selectById() #126
        +---[0.084ms] com.sf.inventory.converter.InventoryConverter:convert() #131
        `---[12.887ms] com.sf.inventory.cache.InventoryCache:put() #135

`---ts=2026-09-09 09:07:54.190;thread_name=http-nio-8080-exec-42;id=137;
    `---[3954.816ms] com.sf.inventory.service.InventoryService:getDetail()
        +---[0.096ms] com.sf.inventory.cache.InventoryCache:get() #118
        +---[3940.272ms] com.sf.inventory.mapper.InventoryMapper:selectById() #126
        +---[0.091ms] com.sf.inventory.converter.InventoryConverter:convert() #131
        `---[13.942ms] com.sf.inventory.cache.InventoryCache:put() #135
```

## 判断

当前最可疑：

```text
InventoryMapper.selectById() #126
    ↓
耗时 3.9 ~ 4.2s
```

但仍然不能直接说：

> “SQL 执行了 4 秒。”

因为一次 Mapper 调用可能包含：

```text
先向连接池申请数据库连接
    ↓
拿到连接
    ↓
真正执行 SQL
```

所以要继续区分：

> 是“拿连接慢”，还是“SQL 执行慢”？

---

# 五、trace HikariCP：确认是不是拿连接慢

输入：

```bash
trace com.zaxxer.hikari.pool.HikariPool getConnection '#cost > 100'
```

模拟输出：

```text
Press Q or Ctrl+C to abort.
Affect(class count: 1 , method count: 1) cost in 73 ms, listenerId: 4

`---ts=2026-09-09 09:08:41.227;thread_name=http-nio-8080-exec-31;id=126;
    `---[4018.624ms] com.zaxxer.hikari.pool.HikariPool:getConnection()
        `---[4018.391ms] com.zaxxer.hikari.pool.HikariPool:getConnection() #162

`---ts=2026-09-09 09:08:43.914;thread_name=http-nio-8080-exec-42;id=137;
    `---[3726.118ms] com.zaxxer.hikari.pool.HikariPool:getConnection()
        `---[3725.887ms] com.zaxxer.hikari.pool.HikariPool:getConnection() #162
```

## 关键判断

现在两层时间几乎吻合：

```text
selectById() ≈ 4s
getConnection() ≈ 4s
```

所以：

> 这 4 秒主要花在“从 Hikari 连接池拿连接”，不是 SQL 真正执行了 4 秒。

排查方向正式变成：

```text
为什么连接池暂时拿不到连接？
```

常见候选：

```text
1. 连接池满了
2. 慢 SQL 长时间占着连接
3. 事务范围过大，连接长时间不释放
4. 连接泄漏
```

---

# 六、看 WAITING 线程：确认大量请求都在等连接

输入：

```bash
thread --state WAITING
```

模拟输出：

```text
Threads Total: 196, NEW: 0, RUNNABLE: 34, BLOCKED: 0, WAITING: 71, TIMED_WAITING: 91, TERMINATED: 0

"http-nio-8080-exec-31" Id=126 WAITING
    at jdk.internal.misc.Unsafe.park(Native Method)
    at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:252)
    at com.zaxxer.hikari.util.ConcurrentBag.borrow(ConcurrentBag.java:119)
    at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:164)
    at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:146)
    at com.zaxxer.hikari.HikariDataSource.getConnection(HikariDataSource.java:100)
    at org.springframework.jdbc.datasource.DataSourceUtils.fetchConnection(DataSourceUtils.java:160)
    at org.mybatis.spring.transaction.SpringManagedTransaction.openConnection(SpringManagedTransaction.java:80)
    ...

"http-nio-8080-exec-42" Id=137 WAITING
    at jdk.internal.misc.Unsafe.park(Native Method)
    at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:252)
    at com.zaxxer.hikari.util.ConcurrentBag.borrow(ConcurrentBag.java:119)
    at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:164)
    at com.zaxxer.hikari.HikariDataSource.getConnection(HikariDataSource.java:100)
    ...

"http-nio-8080-exec-44" Id=139 WAITING
    at jdk.internal.misc.Unsafe.park(Native Method)
    at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:252)
    at com.zaxxer.hikari.util.ConcurrentBag.borrow(ConcurrentBag.java:119)
    at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:164)
    ...

"http-nio-8080-exec-47" Id=142 WAITING
    at jdk.internal.misc.Unsafe.park(Native Method)
    at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:252)
    at com.zaxxer.hikari.util.ConcurrentBag.borrow(ConcurrentBag.java:119)
    at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:164)
    ...

... 23 similar threads omitted ...
```

## 判断

大量 HTTP 请求线程都在：

```text
ConcurrentBag.borrow()
    ↓
HikariPool.getConnection()
```

说明：

> 不是某一个请求偶尔拿连接慢，而是连接池里当前没有足够可用连接，大量请求都在排队等。

现在要继续问：

> 连接到底被谁占着？为什么迟迟不归还？

---

# 七、看 TIMED_WAITING：找“拿着资源但正在等待”的线程

输入：

```bash
thread --state TIMED_WAITING
```

模拟输出：

```text
Threads Total: 196, NEW: 0, RUNNABLE: 34, BLOCKED: 0, WAITING: 71, TIMED_WAITING: 91, TERMINATED: 0

"http-nio-8080-exec-8" Id=103 TIMED_WAITING
    at jdk.internal.misc.Unsafe.park(Native Method)
    at java.util.concurrent.locks.LockSupport.parkNanos(LockSupport.java:252)
    at java.util.concurrent.CompletableFuture$Signaller.block(CompletableFuture.java:1866)
    at java.util.concurrent.ForkJoinPool.unmanagedBlock(ForkJoinPool.java:3465)
    at java.util.concurrent.CompletableFuture.timedGet(CompletableFuture.java:1939)
    at java.util.concurrent.CompletableFuture.get(CompletableFuture.java:2095)
    at com.sf.inventory.client.WarehouseClient.queryStock(WarehouseClient.java:148)
    at com.sf.inventory.service.InventoryService.getDetail(InventoryService.java:121)
    at com.sf.inventory.service.InventoryService$$EnhancerBySpringCGLIB.getDetail(<generated>)
    at org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:119)
    ...

"http-nio-8080-exec-11" Id=106 TIMED_WAITING
    at java.util.concurrent.CompletableFuture.timedGet(CompletableFuture.java:1939)
    at java.util.concurrent.CompletableFuture.get(CompletableFuture.java:2095)
    at com.sf.inventory.client.WarehouseClient.queryStock(WarehouseClient.java:148)
    at com.sf.inventory.service.InventoryService.getDetail(InventoryService.java:121)
    at org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:119)
    ...

"http-nio-8080-exec-15" Id=110 TIMED_WAITING
    at java.util.concurrent.CompletableFuture.timedGet(CompletableFuture.java:1939)
    at java.util.concurrent.CompletableFuture.get(CompletableFuture.java:2095)
    at com.sf.inventory.client.WarehouseClient.queryStock(WarehouseClient.java:148)
    at com.sf.inventory.service.InventoryService.getDetail(InventoryService.java:121)
    at org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:119)
    ...

... 19 similar http-nio threads omitted ...
```

## 这里出现两个强线索

第一条：

```text
WarehouseClient.queryStock()
    ↓
CompletableFuture.get(timeout)
    ↓
TIMED_WAITING
```

说明很多请求在等外部 Warehouse 调用返回。

第二条：

```text
TransactionInterceptor
```

说明这段调用处在 Spring 事务拦截器里面。

因此开始怀疑：

> 外部 RPC 被放在事务范围内，线程在等待 RPC 时，事务相关资源可能一直被占着，造成连接池资源紧张。

这时候一个合理判断就是：

```text
长事务 / 事务范围过大
```

---

# 八、`jad` 看真实事务范围

输入：

```bash
jad com.sf.inventory.service.InventoryService
```

模拟输出：

```java
package com.sf.inventory.service;

import org.springframework.transaction.annotation.Transactional;

public class InventoryService {

    @Transactional
    public InventoryDetail getDetail(Long id) {

        // #118 查询本地缓存
        InventoryDetail detail = inventoryCache.get(id);

        if (detail == null) {

            // #121 查询仓库服务库存
            StockInfo stock =
                warehouseClient.queryStock(id);

            // #126 查询数据库
            detail =
                inventoryMapper.selectById(id);

            detail.setStock(stock);

            inventoryCache.put(id, detail);
        }

        return detail;
    }
}
```

## 根因链路

现在可以把证据串起来：

```text
@Transactional 包住整个 getDetail()
        ↓
请求进入事务
        ↓
WarehouseClient.queryStock()
        ↓
CompletableFuture.get() 等外部 RPC
        ↓
事务持续时间被 RPC 延长
        ↓
数据库连接/事务资源长期被占用
        ↓
连接池可用连接减少
        ↓
后续请求 HikariPool.getConnection() 等待
        ↓
大量线程 WAITING
        ↓
接口 P99 飙升，但 CPU 仍然不高
```

核心问题：

> **事务范围过大，把不可控耗时的外部调用包进了事务。**

---

# 九、错误代码与修复思路

错误：

```java
@Transactional
public Detail getDetail(Long id) {

    StockInfo stock =
        warehouseClient.queryStock(id);

    return mapper.selectById(id);
}
```

问题在于：

```text
RPC 调用
    ↓
被事务包住
    ↓
RPC 如果慢 3 秒
    ↓
事务就跟着拖 3 秒
```

更合理的思路：

```java
public Detail getDetail(Long id) {

    StockInfo stock =
        warehouseClient.queryStock(id);

    return transactionTemplate.execute(status -> {
        Detail detail = mapper.selectById(id);
        detail.setStock(stock);
        return detail;
    });
}
```

状态流转变成：

```text
先做 RPC
    ↓
RPC 结束
    ↓
再进入短事务
    ↓
数据库操作
    ↓
快速提交
```

核心原则：

> **不要把 RPC、HTTP、MQ 等不可控耗时的外部调用放进不必要的大事务里。**

---

# 十、这道题和 CPU 高场景怎么区分？

## 场景 A：CPU 高

```text
接口慢
 ↓
top：Java CPU 高
 ↓
thread -n 5
 ↓
线程 RUNNABLE
 ↓
热点计算
 ↓
trace / profiler
```

典型根因：

```text
死循环
正则灾难性回溯
大对象计算
序列化热点
算法复杂度问题
```

## 场景 B：CPU 正常

```text
接口慢
 ↓
top：CPU 正常
 ↓
已知慢接口 → trace 调用链
 ↓
发现资源等待
 ↓
thread --state WAITING / TIMED_WAITING
 ↓
定位连接池 / RPC / 锁等等待
```

本题最终：

```text
CPU 正常
 ↓
接口 P99 4.8s
 ↓
Mapper 看起来耗时 4s
 ↓
Hikari getConnection 也耗时 4s
 ↓
大量线程等连接
 ↓
另一些线程在事务内等 Warehouse RPC
 ↓
事务范围过大
 ↓
连接池资源耗尽
```

---

# 十一、几个容易答错的问题

## Q1：CPU 不高，是不是直接判断慢 SQL？

不是。

CPU 不高只能说明：

> 更像资源等待型问题，而不是纯计算型问题。

候选还有 Redis、RPC、连接池、线程池、锁、网络等。

---

## Q2：Mapper 方法耗时 4 秒，是不是 SQL 就执行了 4 秒？

不是。

Mapper 调用里可能包含：

```text
等待数据库连接
+
真正 SQL 执行
```

本题通过：

```bash
trace com.zaxxer.hikari.pool.HikariPool getConnection
```

发现 4 秒几乎全花在拿连接。

---

## Q3：`thread -n 5` 为什么这题不是第一选择？

因为它更适合：

> 找 CPU 占用最高的 Java 线程。

本题 CPU 根本不高，而且已经知道具体慢接口，所以优先 `trace Controller` 更直接。

---

## Q4：WAITING 和 TIMED_WAITING 在这题里分别说明什么？

本题中：

```text
WAITING
→ 大量请求在 Hikari ConcurrentBag.borrow() 等连接
```

而：

```text
TIMED_WAITING
→ 已经进入业务流程的请求在 CompletableFuture.get(timeout) 等 Warehouse RPC
```

两个状态放在一起，帮助把“谁在等连接”和“谁在拖着事务等外部调用”串起来。

---

# 十二、最终背诵链路

```text
慢接口
 ↓
top
 ↓
CPU 正常
 ↓
已知具体接口
 ↓
Arthas trace Controller
 ↓
定位 Service
 ↓
trace Service
 ↓
发现 Mapper 调用耗时 4s
 ↓
不要直接判断慢 SQL
 ↓
trace HikariPool.getConnection
 ↓
发现拿连接耗时 4s
 ↓
thread --state WAITING
 ↓
大量请求在等数据库连接
 ↓
thread --state TIMED_WAITING
 ↓
发现很多线程在事务内等待 Warehouse RPC
 ↓
jad 查看代码
 ↓
@Transactional 包住 RPC + DB
 ↓
根因：事务范围过大，外部调用拖长事务，连接池资源被长期占用
```

一句话总结：

> **慢接口先判断是“CPU 在算”还是“线程在等”。CPU 不高时不要先入为主认定慢 SQL；沿调用链 trace，继续区分 SQL 执行时间和连接池等待时间，再结合线程状态和真实事务范围，才能把“接口慢 → 等连接 → 事务内等 RPC → 长事务”这条证据链闭环。**