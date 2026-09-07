# ⭐⭐⭐ SQL 面试高频函数速查表

这篇笔记专门记录 SQL 面试和 LeetCode SQL 中常见、但如果不知道函数名就很容易现场写不出来的函数。

每个函数都按下面四件事记：

```text
1. 什么时候用
2. 基本写法
3. 典型例子
4. 易错点
```

---

## 一、日期

### 1. DATEDIFF(a, b)

**什么时候用：** 判断两个日期相差几天，例如“前一天”“隔 7 天”“两个日期是否连续”。

作用：计算两个日期相差多少天，结果是 `a - b`。

基本写法：

```sql
DATEDIFF(date1, date2)
```

例如：

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

**易错点：** 参数顺序不能反，`DATEDIFF(a,b)` 算的是 `a-b`。

---

### 2. DATE_FORMAT(date, format)

**什么时候用：** 需要把完整日期转换成“年”“年月”“年月日”等格式，尤其常用于“按月统计”。

作用：把日期格式化成想要的字符串。

基本写法：

```sql
DATE_FORMAT(date, format)
```

最常用：

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

按月统计时：

```sql
SELECT
    DATE_FORMAT(trans_date, '%Y-%m') AS month,
    COUNT(*)
FROM Transactions
GROUP BY DATE_FORMAT(trans_date, '%Y-%m');
```

LC1193 每月交易 I 会用到。

**易错点：** `%m` 是数字月份，`%M` 是英文月份全称。

---

### 3. YEAR(date) / MONTH(date)

**什么时候用：** 只需要单独取年份或者月份进行筛选、分组。

作用：直接取年份、月份。

```sql
YEAR('2026-09-06')  → 2026
MONTH('2026-09-06') → 9
```

典型筛选：

```sql
WHERE YEAR(order_date) = 2026
AND MONTH(order_date) = 9
```

典型分组：

```sql
SELECT YEAR(order_date), COUNT(*)
FROM Orders
GROUP BY YEAR(order_date);
```

区别：

```text
DATE_FORMAT → 可以直接得到 2026-09 这种组合
YEAR / MONTH → 单独提取年份或月份
```

---

### 4. BETWEEN start AND end

**什么时候用：** 判断数字、日期是否落在某个区间内。

作用：判断值是否处于闭区间 `[start, end]`。

```sql
purchase_date BETWEEN start_date AND end_date
```

等价于：

```sql
purchase_date >= start_date
AND purchase_date <= end_date
```

典型场景：某一天是否落在商品价格生效区间内。

```sql
ON u.purchase_date BETWEEN p.start_date AND p.end_date
```

**易错点：** 两边都包含。

---

# 二、空值

## IFNULL(x, default)

**什么时候用：** LEFT JOIN 没匹配到数据、聚合结果可能为 NULL，但题目要求返回 0 或其他默认值。

作用：如果 `x` 是 NULL，就使用默认值。

基本写法：

```sql
IFNULL(x, default)
```

例如：

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

以及：

```sql
IFNULL(SUM(amount), 0)
```

可以理解成 Java：

```java
x != null ? x : default
```

---

# 三、聚合

聚合函数本质上都是：**对当前这一组数据做计算。**

经常和：

```sql
GROUP BY
```

一起出现。

---

## 1. COUNT(*)

**什么时候用：** 统计一共有多少行，不关心某个字段是否为 NULL。

```sql
SELECT COUNT(*)
FROM Employee;
```

即使某些字段是 NULL，这一行也照样算。

典型分组：

```sql
SELECT managerId, COUNT(*)
FROM Employee
GROUP BY managerId;
```

表示统计每个经理有多少条下属记录。

---

## 2. COUNT(col)

**什么时候用：** 只想统计某个字段不为 NULL 的行。LEFT JOIN 后统计右表匹配数量时尤其重要。

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

LEFT JOIN 时：

```sql
COUNT(e.subject_name)
```

可以让“没参加考试”的组合统计成 0；如果写 `COUNT(*)`，LEFT JOIN 保留下来的空行也会被算成 1。

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

**什么时候用：** 求金额、数量等字段总和；也可以利用 MySQL 的布尔值 0/1 做条件统计。

普通求和：

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

MySQL 中条件成立为 1，不成立为 0，所以：

```sql
SUM(action = 'confirmed')
```

可以直接统计 confirmed 次数。

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

更通用的写法：

```sql
SUM(CASE WHEN state = 'approved' THEN amount ELSE 0 END)
```

---

## 4. AVG(x)

**什么时候用：** 求平均值、平均耗时、比例。

普通平均：

```sql
AVG(score)
```

表达式平均：

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

例如 LC1934 确认率：

```sql
ROUND(IFNULL(AVG(c.action = 'confirmed'), 0), 2)
```

