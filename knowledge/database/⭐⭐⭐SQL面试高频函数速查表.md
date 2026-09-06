# ⭐⭐⭐ SQL 面试高频函数速查表

这篇笔记专门记录 SQL 面试和 LeetCode SQL 中常见、但如果不知道函数名就很容易现场写不出来的函数。

---

## 一、日期

### 1. DATEDIFF(a, b)

作用：计算两个日期相差多少天，结果是 `a - b`。

```sql
SELECT DATEDIFF('2026-09-06', '2026-09-05');
```

结果：

```text
1
```

LC197 上升的温度里：

```sql
DATEDIFF(a.recordDate, b.recordDate) = 1
```

表示 `a` 是 `b` 的后一天。

---

### 2. DATE_FORMAT(date, format)

作用：把日期格式化成想要的字符串。

```sql
DATE_FORMAT(trans_date, '%Y-%m')
```

例如：

```text
2026-09-06
↓
2026-09
```

常见格式符：

```text
%Y = 2026
%y = 26
%m = 09
%M = September
%d = 06
```

注意：大小写有区别。

```sql
'%Y-%m'
```

得到：

```text
2026-09
```

而：

```sql
'%Y-%M'
```

得到：

```text
2026-September
```

按月统计时特别常用：

```sql
GROUP BY DATE_FORMAT(trans_date, '%Y-%m')
```

LC1193 每月交易 I 会用到。

---

### 3. YEAR(date) / MONTH(date)

作用：直接取年份、月份。

```sql
YEAR('2026-09-06')  → 2026
MONTH('2026-09-06') → 9
```

例如：

```sql
WHERE YEAR(order_date) = 2026
AND MONTH(order_date) = 9
```

区别：

```text
DATE_FORMAT → 可以得到 2026-09 这种组合
YEAR / MONTH → 单独提取年月
```

---

### 4. BETWEEN start AND end

作用：判断值是否处于闭区间 `[start, end]`。

```sql
purchase_date BETWEEN start_date AND end_date
```

等价于：

```sql
purchase_date >= start_date
AND purchase_date <= end_date
```

注意：两边都包含。

---

# 二、空值

## IFNULL(x, default)

作用：如果 `x` 是 NULL，就使用默认值。

```sql
IFNULL(score, 0)
```

```text
score = 90   → 90
score = NULL → 0
```

经常和聚合一起：

```sql
IFNULL(AVG(score), 0)
```

可以理解成 Java：

```java
x != null ? x : default
```

---

# 三、聚合

聚合函数本质上都是：对一组数据做计算。

经常和：

```sql
GROUP BY
```

一起出现。

---

## 1. COUNT(*)

统计所有行数。

```sql
SELECT COUNT(*)
FROM Employee;
```

即使某些字段是 NULL，这一行也照样算。

---

## 2. COUNT(col)

统计 `col` 不为 NULL 的行数。

例如：

```text
score
100
90
NULL
```

那么：

```text
COUNT(*)     = 3
COUNT(score) = 2
```

重要：

```text
AVG(x) = SUM(x) / COUNT(x)
```

如果 `x` 可能有 NULL，不要随便写：

```sql
SUM(x) / COUNT(*)
```

因为 `COUNT(*)` 会把 NULL 那一行也算进去。

---

## 3. SUM(x)

求和。

```sql
SUM(amount)
```

例如：

```text
10
20
30
```

得到：

```text
60
```

MySQL 里一个很实用的写法：

```sql
SUM(action = 'confirmed')
```

因为：

```text
条件成立   → 1
条件不成立 → 0
```

所以它可以直接统计 confirmed 次数。

还可以写：

```sql
SUM((state = 'approved') * amount)
```

含义：

```text
approved     → 1 * amount = amount
not approved → 0 * amount = 0
```

所以可以直接统计 approved 金额总和。

---

## 4. AVG(x)

求平均。

```sql
AVG(score)
```

也可以放表达式：

```sql
AVG(end_time - start_time)
```

表示先逐行算 `end_time - start_time`，再求平均。

MySQL 中还可以放条件：

```sql
AVG(action = 'confirmed')
```

这时：

```text
成立 → 1
不成立 → 0
```

所以 AVG 就是条件成立比例。

注意：`AVG` 会忽略 NULL。

---

## 5. MAX(x) / MIN(x)

最大值、最小值。

```sql
MAX(score)
MIN(score)
```

非常常见的思路：

```sql
HAVING MIN(score) >= 60
```

表示：所有科目都 >= 60。

因为连最低分都 >= 60。

而：

```sql
HAVING MAX(score) >= 90
```

表示：至少有一科 >= 90。

---

# 四、字符串

## 1. LENGTH(str) / CHAR_LENGTH(str)

这两个不要完全当成一样。

```sql
LENGTH(str)
```

统计的是字节数。

```sql
CHAR_LENGTH(str)
```

统计的是字符数。

英文：

```sql
LENGTH('abc') = 3
CHAR_LENGTH('abc') = 3
```

