---
title: MySQL 5.7 vs 8.0：子查询中的字符集（charset/collation）推断差异
date: 2026-06-11 10:00:00
tags:
  - MySQL
  - MySQL 5.7
  - MySQL 8.0
  - 字符集
  - Charset
  - Collation
  - 子查询
  - 数据库迁移
---

### 前言

字符集（charset）和校对规则（collation）问题一直是 MySQL 使用中的高频踩坑点。尤其在跨大版本升级时，很多团队会遭遇一个诡异的现象：**同一套 SQL，在 MySQL 5.7 上跑得好好的，到了 MySQL 8.0 上反而报错了**——或者反过来，5.7 上跑不通的 SQL，8.0 上却能正常执行。

本文将深入分析这个问题背后的根因：**MySQL 优化器在子查询中对字符集信息的推断能力差异**。

### 一、问题现象：一个典型的报错场景

假设我们有两张业务表，它们的 `name` 字段使用了不同的 collation：

```sql
CREATE TABLE t1 (
  id INT PRIMARY KEY,
  name VARCHAR(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci
);

CREATE TABLE t2 (
  id INT PRIMARY KEY,
  name VARCHAR(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci
);
```

注意：两张表的字符集都是 `utf8mb4`，但校对规则不同——一个是 `utf8mb4_unicode_ci`，一个是 `utf8mb4_general_ci`。这在真实业务中很常见，比如不同时期创建的表、不同微服务维护的表，都可能存在 collation 不一致的情况。

然后执行一条带子查询的 SQL：

```sql
SELECT *
FROM t1
WHERE t1.name IN (SELECT name FROM t2);
```

在 **MySQL 5.7** 中，这条 SQL 直接报错：

```
ERROR 1267 (HY000): Illegal mix of collations (utf8mb4_unicode_ci,IMPLICIT)
and (utf8mb4_general_ci,IMPLICIT) for operation '='
```

报错信息告诉我们：MySQL 无法将 `utf8mb4_unicode_ci` 和 `utf8mb4_general_ci` 这两种 collation 混合在一起做 `=` 比较。

但在 **MySQL 8.0** 中，同样两条 `CREATE TABLE` + 同一条 `SELECT`，**执行成功**，返回正确结果。

这就非常令人困惑了——为什么同样的 SQL，在 5.7 和 8.0 上的表现截然不同？

### 二、根因分析：MySQL 5.7 的子查询字符集推导缺陷

#### 2.1 MySQL 的 collation 强制等级（coercibility）

要理解这个问题的本质，首先需要了解 MySQL 是如何决定在比较两个字符串时使用哪个 collation 的。MySQL 为每个字符串表达式赋予了"强制等级"（coercibility），数值越小表示优先级越高：

| 强制等级 | 含义 | 示例 |
|---------|------|------|
| 0 | 显式 COLLATE 子句 | `name COLLATE utf8mb4_unicode_ci` |
| 1 | 两个不同 collation 字符串连接的结果 | `CONCAT(col1, col2)` |
| 2 | 列的值 | `t1.name` |
| 3 | 系统常量 | `USER()` |
| 4 | 字面量 | `'hello'` |
| 5 | NULL 或从 NULL 派生的表达式 | `NULL` |
| 6 | 子查询中的值 | `(SELECT name FROM t2)` |

当 MySQL 需要对两个字符串做比较时，它会选择强制等级数值更小的那个表达式所使用的 collation。如果两个表达式的强制等级相同但 collation 不同，MySQL 就会抛出 `Illegal mix of collations` 错误。

#### 2.2 MySQL 5.7 优化器的局限

问题的关键在于：**MySQL 5.7 的优化器在处理子查询时，无法正确地将子查询结果列的字符集信息"穿透"到外层查询**。

具体来说，在 MySQL 5.7 中：

- 优化器将子查询 `(SELECT name FROM t2)` 的结果视为与内部表 `t2` 中 `name` 列的 collation **没有直接关系**的表达式
- 外层查询在进行 `t1.name = <subquery_result>` 比较时，无法确定该用哪个 collation
- 两个 `IMPLICIT` 级别（都是来自列值，强制等级为 2）但 collation 不同的值碰在一起，MySQL 无法仲裁，于是报错

实际上，这是 MySQL 5.7 优化器的**派生表（derived table）元数据传播缺陷**。当子查询被转换为一个内部派生表时，派生表的列元数据中并没有完整保留原始列的字符集/collation 属性，导致外层查询"看不到"子查询列的真实 collation。

#### 2.3 深入一点：optimizer trace 的视角

