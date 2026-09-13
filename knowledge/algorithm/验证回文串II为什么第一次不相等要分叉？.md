# 验证回文串 II 为什么第一次不相等要分叉？

对应 LeetCode 680：Valid Palindrome II。

## 题型识别

这题本质是：

```text
普通回文双指针
+
最多允许删除一个字符
```

正常情况下：

```java
if (s.charAt(left) == s.charAt(right)) {
    left++;
    right--;
}
```

一旦第一次出现不相等：

```text
s[left] != s[right]
```

因为最多只能删一个字符，所以只剩两种可能：

```text
1. 删除左边当前字符  -> 检查 [left + 1, right]
2. 删除右边当前字符  -> 检查 [left, right - 1]
```

只要其中一个区间本身是回文串，答案就是 true。

## 推荐写法

```java
class Solution {
    public boolean validPalindrome(String s) {

        int left = 0;
        int right = s.length() - 1;

        while (left < right) {

            if (s.charAt(left) == s.charAt(right)) {
                left++;
                right--;
            } else {
                return check(s, left + 1, right)
                        || check(s, left, right - 1);
            }
        }

        return true;
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
}
```

## 为什么不建议硬塞一个 count 到主 while 里

当然可以尝试记录“已经删过几次”，但一旦第一次 mismatch 出现，真正的问题不是继续线性移动，而是：

```text
到底删 left，还是删 right？
```

这其实已经天然分成两个独立分支。

所以最清晰的拆分粒度是：

```text
主函数：
正常双指针，找到第一次冲突

check：
判断删掉一个字符后的剩余区间是否为普通回文串
```

## 例子

```text
s = "abca"
```

开始：

```text
a b c a
↑     ↑
left right
```

`a == a`，继续往里：

```text
a b c a
  ↑ ↑
```

此时：

```text
b != c
```

只能二选一：

```text
删 b -> "aca" -> 回文
删 c -> "aba" -> 回文
```

因此：

```java
check(s, left + 1, right)
||
check(s, left, right - 1)
```

## 易错点

不是：

```text
left - 1
right + 1
```

而是：

```text
left + 1
right - 1
```

因为是在跳过当前冲突字符，继续向中间缩。

## 一句话记忆

```text
LC680 = 普通回文双指针
       + 第一次不相等时，删左或删右二选一。
```