**易错点：** `AVG` 会忽略 NULL。

---

## 5. MAX(x) / MIN(x)

**什么时候用：** 求最大值、最小值；还可以利用极值判断“至少一个”或“全部满足”。

普通用法：

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

还可以求每组最高分：

```sql
SELECT subject, MAX(score)
FROM student_score
GROUP BY subject;
```

---

# 四、字符串

## 1. LENGTH(str) / CHAR_LENGTH(str)

**什么时候用：** 判断字符串长度，例如用户名长度、内容长度。

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

通常是：

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

所以想算“有几个字符”，优先：

```sql
CHAR_LENGTH(str)
```

---

## 2. LOWER() / UPPER()

**什么时候用：** 大小写标准化、统一输出格式、进行大小写转换。

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
SELECT LOWER(email)
FROM User;
```

**注意：** MySQL 某些排序规则本身就不区分大小写，所以不要默认认为比较字符串一定需要 `LOWER()`。

---

## 3. CONCAT(...)

**什么时候用：** 把多个字段、常量拼成一个完整字符串。

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

也可以：

```sql
CONCAT(city, '-', district)
```

---

## 4. SUBSTRING(str, start, length)

**什么时候用：** 截取字符串的一部分，例如取首字母、手机号部分、固定位置编码。

基本写法：

```sql
SUBSTRING(str, start, length)
```

例如：

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

也可以只给开始位置：

```sql
SUBSTRING('abcdef', 3)
```

结果：

```text
cdef
```

---

# 五、条件

## 1. CASE WHEN

**什么时候用：** 根据不同条件返回不同值；做条件分类、条件聚合、状态映射时特别常见。

SQL 里的 `if / else if / else`。

模板：

```sql
CASE
    WHEN 条件1 THEN 结果1
    WHEN 条件2 THEN 结果2
    ELSE 结果3
END
```

例如成绩分级：

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

条件聚合：

```sql
SUM(
    CASE
        WHEN state = 'approved' THEN amount
        ELSE 0
    END
)
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

**什么时候用：** 只有两个结果时，用它比 `CASE WHEN` 更短。

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

窗口函数最重要的整体结构：

```sql
排名函数() OVER (
    PARTITION BY 分组字段
    ORDER BY 排序字段
)
```

其中：

```text
PARTITION BY → 每组分别算
ORDER BY     → 组内按什么顺序排
```

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

**什么时候用：** 每一行必须拥有唯一序号，例如“每组只取第一条”“每组前三条记录”。

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

典型 Top3：

```sql
SELECT *
FROM (
    SELECT
        student_id,
        subject,
        score,
        ROW_NUMBER() OVER (
            PARTITION BY subject
            ORDER BY score DESC
        ) AS rn
    FROM student_score
) t
WHERE rn <= 3;
```

---

## 2. RANK()

**什么时候用：** 允许并列，并且希望后面的名次按照真实占位跳号。

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

**什么时候用：** 允许并列，但不希望跳号；“前 N 个不同工资/不同分数”特别常见。

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

**什么时候用：** 不是对全表统一排名，而是要求“每个部门”“每个科目”“每个用户”分别计算排名、序号、累计值。

例如：

```sql
DENSE_RANK() OVER (
    PARTITION BY department_id
    ORDER BY salary DESC
)
```

含义：

```text
先按 department_id 分组
        ↓
每个部门内部
        ↓
按照 salary DESC 单独排名
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

注意：`PARTITION BY` 和普通 `GROUP BY` 不一样。

```text
GROUP BY
→ 多行聚合成一行

PARTITION BY
→ 原来的每一行仍然保留，只是在组内额外计算排名等结果
```

---

# 最终脑图

```text
日期
DATEDIFF(a,b)        → 相差几天
DATE_FORMAT(字段,'%Y-%m')  → 日期转格式 / 按月统计
YEAR / MONTH         → 取年月
BETWEEN              → 判断区间

NULL
IFNULL(x,0)          → NULL 补默认值

聚合
COUNT(*)             → 所有行
COUNT(col)           → col 非 NULL 行
SUM                   → 求和 / 条件计数
AVG                   → 平均 / 条件比例
MAX / MIN             → 最大最小 / 判断至少一个或全部

字符串
CHAR_LENGTH          → 字符个数
LOWER / UPPER        → 大小写转换
CONCAT               → 拼接
SUBSTRING            → 截取

条件
IF                    → 二选一
CASE WHEN             → 多分支 / 条件聚合

排名
ROW_NUMBER            → 不并列
RANK                  → 并列，跳号
DENSE_RANK            → 并列，不跳号
PARTITION BY          → 每组分别计算，但不压缩行
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

这些属于：**逻辑可能想得出来，但如果不知道函数或语法名字，现场很容易完全写不出来的知识点。**