如果打开 MySQL 5.7 的 optimizer trace，你会发现子查询被物化（materialized）时，其输出列的字符集信息是不完整或缺失的。外层查询在做 comparison 时，optimizer 拿到的两个操作数各自带有 `IMPLICIT` 级别的不同 collation，而它没有足够的上下文去判断谁"更对"——所以直接报错。

### 三、MySQL 8.0 的改进

#### 3.1 优化器对派生列元数据的增强

MySQL 8.0 对优化器进行了大幅重构，其中一个重要改进就是：**派生表（包括子查询物化出的派生表）现在能够完整地保留和传播其输出列的字符集/collation 元数据**。

这意味着：

- 当 `(SELECT name FROM t2)` 被物化时，8.0 的优化器知道"这一列的 collation 是 `utf8mb4_general_ci`，来自 `t2.name`"
- 外层查询在做 `t1.name = <subquery_result>` 比较时，能够清晰地看到两边的 collation
- 虽然 collation 不同，但优化器现在能够基于更多上下文信息正确地推导比较时使用的 collation（比如根据 SQL 语义、表的属性等）

简单来说：**MySQL 8.0 的优化器比 5.7"看得更远"——它能看透子查询，知道里面的列到底是啥字符集**。

#### 3.2 CTE（公共表表达式）同样受益

同样的改进也适用于 CTE（`WITH ... AS`）：

```sql
WITH cte AS (
  SELECT name FROM t2
)
SELECT * FROM t1 WHERE t1.name IN (SELECT name FROM cte);
```

在 MySQL 8.0 中，CTE 中的列元数据同样会被正确传播。而 MySQL 5.7 根本就不支持 CTE 语法——这也是升级的另一个理由。

#### 3.3 验证：同一条 SQL 在 8.0 上正常工作

```sql
-- MySQL 8.0 下执行
mysql> SELECT * FROM t1 WHERE t1.name IN (SELECT name FROM t2);
+----+------+
| id | name |
+----+------+
|  1 | Nic  |
+----+------+
1 row in set (0.00 sec)
```

不需要任何 `COLLATE` 子句，不需要 `CONVERT`，查询正常完成。

### 四、其他触发场景

除了基本的 `IN` 子查询，这个问题还会在以下场景中出现：

#### 4.1 UNION 查询

```sql
SELECT name FROM t1
UNION
SELECT name FROM t2;
```

两张表的 `name` 字段 collation 不同时，MySQL 5.7 在 UNION 的去重排序阶段可能报 `Illegal mix of collations`。MySQL 8.0 则能自动选择更"泛用"的 collation。

#### 4.2 派生表 JOIN

```sql
SELECT *
FROM t1
JOIN (SELECT name FROM t2) AS dt ON t1.name = dt.name;
```

本质上和子查询是一类问题，5.7 同样会报错。

#### 4.3 NOT IN 子查询

```sql
SELECT * FROM t1 WHERE t1.name NOT IN (SELECT name FROM t2);
```

`NOT IN` 比 `IN` 更复杂（涉及 NULL 处理），在 5.7 中报错的几率更高，而且错误信息可能更令人困惑。

#### 4.4 跨库查询

如果 `t1` 在 `db_a`，`t2` 在 `db_b`，且两个库的默认字符集不同，问题会更加隐蔽——因为建表时如果没有显式指定 charset，就会继承数据库级别的默认值，导致两个表不知不觉间就有了不同的 collation。

### 五、MySQL 5.7 的解决方案与变通方法

如果你的生产环境还在使用 MySQL 5.7，或者正在做迁移、需要兼容两边的 SQL，以下几种变通方案可以参考：

#### 5.1 使用 COLLATE 子句显式指定

在子查询的列上直接加 `COLLATE`：

```sql
SELECT *
FROM t1
WHERE t1.name IN (SELECT name COLLATE utf8mb4_unicode_ci FROM t2);
```

或者在外层查询的列上加：

```sql
SELECT *
FROM t1
WHERE t1.name COLLATE utf8mb4_general_ci IN (SELECT name FROM t2);
```

这是最轻量的改动，而且对索引使用影响较小（取决于 collation 方向）。

#### 5.2 使用 CONVERT() 函数

```sql
SELECT *
FROM t1
WHERE t1.name IN (SELECT CONVERT(name USING utf8mb4) COLLATE utf8mb4_unicode_ci FROM t2);
```

`CONVERT` 会先转换字符集，再加 `COLLATE` 指定校对规则。但注意：对列使用函数会导致 MySQL 无法使用该列上的索引，可能引发性能问题。

#### 5.3 改写为 JOIN

很多时候，子查询可以等价改写为 JOIN：

