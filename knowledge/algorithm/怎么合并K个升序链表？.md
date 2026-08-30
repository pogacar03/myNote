# 怎么合并 K 个升序链表？

## 面试回答

这题用 **小顶堆（PriorityQueue）**。

核心不是把所有节点一次性排序，而是：

- 堆里始终维护“每条链表当前最前面的候选节点”；
- 每次从堆顶拿出最小节点，接到结果链表后面；
- 如果这个节点还有 `next`，就把 `next` 放回堆；
- 一直处理到堆为空。

因为任意时刻，所有还没合并的节点里，真正有资格成为“下一个最小值”的，只可能是每条链表当前的头节点。

时间复杂度：`O(N log k)`，其中 `N` 是总节点数，`k` 是链表条数。

---

## 题型识别

看到：

```text
K 个已经有序的链表 / 数组
→ 每次从 K 个候选里选最小值
```

优先想到：

```text
小顶堆
```

这和“第 K 大”不一样：

- 第 K 大：堆是为了淘汰候选，通常维持固定大小 K；
- 合并 K 个链表：堆是为了在 K 条链表当前头节点中不断找最小值。

---

## 为什么堆里放 ListNode，而不是只放 val？

因为拿到最小值以后，还需要继续访问：

```java
node.next
```

所以堆里直接放节点：

```java
PriorityQueue<ListNode> pq =
        new PriorityQueue<>((a, b) -> a.val - b.val);
```

这样 `poll()` 出来以后，既知道当前值，也知道下一节点是谁。

更稳的比较器写法：

```java
PriorityQueue<ListNode> pq =
        new PriorityQueue<>((a, b) -> Integer.compare(a.val, b.val));
```

---

## 关键过程

例如：

```text
1 -> 4 -> 5
1 -> 3 -> 4
2 -> 6
```

初始化时，只把三个头节点放进堆：

```text
1, 1, 2
```

每次：

```text
poll 最小节点
→ 接到结果链表
→ 如果 node.next != null
   把 node.next 再放入堆
```

比如第一次取出 `1` 后，它所在链表的下一个节点是 `4`，于是堆变为：

```text
1, 2, 4
```

继续即可。

---

## 关键代码骨架

```java
public ListNode mergeKLists(ListNode[] lists) {
    PriorityQueue<ListNode> pq =
            new PriorityQueue<>((a, b) -> Integer.compare(a.val, b.val));

    // 先把每条链表的头节点放进堆
    for (ListNode node : lists) {
        if (node != null) {
            pq.offer(node);
        }
    }

    ListNode dummy = new ListNode(0);
    ListNode cur = dummy;

    while (!pq.isEmpty()) {
        ListNode node = pq.poll();

        cur.next = node;
        cur = cur.next;

        // 当前节点出堆后，把它所在链表的下一个候选补进堆
        if (node.next != null) {
            pq.offer(node.next);
        }
    }

    return dummy.next;
}
```

---

## 最容易忘的点

### 1. 不是把所有节点一次性塞进堆

只需要先放每条链表的头节点。

每次弹出一个节点后，再把它的 `next` 补进去。

### 2. 堆里存节点，不只存值

因为需要继续访问：

```java
node.next
```

### 3. 为什么用小顶堆？

因为每一步都需要快速拿到当前所有候选节点里的最小值：

```java
pq.poll();
```

### 4. 关键模板

```java
while (!pq.isEmpty()) {
    ListNode node = pq.poll();
    cur.next = node;
    cur = cur.next;

    if (node.next != null) {
        pq.offer(node.next);
    }
}
```

这几行是这题最值得形成肌肉记忆的代码。
