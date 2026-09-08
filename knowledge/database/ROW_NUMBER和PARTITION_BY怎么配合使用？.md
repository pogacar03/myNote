# ROW_NUMBER 和 PARTITION BY 怎么配合使用？

## 核心作用

`ROW_NUMBER() OVER (...)` 用来按照 `OVER(...)` 里定义的分组和排序规则，给每一行编号。

其中：

```text
ROW_NUMBER()  → 给行编号
PARTITION BY  → 按什么字段分组，每组重新从 1 开始
ORDER BY      → 每组内部按什么顺序编号
```

---

## 例子

假设数据：

```text
product_id | change_date
1          | 2026-09-01
1          | 2026-08-20
2          | 2026-09-03
2          | 2026-08-01
```

加上：

```sql
ROW_NUMBER() OVER (
    PARTITION BY product_id
    ORDER BY change_date DESC
) AS rn
```

结果：

```text
product_id | change_date | rn
1          | 2026-09-01  | 1
1          | 2026-08-20  | 2
2          | 2026-09-03  | 1
2          | 2026-08-01  | 2
```

解释：

```text
先按 product_id 分组
        ↓
每个 product_id 单独排序
        ↓
按 change_date DESC 从新到旧
        ↓
每组从 1 开始编号
```

所以：

```sql
WHERE rn = 1
```

就表示：

> 每个 product_id 只保留最新的一条记录。

---

## 常用模板

```sql
SELECT *
FROM (
    SELECT
        *,
        ROW_NUMBER() OVER (
            PARTITION BY 分组字段
            ORDER BY 排序字段 DESC
        ) AS rn
    FROM 表名
) t
WHERE rn = 1;
```

常见用途：

```text
每组取最新一条
每组取最早一条
每组取 Top N
每个班级前 10 名
每个商品最新价格
```

如果要每组 Top 10：

```sql
WHERE rn <= 10
```
