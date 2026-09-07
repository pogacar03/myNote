# A 向 B/C/D 并发请求，任意一个返回后取消其余任务

## 一、场景

线程 A 同时向 B、C、D 发起请求：

```text
A
├── 请求 B
├── 请求 C
└── 请求 D
```

要求：

1. B、C、D 并发执行；
2. 任意一个任务**成功返回**后，A 立即拿到这个结果；
3. 其他两个任务的返回结果不再处理；
4. 尝试取消其他两个还没有完成的任务。

---

## 二、最适合的 API：`invokeAny()`

Java `ExecutorService` 提供：

```java
invokeAny()
```

它的语义就是：

> 同时执行多个 `Callable`，任意一个任务成功完成后立即返回它的结果，并取消其他尚未完成的任务。

### 示例

```java
ExecutorService pool = Executors.newFixedThreadPool(3);

List<Callable<String>> tasks = List.of(
        () -> requestB(),
        () -> requestC(),
        () -> requestD()
);

try {
    String result = pool.invokeAny(tasks);

    System.out.println("最终结果：" + result);

} catch (InterruptedException | ExecutionException e) {
    e.printStackTrace();
}
```

---

## 三、执行流程

假设：

```text
B：500ms 返回
C：100ms 返回
D：300ms 返回
```

执行过程：

```text
A
↓
同时提交 B / C / D

B ───────────────── 500ms
C ─── 100ms 返回
D ─────────── 300ms

        ↓

C 第一个成功返回

        ↓

invokeAny() 返回 C 的结果

        ↓

取消 B、D 尚未完成的任务
```

因此 A 最终只拿到：

```text
C 的结果
```

---

# 四、取消任务不是“强制杀死线程”

这是这个问题里最重要的点。

当其他任务被取消时，本质上通常相当于：

```java
future.cancel(true);
```

内部会尝试：

```java
thread.interrupt();
```

状态流转：

```text
cancel(true)
    ↓
interrupt()
    ↓
设置线程中断标志
```

但是：

> `interrupt()` 只是通知线程“你应该停止了”，并不会暴力杀死线程。

例如：

```java
while (true) {
    // 一直计算
}
```

如果代码完全不检查中断状态，那么即使调用：

```java
cancel(true);
```

线程仍然可能继续运行。

---

## 五、任务必须主动响应中断

CPU 型任务可以主动检查：

```java
while (!Thread.currentThread().isInterrupted()) {
    // 执行业务
}
```

一旦被取消：

```text
cancel(true)
    ↓
interrupt()
    ↓
isInterrupted() == true
    ↓
退出循环
    ↓
线程结束
```

---

## 六、阻塞任务如何响应 interrupt

Java 中很多阻塞方法天然支持中断，例如：

```java
Thread.sleep()
BlockingQueue.take()
CountDownLatch.await()
Future.get()
```

线程阻塞期间收到：

```java
interrupt()
```

通常会抛出：

```java
InterruptedException
```

因此可以这样处理：

```java
try {
    Thread.sleep(10000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    return;
}
```

---

# 七、如果 B/C/D 是 HTTP / RPC 请求

这里要特别注意：

```text
Future.cancel(true)
```

并不一定意味着：

```text
HTTP 请求真的被取消
```

因为可能出现：

```text
Java 工作线程
    ↓
发送 HTTP 请求
    ↓
线程等待远程服务响应
```

此时：

```text
cancel(true)
```

虽然给工作线程发出了 interrupt，但底层 HTTP Client 是否真正取消网络请求，要看客户端实现。

更完整的取消链路应该是：

```text
C 第一个成功
    ↓
取消 B、D Future
    ↓
interrupt B、D 工作线程
    ↓
取消底层 HTTP / RPC Call
    ↓
关闭或终止网络等待
```

例如 OkHttp 可以：

```java
call.cancel();
```

因此：

> Java 任务取消和底层网络请求取消是两个层面的事情。

---

# 八、“拒绝其他两个返回”和“停止线程”是两个问题

需求其实包含两个目标：

```text
① 只接受第一个结果
② 尽量停止其他任务
```

第二个目标可能失败，因此业务层最好再增加一道保护。

