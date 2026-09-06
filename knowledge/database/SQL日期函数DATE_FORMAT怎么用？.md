# SQL 日期函数 DATE_FORMAT 怎么用？

## 对应力扣题

- **LC1193：每月交易 I**
- 重点：把日期格式化成 `YYYY-MM`，然后按“月份 + 国家”分组。

## 核心写法

```sql
DATE_FORMAT(trans_date, '%Y-%m')
```

含义：

```text
2026-09-06
→ 2026-09
```

所以需要“按月统计”时，可以直接写：

```sql
SELECT
    DATE_FORMAT(trans_date, '%Y-%m') AS month,
    country,
    COUNT(*)
FROM Transactions
GROUP BY month, country;
```

## 常见格式

这些格式符是 MySQL 规定好的，**大小写含义不同**：

```text
%Y  → 四位年份，例如 2026
%y  → 两位年份，例如 26
%m  → 两位月份，例如 09
%M  → 月份英文全称，例如 September
%d  → 两位日期，例如 06
```

因此：

```sql
DATE_FORMAT('2026-09-06', '%Y-%m')
```

得到：

```text
2026-09
```

而：

```sql
DATE_FORMAT('2026-09-06', '%Y-%M')
```

得到类似：

```text
2026-September
```

所以 LC1193 要输出 `2026-09`，必须写：

```sql
'%Y-%m'
```

而不是：

```sql
'%Y-%M'
```

## 最终速记

```text
%Y = 2026
%y = 26
%m = 09
%M = September
%d = 06
```

```text
按月分组
→ DATE_FORMAT(date, '%Y-%m')

按天格式化
→ DATE_FORMAT(date, '%Y-%m-%d')
```

LC1193 里最关键的是：

```sql
DATE_FORMAT(trans_date, '%Y-%m')
```

如果不知道这个函数，逻辑可能想得出来，但很难把“按年月分组”准确写出来。