```sql
-- 原查询
SELECT * FROM t1 WHERE t1.name IN (SELECT name FROM t2);

-- 改写为 JOIN（去重语义下可能需要 SELECT DISTINCT）
SELECT DISTINCT t1.*
FROM t1
JOIN t2 ON t1.name = t2.name;
```

不过需要注意：`IN` 子查询和 `JOIN` 的语义不完全相同——`IN` 有隐式的去重效果，而 `JOIN` 会产生笛卡尔积的重复行。如果需要保持精确语义，可以用 `EXISTS` 替代。

#### 5.4 统一表的字符集（推荐）

最彻底的解决方案：统一所有相关表的 charset 和 collation：

```sql
ALTER TABLE t2 MODIFY name VARCHAR(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

**这是长期来看最好的做法**——从根本上消除 collation 不一致，不需要在每条 SQL 里加 `COLLATE`。

#### 方案对比

| 方案 | 对 SQL 的侵入性 | 性能影响 | 长期可维护性 |
|------|:--:|:--:|:--:|
| COLLATE 子句 | 中（每条 SQL 都要加） | 低 | 差 |
| CONVERT() | 中 | 高（可能无法用索引） | 差 |
| 改写为 JOIN | 高（需逐条改写） | 取决于改写质量 | 中 |
| 统一字符集 | 一次性 | 无 | 优 |

### 六、迁移到 MySQL 8.0 的注意事项

#### 6.1 默认字符集变了

MySQL 8.0 将默认字符集从 `latin1` 改为了 `utf8mb4`，默认 collation 从 `latin1_swedish_ci` 改为了 `utf8mb4_0900_ai_ci`。

这意味着：

- **新建的表**：如果没有显式指定 charset，会自动使用 `utf8mb4_0900_ai_ci`
- **从 5.7 迁移过来的老表**：仍然保持原来的 charset/collation
- **新老表混合查询时**：新默认值和老表之间的 collation 差异会引入新的 "Illegal mix of collations" 风险

**建议**：建表时始终显式指定 `CHARACTER SET` 和 `COLLATE`，不要依赖默认值。

#### 6.2 `utf8mb4_0900_ai_ci` vs `utf8mb4_general_ci` 的区别

MySQL 8.0 的默认 collation `utf8mb4_0900_ai_ci` 基于 Unicode 9.0 标准，是 accent-insensitive（不区分重音）的。这与 5.7 常用的 `utf8mb4_general_ci` 在排序和比较行为上存在细微差异。在迁移前，建议检查业务逻辑是否依赖特定的排序行为。

#### 6.3 迁移检查清单

1. **审查所有表的字符集**：`SELECT table_name, table_collation FROM information_schema.tables WHERE table_schema = 'your_db';`
2. **找出 collation 不一致的表对**：检查哪些表之间存在 collation 差异，这些表之间如果有 JOIN/子查询就可能出问题
3. **在 8.0 环境中全量回归测试**：把所有涉及字符串比较的 SQL 在 8.0 上跑一遍
4. **确认 `collation_server` 配置**：如果不想用 8.0 的默认 collation，在 `my.cnf` 中显式设置为你期望的值
5. **关注 `sql_mode` 的变化**：8.0 默认 `sql_mode` 也变了，可能影响其他行为

### 七、总结

- MySQL 5.7 的优化器在子查询、派生表等场景下，无法正确地将列的字符集/collation 元数据传播到外层查询，导致 `Illegal mix of collations` 错误
- MySQL 8.0 优化器在这方面做了大幅改进，能够正确识别子查询结果列的字符集，使很多在 5.7 上需要手动 `COLLATE` 的 SQL 可以直接执行
- 短期变通方案：使用 `COLLATE` 子句、`CONVERT()` 函数，或改写为 `JOIN`
- 长期最佳实践：统一所有表的 charset 和 collation，建表时始终显式指定，不依赖默认值
- 迁移到 8.0 前，务必在测试环境全面验证涉及字符串比较的 SQL

> 参考：
>
> [MySQL 8.0 Reference Manual: Collation Coercibility](https://dev.mysql.com/doc/refman/8.0/en/charset-collation-coercibility.html)
>
> [MySQL 8.0 Release Notes: Optimizer Improvements](https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-0.html)
>
> [MySQL 8.0 Reference Manual: The utf8mb4 Character Set](https://dev.mysql.com/doc/refman/8.0/en/charset-unicode-utf8mb4.html)
>
> [MySQL 5.7 Reference Manual: Collation Issues](https://dev.mysql.com/doc/refman/5.7/en/charset-collation-issues.html)
