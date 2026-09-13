# 回文分割怎么用 start 和 end 做切割型回溯？

对应 LeetCode 131：Palindrome Partitioning。

## 题型识别

这题不是单纯判断回文，而是：

> 把字符串切成若干段，并且要求每一段都是回文串。

所以本质是：

```text
回溯 + 枚举切割点 + 双指针判断回文
```

这题很适合综合复习：

- 回溯
- start / end 区间定义
- substring 左闭右开
- 双指针判断回文
- 递归终止条件
- path 的加入与撤销

---

## 核心拆分

把职责拆成两部分：

```text
check(left, right)
负责：判断 s[left...right] 是否是回文

backtrack(start)
负责：从 start 开始，枚举这一段切到哪里
```

最关键的三个变量：

```text
start：当前这一段从哪里开始
end：当前这一段切到哪里
path：前面已经切好的那些段
```

当前候选区间统一定义成：

```text
[start, end]
```

因此后面所有代码都围绕这个定义写。

---

## 最新写法

```java
class Solution {
    List<List<String>> res = new ArrayList<>();
    List<String> path = new ArrayList<>();

    public List<List<String>> partition(String s) {
        backtrack(s, 0);
        return res;
    }

    public boolean check(String s, int left, int right) {
        while (left < right) {
            if (s.charAt(left) != s.charAt(right)) {
                return false;
            }
            left++;
            right--;
        }
        return true;
    }

    public void backtrack(String s, int start) {
        if (start == s.length()) {
            res.add(new ArrayList<>(path));
            return;
        }

        for (int end = start; end < s.length(); end++) {
            String cur = s.substring(start, end + 1);

            if (check(s, start, end)) {
                path.add(cur);
                backtrack(s, end + 1);
                path.remove(path.size() - 1);
            }
        }
    }
}
```

---

## 为什么 for 从 end = start 开始

当前这一段是：

```text
[start, end]
```

所以 end 必须从 start 开始尝试。

例如：

```text
s = "aab"
start = 0
```

第一层可以尝试：

```text
end = 0 -> "a"
end = 1 -> "aa"
end = 2 -> "aab"
```

也就是枚举“这一刀切到哪里”。

正确：

```java
for (int end = start; end < s.length(); end++)
```

不能写：

```java
end < s.length() - 1
```

否则最后一个字符永远不会被包含进当前区间。

---

## substring 为什么要 end + 1

Java 的：

```java
s.substring(a, b)
```

表示：

```text
[a, b)
```

左边包含，右边不包含。

但我们当前区间定义的是：

```text
[start, end]
```

所以真正截取时必须写：

```java
s.substring(start, end + 1)
```

统一记忆：

```text
逻辑区间：[start, end]
substring：substring(start, end + 1)
```

---

## 为什么 check 直接检查原字符串

当前区间已经明确是：

```text
[start, end]
```

因此直接：

```java
check(s, start, end)
```

不需要先 substring 出 cur 再拿原字符串的 start/end 去检查 cur。

因为 cur 的下标会从 0 重新开始，和原字符串的 start/end 不是同一套下标体系。

---

## 为什么递归是 end + 1

当前已经选中了：

```text
[start, end]
```

那么下一段就应该从：

```text
end + 1
```

开始。

所以：

```java
backtrack(s, end + 1);
```

这一句不是“访问 s[end + 1]”，而是在表达：

> 前面已经切到 end 了，下一段从 end + 1 开始。

---

## start == s.length() 为什么不越界

如果最后一段刚好切到：

```text
end = n - 1
```

那么递归会进入：

```java
backtrack(s, n);
```

这里的 n 不能作为字符串下标，但可以作为一种“已经处理完”的状态值。

所以一进递归先判断：

```java
if (start == s.length()) {
    res.add(new ArrayList<>(path));
    return;
}
```

于是不会继续访问：

```java
s.charAt(n)
```

统一记忆：

```text
0 ~ n-1：合法字符下标
n：可以作为“刚好处理完”的递归状态
```

---

## check 为什么 while(left < right)

判断回文：

```java
while (left < right) {
    if (s.charAt(left) != s.charAt(right)) {
        return false;
    }
    left++;
    right--;
}
```

当：

```text
left == right
```

说明已经来到中间单个字符，不需要继续比较。

所以：

```java
left < right
```

比 `left <= right` 更自然。

---

## 用 aab 跑一遍

```text
s = "aab"
```

第一条路径：

```text
start = 0
end = 0
当前段 "a" 是回文
path = ["a"]

    start = 1
    end = 1
    当前段 "a" 是回文
    path = ["a", "a"]

        start = 2
        end = 2
        当前段 "b" 是回文
        path = ["a", "a", "b"]

            start = 3 == n
            收集答案
```

另一条路径：

```text
start = 0
end = 1
当前段 "aa" 是回文
path = ["aa"]

    start = 2
    end = 2
    当前段 "b" 是回文
    path = ["aa", "b"]
```

最终：

```text
[
  ["a", "a", "b"],
  ["aa", "b"]
]
```

---

## 切割型回溯模板

只要题目出现：

> 把一个序列切成若干段，并要求每一段满足某个条件

就可以优先想：

```text
backtrack(start)

for end = start ... n-1
    当前候选 = [start, end]

    如果当前候选合法
        加入 path
        backtrack(end + 1)
        撤销 path
```

对应 LC131：

```text
合法条件 = [start, end] 是回文串
```

---

## 一句话记忆

```text
当前段永远定义成 [start, end]

for：end = start ... n-1
判断：check(start, end)
截取：substring(start, end + 1)
递归：backtrack(end + 1)
终止：start == n
```

这五句就是 LC131 的核心骨架。
