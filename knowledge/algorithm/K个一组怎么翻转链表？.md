# K 个一组怎么翻转链表？

对应 LeetCode 25：K 个一组翻转链表（Reverse Nodes in k-Group）。

## 面试回答

每一组先从 `groupPrev` 往后数 `k` 个节点，确认数量足够；不足 `k` 个直接结束，不翻转。

每组需要记住四个角色：

```text
groupPrev  当前组前一个节点
groupStart 当前组第一个节点
kth        当前组第 k 个节点
groupNext  下一组第一个节点
```

翻转完成后：

```text
原来的 kth       -> 新组头
原来的 groupStart -> 新组尾
```

然后重新接回前后链表。

## k = 3 的结构

翻转前：

```text
prev -> 1 -> 2 -> 3 -> 4
        ^         ^    ^
      oldHead    kth  next
```

翻转后：

```text
prev -> 3 -> 2 -> 1 -> 4
                    ^
                新 groupPrev
```

## 核心步骤

```text
1. 从 groupPrev 往后移动 k 次找 kth
2. kth == null -> 剩余不足 k 个，结束
3. groupNext = kth.next
4. 保存 groupStart = groupPrev.next
5. 反转 [groupStart, kth]
6. groupPrev.next = kth
7. groupPrev = groupStart
```

## 组内反转的关键技巧

把普通反转链表的 `prev = null` 改成：

```java
ListNode prev = groupNext;
ListNode cur = groupStart;
```

这样原组头翻转成组尾后，会自然接到 `groupNext`。

组内循环可以写成：

```java
while (cur != groupNext) {
    ListNode next = cur.next;
    cur.next = prev;
    prev = cur;
    cur = next;
}
```

然后：

```java
groupPrev.next = kth;
groupPrev = groupStart;
```

## 易错点

- 必须先判断剩余节点是否够 `k` 个。
- 下一轮 `groupPrev` 不是 `groupNext`，而是“原来的组头”，因为它翻转后成了组尾。
- 这题需要重点复习，重点不是普通反转，而是每组翻完之后前后怎么重新接。

## 一句话记忆

**先数够 k 个；kth 变新头，原组头变新尾；原组头继续做下一轮 groupPrev。**