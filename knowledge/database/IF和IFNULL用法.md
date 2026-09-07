# IF 和 IFNULL 用法

## 一、IF：条件判断

语法：

```sql
IF(条件, 条件成立时返回值, 条件不成立时返回值)
```

例如：

```sql
SELECT IF(score >= 60, '及格', '不及格')
FROM student;
```

执行逻辑：

```text
score >= 60
    ↓
true  → '及格'
false → '不及格'
```

### 常见用法 1：条件计数

```sql
SUM(IF(state = 'approved', 1, 0))
```

MySQL 中还可以简写成：

```sql
SUM(state = 'approved')
```

### 常见用法 2：条件求和

```sql
SUM(IF(state = 'approved', amount, 0))
```

含义：

```text
state = 'approved'
        ↓
是 → 加 amount
否 → 加 0
```

---

## 二、IFNULL：处理 NULL

语法：

```sql
IFNULL(字段, 字段为 NULL 时返回的默认值)
```

例如：

```sql
SELECT IFNULL(name, '未知')
FROM user;
```

执行逻辑：

```text
如果 name != NULL
    ↓
返回 name

如果 name == NULL
    ↓
返回 '未知'
```

### 常见用法 1：NULL 转 0

```sql
IFNULL(amount, 0)
```

### 常见用法 2：避免 NULL 参与计算

如果：

```sql
price * count
```

而 `count` 为 NULL，那么结果也是 NULL。

可以改成：

```sql
price * IFNULL(count, 0)
```

---

## 三、IF 和 IFNULL 的区别

| 函数 | 作用 | 写法 |
|---|---|---|
| `IF` | 判断一个条件 | `IF(condition, true_value, false_value)` |
| `IFNULL` | 判断一个值是否为 NULL | `IFNULL(value, default_value)` |

记忆：

```text
IF      → 判断条件
IFNULL  → 判断 NULL
```

例如：

```sql
IF(age >= 18, '成年', '未成年')
```

是在判断条件：

```text
age >= 18 ?
```

而：

```sql
IFNULL(age, 0)
```

是在判断：

```text
age 是不是 NULL？
```

---

## 四、补充：COALESCE

```sql
COALESCE(a, b, c, 0)
```

会返回第一个不是 NULL 的值。

例如：

```sql
COALESCE(NULL, NULL, 3, 4)
```

结果：

```text
3
```

因此：

```sql
IFNULL(a, b)
```

可以看成：

```sql
COALESCE(a, b)
```

的两参数版本。
