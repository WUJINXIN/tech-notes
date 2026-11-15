# PostgreSQL 数据类型通俗讲解

🌐 **简体中文** | [English](postgresql_data_types_en.md)

> 主题：`NUMERIC` vs `DOUBLE PRECISION`；`VARCHAR` vs `TEXT`；`TIMESTAMP` vs `TIMESTAMPTZ`。

## 1. 为什么金额使用 `NUMERIC`，而不是 `DOUBLE PRECISION`？

**大白话：钱要算得准；`NUMERIC` 保存精确十进制数，`DOUBLE PRECISION` 保存近似浮点数。**

想象你在记账。0.1 元加 0.2 元，账本上就应该准确得到 0.3 元。但是计算机用二进制浮点数表示 `0.1`、`0.2` 时，往往只能保存非常接近它们的数。

Java 示例：

```java
double a = 0.1;
double b = 0.2;
System.out.println(a + b); // 0.30000000000000004
```

用 `BigDecimal` 处理精确金额：

```java
BigDecimal a = new BigDecimal("0.1");
BigDecimal b = new BigDecimal("0.2");
System.out.println(a.add(b)); // 0.3
```

PostgreSQL 表字段：

```sql
CREATE TABLE accounts (
    account_id BIGINT PRIMARY KEY,
    balance NUMERIC(12, 2) NOT NULL
);
```

`NUMERIC(12, 2)`：最多 **12 位十进制数字**，其中 **2 位是小数**，也就是整数部分最多 10 位。该字段会按定义的小数位规则进行舍入；这并不意味着所有写入值都能无限精确地原样保存。

| 对比 | `NUMERIC` | `DOUBLE PRECISION` |
|---|---|---|
| 保存方式 | 精确十进制 | 近似二进制浮点 |
| 典型用途 | 金额、财务计算 | 科学测量、允许误差的计算 |
| 速度与存储 | 通常计算较慢、空间可变 | 通常计算较快、固定 8 字节 |
| Java 常见对应 | `BigDecimal` | `double` / `Double` |

**记住：金额优先使用 `NUMERIC`，Java 中优先使用 `BigDecimal`。** 还有一种常见设计是用整数存储最小货币单位（例如分），但必须统一单位、注意溢出。

## 2. `VARCHAR` 和 `TEXT` 有什么区别？

**大白话：`VARCHAR(50)` 是“最多装 50 个字符的盒子”，`TEXT` 是“不预先指定字符数上限的盒子”。**

```sql
CREATE TABLE articles (
    title VARCHAR(100) NOT NULL,
    body TEXT NOT NULL
);
```

- `title` 最多 100 个字符。超出限制通常会报错。
- `body` 不指定字符数量上限，适合长文、说明和备注。
- 在 PostgreSQL 中，`VARCHAR(n)` 和 `TEXT` 使用相近的存储机制，**通常没有明显的性能差别**；选哪个主要看是否需要长度限制。
- `VARCHAR(100)` 的 `100` 表示**字符数**，不是字节数。UTF-8 下一个汉字通常占多个字节，但在这里计为一个字符。
- `TEXT` 并不是真的无限大：仍受到 PostgreSQL 字段大小等实际限制。

| 对比 | `VARCHAR(100)` | `TEXT` |
|---|---|---|
| 长度限制 | 最多 100 字符 | 不预设字符数上限 |
| 查询和索引 | 可以 | 可以 |
| 性能 | 通常相近 | 通常相近 |
| Java 常见对应 | `String` | `String` |

**记住：需要数据库帮你限制输入长度，用 `VARCHAR(n)`；不需要固定长度限制，用 `TEXT`。**

## 3. `TIMESTAMP` 和 `TIMESTAMPTZ` 有什么区别？

**大白话：`TIMESTAMP` 记录“钟表上写着几点”；`TIMESTAMPTZ` 记录“世界时间线上是哪一个瞬间”。**

假设东京时间是 **2026-11-15 20:00（UTC+09）**。同一瞬间：

| 地区 / 时区 | 当地显示 |
|---|---|
| 东京（UTC+09） | 2026-11-15 20:00 |
| 纽约（UTC−05，当天是标准时间） | 2026-11-15 06:00 |
| UTC | 2026-11-15 11:00 |

### `TIMESTAMP`（不带时区）

```sql
CREATE TABLE appointments (
    local_start TIMESTAMP
);
```

如果写入 `2026-11-15 20:00:00`，数据库保存这个本地日期和时间，**这个值本身没有说明它是东京还是纽约时间**。

### `TIMESTAMPTZ`（带时区语义）

```sql
CREATE TABLE events (
    created_at TIMESTAMPTZ NOT NULL
);

INSERT INTO events (created_at)
VALUES ('2026-11-15 20:00:00+09');
```

PostgreSQL 会把输入理解为一个确定的瞬间，并按查询会话的时区显示：

```sql
SET TIME ZONE 'Asia/Tokyo';
SELECT created_at FROM events;  -- 2026-11-15 20:00:00+09

SET TIME ZONE 'UTC';
SELECT created_at FROM events;  -- 2026-11-15 11:00:00+00
```

**重要：`TIMESTAMPTZ` 不保存原始时区名称（例如 `Asia/Tokyo`）。** 它保存的是时间点，查询时按会话时区转换显示。若业务需要记住“用户选择了东京时区”，可另外存储时区标识。

| 典型数据 | 推荐类型 | 原因 |
|---|---|---|
| 生日 | `DATE` | 只需要日期 |
| 订单创建时间、系统日志 | `TIMESTAMPTZ` | 需要确定的全球时间点 |
| 不附带时区的本地日程 | `TIMESTAMP` | 只表达当地钟表时间；跨时区日程常常还需单独保存时区 |

Java 常见映射：`TIMESTAMP` ↔ `LocalDateTime`；`TIMESTAMPTZ` ↔ `OffsetDateTime`（具体取决于 JDBC 驱动及框架配置）。

### 中文速记

- **`NUMERIC` vs `DOUBLE`**：精确十进制 vs 近似浮点；金额选前者。
- **`VARCHAR(n)` vs `TEXT`**：限制字符数 vs 不预设字符数限制；性能通常相近。
- **`TIMESTAMP` vs `TIMESTAMPTZ`**：本地钟表时间 vs 确定的时间点。
