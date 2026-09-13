# LC18 四数之和：命中 target 后为什么必须去重？long 为什么要提前强转？

## 1. 核心结构

LC18 的标准做法：

```java
Arrays.sort(nums);

for (int i = 0; i < n - 3; i++) {
    // i 去重

    for (int j = i + 1; j < n - 2; j++) {
        // j 去重

        int left = j + 1;
        int right = n - 1;

        while (left < right) {
            ...
        }
    }
}
```

可以记成：

```text
4Sum = 两层 for + 双指针
```

---

## 2. 为什么 sum == target 后必须去重？

当：

```java
sum == target
```

说明当前四个数已经组成了一个合法答案：

```java
res.add(Arrays.asList(nums[i], nums[j], nums[left], nums[right]));
```

这时如果只是：

```java
left++;
right--;
```

但不跳过重复值，下一轮可能又得到完全相同的四元组，导致结果重复。

因此找到答案后要：

```java
left++;
right--;

while (left < right && nums[left] == nums[left - 1]) {
    left++;
}

while (left < right && nums[right] == nums[right + 1]) {
    right--;
}
```

### 记忆

```text
找到答案后去重：必须做
因为它直接关系到结果里会不会出现重复四元组。
```

而在：

```java
sum < target
```

或者：

```java
sum > target
```

时没有找到答案，此时跳过重复值主要是减少重复计算，所以去重更偏向优化；不写也不影响最终正确性。

一句话：

```text
没找到答案时，去重主要是优化；
找到答案后，去重是为了保证结果不重复。
```

---

## 3. 双指针去重下标怎么记？

原则：

> 指针移动后，和它刚刚走过来的那个位置比较。

左指针先：

```java
left++;
```

所以比较：

```java
nums[left] == nums[left - 1]
```

右指针先：

```java
right--;
```

所以比较：

```java
nums[right] == nums[right + 1]
```

---

## 4. long 强转为什么必须写在第一个数前面？

错误写法：

```java
long sum = (long) (nums[i] + nums[j] + nums[left] + nums[right]);
```

这段代码会先让四个 `int` 做加法。

也就是说：

```text
int + int + int + int
```

如果中间结果超过 `int` 范围，会先发生溢出。

等整个表达式已经算错之后，再转成 `long` 已经来不及了。

正确写法：

```java
long sum = (long) nums[i]
        + nums[j]
        + nums[left]
        + nums[right];
```

因为第一个操作数已经是 `long`，后面的整个加法都会按 `long` 计算。

### 记忆

```text
(long)(a + b + c + d)   // 错：先 int 相加，再转 long

(long)a + b + c + d     // 对：从一开始就按 long 计算
```

口诀：

> 强转要发生在运算之前，不是运算之后。

---

## 5. 推荐模板

```java
class Solution {
    public List<List<Integer>> fourSum(int[] nums, int target) {
        List<List<Integer>> res = new ArrayList<>();
        int n = nums.length;

        Arrays.sort(nums);

        for (int i = 0; i < n - 3; i++) {
            if (i > 0 && nums[i] == nums[i - 1]) {
                continue;
            }

            for (int j = i + 1; j < n - 2; j++) {
                if (j > i + 1 && nums[j] == nums[j - 1]) {
                    continue;
                }

                int left = j + 1;
                int right = n - 1;

                while (left < right) {
                    long sum = (long) nums[i]
                            + nums[j]
                            + nums[left]
                            + nums[right];

                    if (sum == target) {
                        res.add(Arrays.asList(
                                nums[i], nums[j], nums[left], nums[right]
                        ));

                        left++;
                        right--;

                        while (left < right && nums[left] == nums[left - 1]) {
                            left++;
                        }

                        while (left < right && nums[right] == nums[right + 1]) {
                            right--;
                        }

                    } else if (sum < target) {
                        left++;
                    } else {
                        right--;
                    }
                }
            }
        }

        return res;
    }
}
```

## 最后记忆

```text
LC18 = 两层 for + 双指针

sum == target：
加入答案 → 左右移动 → 必须去重

long：
(long)a + b + c + d
不要写成 (long)(a + b + c + d)
```
