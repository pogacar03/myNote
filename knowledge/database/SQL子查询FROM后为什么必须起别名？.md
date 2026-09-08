# SQL 子查询 FROM 后为什么必须起别名？

## 核心规则

```text
只要是 FROM (子查询)，后面就必须给这个子查询结果起一个表名。
```

例如：

```sql
SELECT MAX(num) AS num
FROM (
    SELECT num
    FROM MyNumbers
    GROUP BY num
    HAVING COUNT(*) = 1
) t;
```

这里的：

```sql
) t
```

就是给子查询结果起别名 `t`。

也可以写成：

```sql
) AS t
```

两种写法都可以。

---

## 为什么必须起别名？

`FROM (...)` 里的子查询执行后，会产生一张临时结果表。

流程：

```text
子查询先执行
    ↓
得到一张临时结果表
    ↓
这张临时表需要一个名字
    ↓
外层查询再从这张表里继续查
```

所以：

```sql
FROM (
    SELECT ...
) t
```

可以理解为：

```text
FROM 临时表 t
```

---

## 错误写法

```sql
SELECT MAX(num)
FROM (
    SELECT num
    FROM MyNumbers
    GROUP BY num
    HAVING COUNT(*) = 1
);
```

MySQL 会报错，因为派生表没有别名。

---

## 最终速记

```text
FROM (子查询) t
```

记住：

```text
子查询放在 FROM 后面
→ 它会被当成一张临时表
→ 临时表必须有名字
→ 所以后面要写 t / AS t
```

对应题目：**LC619 只出现一次的最大数字**。
