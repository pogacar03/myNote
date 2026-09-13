# LC16 最接近的三数之和：为什么用 Math.abs？

## 核心

题目要找的是：三个数的和 `sum` 距离 `target` 最近。

“距离”只看差多少，不关心是在 target 左边还是右边，所以要取绝对值：

```java
int curGap = Math.abs(sum - target);
```

例如：

```text
target = 10
sum = 7   -> sum - target = -3 -> Math.abs(-3) = 3
sum = 13  -> sum - target =  3 -> Math.abs(3)  = 3
```

`7` 和 `13` 距离 `10` 都是 3。

## Math.abs 是什么？

`Math.abs(x)`：返回 `x` 的绝对值。

```java
Math.abs(5);   // 5
Math.abs(-5);  // 5
Math.abs(0);   // 0
```

算法题里常用于计算两个数之间的差距：

```java
Math.abs(a - b)
```

可以直接理解成：

> a 和 b 相差多少。

## LC16 中怎么用

```java
int minGap = Integer.MAX_VALUE;
int ans = 0;

int sum = nums[i] + nums[left] + nums[right];
int curGap = Math.abs(sum - target);

if (curGap < minGap) {
    minGap = curGap;
    ans = sum;
}
```

注意：

- `minGap` 记录目前最小的距离
- `ans` 记录产生这个最小距离的三数之和
- 最后返回的是 `ans`，不是 `minGap`

## 双指针移动规则

数组先排序：

```java
Arrays.sort(nums);
```

然后固定 `i`，左右双指针扫描：

```java
if (sum < target) {
    left++;      // 和太小，需要变大
} else if (sum > target) {
    right--;     // 和太大，需要变小
} else {
    return target;
}
```

一句话模板：

> `Math.abs(sum - target)` 算距离；`sum < target` 左指针右移，`sum > target` 右指针左移。
