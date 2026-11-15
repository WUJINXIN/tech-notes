# PostgreSQL Data Types Explained

🌐 [简体中文](postgresql_data_types_zh.md) | **English**

> Topics: `NUMERIC` vs `DOUBLE PRECISION`; `VARCHAR` vs `TEXT`; `TIMESTAMP` vs `TIMESTAMPTZ`.

## 1. Why use `NUMERIC` instead of `DOUBLE PRECISION` for money?

**Simply put: money needs exact decimal arithmetic. `NUMERIC` stores exact decimal values, while `DOUBLE PRECISION` stores approximate binary floating-point values.**

For example, `0.1 + 0.2` should equal `0.3`. Binary floating-point cannot represent many decimal fractions exactly:

```java
double a = 0.1;
double b = 0.2;
System.out.println(a + b); // 0.30000000000000004
```

Use `BigDecimal` in Java for precise decimal arithmetic:

```java
BigDecimal a = new BigDecimal("0.1");
BigDecimal b = new BigDecimal("0.2");
System.out.println(a.add(b)); // 0.3
```

In PostgreSQL:

```sql
CREATE TABLE accounts (
    account_id BIGINT PRIMARY KEY,
    balance NUMERIC(12, 2) NOT NULL
);
```

`NUMERIC(12, 2)` allows up to **12 decimal digits in total**, with **2 digits after the decimal point**. Values are rounded according to the declared scale, so this is not unlimited precision for every input.

| Feature | `NUMERIC` | `DOUBLE PRECISION` |
|---|---|---|
| Representation | Exact decimal | Approximate binary floating point |
| Typical use | Money, accounting | Measurements, calculations that tolerate rounding error |
| Performance/storage | Often slower arithmetic; variable size | Often faster arithmetic; fixed 8 bytes |
| Common Java type | `BigDecimal` | `double` / `Double` |

**Rule of thumb: use `NUMERIC` for monetary amounts and `BigDecimal` in Java.** Storing amounts as integer minor units (such as cents) is another valid design if the units are consistent and overflow is handled.

## 2. What is the difference between `VARCHAR` and `TEXT`?

**Think of `VARCHAR(50)` as a box that holds at most 50 characters; `TEXT` is a box without a predefined character limit.**

```sql
CREATE TABLE articles (
    title VARCHAR(100) NOT NULL,
    body TEXT NOT NULL
);
```

- `title` allows up to 100 characters; exceeding the limit normally causes an error.
- `body` has no declared character-count limit and suits long descriptions.
- In PostgreSQL, `VARCHAR(n)` and `TEXT` use similar storage, with **no significant performance difference in typical usage**.
- The `100` in `VARCHAR(100)` counts **characters, not bytes**.
- `TEXT` is not literally infinite; practical PostgreSQL field-size limits still apply.

| Feature | `VARCHAR(100)` | `TEXT` |
|---|---|---|
| Declared character limit | 100 | None |
| Indexable | Yes | Yes |
| Typical performance | Similar | Similar |
| Common Java type | `String` | `String` |

**Rule of thumb: use `VARCHAR(n)` when your business rules require a character limit; otherwise `TEXT` is often fine.**

## 3. What is the difference between `TIMESTAMP` and `TIMESTAMPTZ`?

**Simply put: `TIMESTAMP` stores a calendar date and clock time; `TIMESTAMPTZ` represents a specific instant in time.**

Suppose it is **20:00 in Tokyo on November 15, 2026 (UTC+09)**. The same instant is:

| Location / offset | Local display |
|---|---|
| Tokyo (UTC+09) | 2026-11-15 20:00 |
| New York (UTC−05, standard time on that date) | 2026-11-15 06:00 |
| UTC | 2026-11-15 11:00 |

### `TIMESTAMP` (without time zone)

```sql
CREATE TABLE appointments (
    local_start TIMESTAMP
);
```

A value such as `2026-11-15 20:00:00` records the local date and time, but **does not identify which time zone it belongs to**.

### `TIMESTAMPTZ` (with time-zone semantics)

```sql
CREATE TABLE events (
    created_at TIMESTAMPTZ NOT NULL
);

INSERT INTO events (created_at)
VALUES ('2026-11-15 20:00:00+09');
```

PostgreSQL interprets the offset and stores the instant. Its display depends on the session time zone:

```sql
SET TIME ZONE 'Asia/Tokyo';
SELECT created_at FROM events;  -- 2026-11-15 20:00:00+09

SET TIME ZONE 'UTC';
SELECT created_at FROM events;  -- 2026-11-15 11:00:00+00
```

**Important: `TIMESTAMPTZ` does not preserve the original time-zone name** (such as `Asia/Tokyo`). If your application needs to remember a user's chosen time zone, store that identifier separately.

| Use case | Recommended type | Why |
|---|---|---|
| Birthday | `DATE` | Only the date matters |
| Order creation time, application logs | `TIMESTAMPTZ` | A specific instant matters |
| Local appointment without a time-zone context | `TIMESTAMP` | The wall-clock value matters; scheduling across zones may require a separate zone identifier |

Common Java mappings: `TIMESTAMP` ↔ `LocalDateTime`; `TIMESTAMPTZ` ↔ `OffsetDateTime` (depending on the JDBC driver and framework).

### Quick recap

- **`NUMERIC` vs `DOUBLE`**: exact decimal vs approximate floating point; prefer `NUMERIC` for money.
- **`VARCHAR(n)` vs `TEXT`**: enforced character limit vs no declared character limit; typical performance is similar.
- **`TIMESTAMP` vs `TIMESTAMPTZ`**: wall-clock date/time vs a specific instant.
