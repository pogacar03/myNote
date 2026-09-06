# SQL 中 AVG、SUM 和 IFNULL 怎么用？

## 对应力扣题

- **LC1934：确认率（Confirmation Rate）**
- 重点：`AVG(condition)` 计算比例，`IFNULL(..., 0)` 处理没有确认记录的用户。

```sql
SELECT
    s.user_id,
    ROUND(IFNULL(AVG(c.action = 'confirmed'), 0), 2) AS confirmation_rate
FROM Signups s
LEFT JOIN Confirmations c
ON s.user_id = c.user_id
GROUP BY s.user_id;
```

---

## AVG

### 1. 普通求平均值

```sql
SELECT AVG(score)
FROM student_score;
```

`AVG(column)` 会忽略 `NULL`。

### 2. 分组后求每组平均值

```sql
SELECT student_id, AVG(score)
FROM student_score
GROUP BY student_id;
```

`GROUP BY 谁`，就是对谁这一组求平均值。

### 3. 过滤平均值

```sql
SELECT student_id
FROM student_score
GROUP BY student_id
HAVING AVG(score) > 80;
```

聚合结果要用 `HAVING`，不能用 `WHERE`。

### 4. AVG 里放表达式

```sql
AVG(end_time - start_time)
```

表示先对每一行计算 `end_time - start_time`，再对这些结果求平均。

### 5. MySQL 中 AVG(条件)

```sql
AVG(action = 'confirmed')
```

MySQL 中：

```text
条件成立   -> 1
条件不成立 -> 0
NULL       -> NULL
```

所以：

```sql
AVG(action = 'confirmed')
```

可以直接表示 confirmed 所占比例。

更通用的写法：

```sql
AVG(CASE WHEN action = 'confirmed' THEN 1 ELSE 0 END)
```

---

## SUM

### 1. 普通求和

```sql
SELECT SUM(score)
FROM student_score;
```

### 2. 分组求和

```sql
SELECT student_id, SUM(score)
FROM student_score
GROUP BY student_id;
```

### 3. MySQL 中 SUM(条件)

```sql
SUM(action = 'confirmed')
```

因为条件成立为 1、不成立为 0，所以它表示 confirmed 的次数。

例如：

```text
confirmed -> 1
confirmed -> 1
timeout   -> 0
```

则：

```text
SUM = 2
```

更通用的写法：

```sql
SUM(CASE WHEN action = 'confirmed' THEN 1 ELSE 0 END)
```

---

## IFNULL

```sql
IFNULL(value, default_value)
```

含义：

```text
value 不是 NULL -> 返回 value
value 是 NULL   -> 返回 default_value
```

例如：

```sql
IFNULL(score, 0)
```

```sql
IFNULL(AVG(score), 0)
```

可以粗略理解成 Java：

```java
value != null ? value : defaultValue
```

也有点像：

```java
map.getOrDefault(key, defaultValue)
```

但更准确地说，`IFNULL` 判断的是“表达式结果是否为 NULL”。

---

## 最终速记

```text
AVG(x)
= 对当前组里的 x 求平均

SUM(x)
= 对当前组里的 x 求和

AVG(condition)
= 条件成立比例（MySQL）

SUM(condition)
= 条件成立次数（MySQL）

IFNULL(x, default)
= x 为 NULL 时使用默认值
```
