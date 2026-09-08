# LC610 IF 函数判断三角形

## 题目

LeetCode 610：判断三角形（Triangle Judgement）

给定三条边 `x`、`y`、`z`，判断是否可以组成三角形。

三角形成立条件：

```text
x + y > z
x + z > y
y + z > x
```

三条必须同时成立。

---

## IF 的基本用法

MySQL 中：

```sql
IF(condition, value_if_true, value_if_false)
```

含义：

```text
condition 成立   → 返回 value_if_true
condition 不成立 → 返回 value_if_false
```

可以理解成 Java 三元表达式：

```java
condition ? a : b
```

---

## LC610 写法

```sql
SELECT
    x,
    y,
    z,
    IF(
        x + y > z
        AND x + z > y
        AND y + z > x,
        'Yes',
        'No'
    ) AS triangle
FROM Triangle;
```

逻辑：

```text
三个条件全部成立
→ IF 返回 'Yes'

否则
→ IF 返回 'No'
```

---

## 什么时候用 IF

只有两个结果时，`IF` 比 `CASE WHEN` 更短。

例如：

```sql
IF(score >= 60, 'pass', 'fail')
```

相当于：

```sql
CASE
    WHEN score >= 60 THEN 'pass'
    ELSE 'fail'
END
```

再比如条件聚合：

```sql
SUM(IF(state = 'approved', amount, 0))
```

表示：

```text
approved → 加 amount
否则     → 加 0
```

---

## 速记

```text
IF(condition, a, b)
= condition ? a : b
```

```text
简单二选一 → IF
复杂多条件 → CASE WHEN
```