例如使用：

```java
AtomicBoolean completed = new AtomicBoolean(false);
```

每个任务返回时：

```java
if (completed.compareAndSet(false, true)) {
    // 第一个返回的任务
    // 结果生效
} else {
    // 已经有其他任务返回
    // 当前结果直接丢弃
}
```

状态流转：

```text
B / C / D 并发执行
        ↓
某个任务先返回
        ↓
CAS：false → true
        ↓
成为 winner
        ↓
A 接受结果
        ↓
取消其他任务
```

其他任务之后再返回：

```text
CAS 失败
    ↓
说明已经存在 winner
    ↓
当前结果直接丢弃
```

因此形成双保险：

```text
第一层：cancel()
尽量停止其他任务

第二层：AtomicBoolean / CAS
即使取消失败，也禁止其他结果生效
```

---

# 九、为什么不能只依赖 cancel

例如：

```text
C 第一个返回
    ↓
cancel B
cancel D
```

但是 B 此时可能已经：

```text
请求发送到远程服务器
        ↓
远程服务器已经执行完成
        ↓
响应正在网络中返回
```

这时 B 可能依然返回结果。

所以真正可靠的是：

```text
winner 机制
+
cancel 机制
```

而不是单纯：

```text
cancel
```

---

# 十、线程池本身如何优雅关闭

如果这组任务使用的是一个独立线程池，任务结束后还需要关闭线程池。

常用 API：

| 方法 | 作用 |
|---|---|
| `shutdown()` | 不再接收新任务，已经提交的任务继续执行 |
| `shutdownNow()` | 尝试中断正在执行的线程，并返回尚未开始执行的任务 |
| `awaitTermination()` | 等待线程池在指定时间内结束 |
| `isShutdown()` | 判断线程池是否已进入关闭状态 |
| `isTerminated()` | 判断线程池是否已经彻底终止 |

典型写法：

```java
pool.shutdown();

try {
    if (!pool.awaitTermination(30, TimeUnit.SECONDS)) {
        pool.shutdownNow();
    }
} catch (InterruptedException e) {
    pool.shutdownNow();
    Thread.currentThread().interrupt();
}
```

状态流转：

```text
RUNNING
    ↓
shutdown()
    ↓
不再接收新任务
    ↓
已提交任务继续执行
    ↓
全部完成
    ↓
TERMINATED
```

如果等待超时：

```text
shutdown()
    ↓
awaitTermination()
    ↓
超时
    ↓
shutdownNow()
    ↓
interrupt 剩余任务
```

---

# 十一、面试回答

如果面试官问：

> A 同时请求 B、C、D，只需要最快成功的一个结果，另外两个怎么关闭？

可以回答：

> 我会使用 `ExecutorService.invokeAny()` 并发提交 B、C、D 三个 `Callable`。`invokeAny()` 会在任意一个任务成功完成后立即返回该结果，并取消其他尚未完成的任务。
>
> 但是取消任务本质上通常依赖 `Future.cancel(true)` 和线程 `interrupt`，并不是强制杀死线程，因此任务本身必须正确响应中断。如果 B、C、D 执行的是 HTTP 或 RPC 请求，还需要结合底层客户端的 cancel 能力真正取消网络调用。
>
> 另外我会在业务层通过 `AtomicBoolean` 或 CAS 保证只有第一个成功结果能够生效。这样即使其他任务取消失败、后续仍然返回，它们的结果也会被直接丢弃。

---

# 十二、核心记忆

```text
A 同时请求 B / C / D
        ↓
ExecutorService.invokeAny()
        ↓
任意一个成功返回
        ↓
A 获取 winner 结果
        ↓
cancel 其他 Future
        ↓
interrupt 其他线程
```

但要牢记：

```text
cancel(true)
≠
强制杀死线程
```

完整方案：

```text
invokeAny
    +
Future.cancel(true)
    +
任务响应 interrupt
    +
HTTP / RPC 底层 cancel
    +
CAS 保证只有一个结果生效
```

一句话记忆：

> **结果层靠 CAS 保证“只有一个赢家”，执行层靠 cancel + interrupt 尽量停止其他任务。**
