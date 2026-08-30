# PriorityQueue 会自动维护顺序吗？

## 面试回答

会自动维护，但维护的是“堆序”，不是把所有元素完整排序。

Java 默认：

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();
```

是小顶堆，保证：

```java
pq.peek()
```

始终拿到当前最小值。

但直接打印 `pq`，内部顺序不一定是完整升序；只有不断 `poll()` 时，才会按从小到大依次取出。

## 默认比较规则

对于 `Integer`，默认自然顺序效果相当于：

```java
(a, b) -> Integer.compare(a, b)
```

也可以近似理解为：

```java
(a, b) -> a - b
```

即“小的优先”，所以堆顶是最小值，也就是小顶堆。

## 大顶堆

推荐写法：

```java
PriorityQueue<Integer> maxHeap =
        new PriorityQueue<>(Comparator.reverseOrder());
```

也常见：

```java
PriorityQueue<Integer> maxHeap =
        new PriorityQueue<>((a, b) -> b - a);
```

大顶堆保证 `peek()` 是当前最大值。

## Top K 里的常用结论

```text
求第 K 大
-> 保留最大的 K 个
-> 用小顶堆
-> 超过 K 个就 poll 最小值

求第 K 小
-> 保留最小的 K 个
-> 用大顶堆
```

第 K 大的简洁模板：

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();

for (int num : nums) {
    pq.offer(num);

    if (pq.size() > k) {
        pq.poll();
    }
}

return pq.peek();
```

## 高频元素的自定义比较

如果 `map` 存的是：

```text
数字 -> 频率
```

按频率构造小顶堆：

```java
PriorityQueue<Integer> pq =
        new PriorityQueue<>((a, b) -> Integer.compare(map.get(a), map.get(b)));
```

此时堆顶是“频率最低”的候选，超过 `k` 个就淘汰它，最后留下频率最高的 `k` 个。

## 一句话记忆

**PriorityQueue 自动维护堆顶，不自动把整个容器完整排序；默认小顶堆，`peek()` 最小。**