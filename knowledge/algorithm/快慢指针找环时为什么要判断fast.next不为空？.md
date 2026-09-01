# 快慢指针找环时为什么要判断 fast.next 不为空？

对应 LeetCode 141：环形链表（Linked List Cycle）。

## 面试回答

因为快指针每次走两步：

```java
fast = fast.next.next;
```

所以循环条件不能只判断 `fast != null`，还必须保证 `fast.next != null`，否则访问第二个 `next` 时可能空指针。

## 关键代码

```java
ListNode slow = head;
ListNode fast = head;

while (fast != null && fast.next != null) {
    slow = slow.next;
    fast = fast.next.next;

    if (slow == fast) {
        return true;
    }
}

return false;
```

## 一句话记忆

**fast 一次走两步，就要提前保证 `fast` 和 `fast.next` 都不是 null。**