中文在 UTF-8 下：

```sql
LENGTH('你好')
```

可能是：

```text
6
```

而：

```sql
CHAR_LENGTH('你好')
```

是：

```text
2
```

所以想算“有几个字符”，优先 `CHAR_LENGTH`。

---

## 2. LOWER() / UPPER()

转换大小写。

```sql
LOWER('HELLO')
```

结果：

```text
hello
```

```sql
UPPER('hello')
```

结果：

```text
HELLO
```

例如：

```sql
WHERE LOWER(email) = 'abc@gmail.com'
```

注意：MySQL 某些排序规则本身就可能不区分大小写。

---

## 3. CONCAT(...)

字符串拼接。

```sql
CONCAT(first_name, ' ', last_name)
```

例如：

```text
Yu + Yue
↓
Yu Yue
```

完整写法：

```sql
SELECT CONCAT(first_name, ' ', last_name) AS full_name
FROM User;
```

---

## 4. SUBSTRING(str, start, length)

截取字符串。

```sql
SUBSTRING('abcdef', 2, 3)
```

结果：

```text
bcd
```

注意：MySQL 下标从 1 开始。

```text
abcdef
123456
```

例如：

```sql
SUBSTRING(name, 1, 1)
```

表示取第一个字符。

---

# 五、条件

## 1. CASE WHEN

SQL 里的 `if / else if / else`。

模板：

```sql
CASE
    WHEN 条件1 THEN 结果1
    WHEN 条件2 THEN 结果2
    ELSE 结果3
END
```

例如：

```sql
SELECT
    name,
    CASE
        WHEN score >= 90 THEN 'A'
        WHEN score >= 60 THEN 'B'
        ELSE 'C'
    END AS level
FROM Student;
```

可以理解成 Java：

```java
if (...) {
} else if (...) {
} else {
}
```

---

## 2. IF(condition, a, b)

MySQL 简化版三元表达式。

```sql
IF(score >= 60, 'pass', 'fail')
```

相当于 Java：

```java
score >= 60 ? "pass" : "fail"
```

例如：

```sql
SUM(IF(state = 'approved', amount, 0))
```

含义：

```text
approved → 加 amount
否则     → 加 0
```

它和：

```sql
SUM(
    CASE
        WHEN state = 'approved' THEN amount
        ELSE 0
    END
)
```

本质一致。

速记：

```text
简单二选一 → IF
复杂多条件 → CASE WHEN
```

---

# 六、排序排名：窗口函数

假设成绩：

```text
score
100
100
90
80
```

---

## 1. ROW_NUMBER()

强制每一行一个不同排名。

```sql
ROW_NUMBER() OVER (
    ORDER BY score DESC
)
```

结果：

```text
100 → 1
100 → 2
90  → 3
80  → 4
```

同分也不会并列。

---

## 2. RANK()

允许并列，但是会跳号。

```sql
RANK() OVER (
    ORDER BY score DESC
)
```

结果：

```text
100 → 1
100 → 1
90  → 3
80  → 4
```

没有第 2 名。

---

## 3. DENSE_RANK()

允许并列，而且不跳号。

```sql
DENSE_RANK() OVER (
    ORDER BY score DESC
)
```

结果：

```text
100 → 1
100 → 1
90  → 2
80  → 3
```

做“前三个不同工资”这类题时非常常用。

---

# 七、PARTITION BY

窗口函数经常不是全表排名，而是每组内部排名。

例如：

```sql
DENSE_RANK() OVER (
    PARTITION BY department_id
    ORDER BY salary DESC
)
```

含义：

```text
每个 department
    ↓
自己单独排名
```

对应“每个部门工资前三”。

再比如：

```sql
ROW_NUMBER() OVER (
    PARTITION BY subject
    ORDER BY score DESC
)
```

就是：每个科目内部单独给学生排名。

---

# 最终脑图

```text
日期
DATEDIFF(a,b)        → 相差几天
DATE_FORMAT          → 日期转格式
YEAR / MONTH         → 取年月
BETWEEN              → 区间

NULL
IFNULL(x,0)          → NULL 补默认值

聚合
COUNT                → 数数量
SUM                  → 求和
AVG                  → 平均
MAX / MIN            → 最大最小

字符串
CHAR_LENGTH          → 字符个数
LOWER / UPPER        → 大小写
CONCAT               → 拼接
SUBSTRING            → 截取

条件
IF                    → 二选一
CASE WHEN             → 多分支

排名
ROW_NUMBER            → 不并列
RANK                  → 并列，跳号
DENSE_RANK            → 并列，不跳号
PARTITION BY          → 每组分别排名
```

---

# 当前最应该优先熟练

优先把下面这些练熟：

```text
CASE WHEN
IF
DATE_FORMAT
DATEDIFF
SUBSTRING
DENSE_RANK + PARTITION BY
```

这些属于：逻辑可能想得出来，但如果不知道函数或语法名字，现场很容易完全写不出来的知识点。
