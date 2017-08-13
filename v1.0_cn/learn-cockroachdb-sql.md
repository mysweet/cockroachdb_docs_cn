# SQL 语句

本节将会向你介绍一部分最必不可少的 CockroachDB SQL 语句。完整的 SQL 语句列表以及相关细节，见 [SQL 语句](sql-statements.md)。

> **注意**<br>CockroachDB 旨在提供标准的 SQL 及扩展，但是一些标准的 SQL 尚不可用。详情见 [SQL 特性支持](sql-feature-support.md)。

## 创建数据库

CockroachDB 带着一个单一、默认的 `system` 数据库，其中包含着 CockroachDB 的元数据并且是只读的。若要新建一个数据库，可以使用 [`CREATE DATABASE`](create-database.md) 后接数据库的名字，例如：

```sql
> CREATE DATABASE bank;
```

数据库的名字必须遵守[这些命名规范](keywords-and-identifiers.md#identifiers)。为了避免数据库已经存在的情况，可以加上 `IF NOT EXISTS`，例如：

```sql
> CREATE DATABASE IF NOT EXISTS bank;
```

当你不再需要一个数据库时，可使用 [`DROP DATABASE`](drop-database.md) 后接数据库的名字，移除该数据库及其所有对象，例如：

```sql
> DROP DATABASE bank;
```

## 查看数据库

使用 [`SHOW DATABASES`](show-databases.md) 语句可以查看所有的数据库：

```sql
> SHOW DATABASES;
```

```
+--------------------+
|      Database      |
+--------------------+
| bank               |
| crdb_internal      |
| information_schema |
| pg_catalog         |
| system             |
+--------------------+
(5 rows)
```

## 设置默认数据库

使用 [`SET`](set-vars.md#examples) 语句可以设置默认数据库，例如：

```sql
> SET DATABASE = bank;
```

在使用默认数据库的时候，不需要在语句中明确地提及。若想查看当前使用的默认数据库，可使用 `SHOW DATABASE` 语句（注意是单数形式）：

```sql
> SHOW DATABASE;
```

```
+----------+
| database |
+----------+
| bank     |
+----------+
(1 row)
```

## 创建表

使用 [`CREATE TABLE`](create-table.md) 后接表名，列名、及每个列的[数据类型](data-types.md)和[约束](constraints.md) （如果有的话），新建一张表，例如：

```sql
> CREATE TABLE accounts (
    id INT PRIMARY KEY,
    balance DECIMAL
);
```

表名和列名必须遵守 [这些规范](keywords-and-identifiers.md#identifiers)。而且，当没有明确地定义一个主键时，CockroachDB 会自动地添加一个隐藏的 `rowid` 列作为主键。

为了避免表已经存在的情况，可以加上 `IF NOT EXISTS`，例如：

```sql
> CREATE TABLE IF NOT EXISTS accounts (
    id INT PRIMARY KEY,
    balance DECIMAL
);
```

使用 [`SHOW COLUMNS FROM`](show-columns.md) 后接表名可查看表中的所有列，例如：

```sql
> SHOW COLUMNS FROM accounts;
```

```
+---------+---------+-------+---------+-----------+
|  Field  |  Type   | Null  | Default |  Indices  |
+---------+---------+-------+---------+-----------+
| id      | INT     | false | NULL    | {primary} |
| balance | DECIMAL | true  | NULL    | {}        |
+---------+---------+-------+---------+-----------+
(2 rows)
```

当不再需要一张表时，使用  [`DROP TABLE`](drop-table.md) 后接表名可移除该表及其所有数据，例如：

```sql
> DROP TABLE accounts;
```

## 查看表

使用  [`SHOW TABLES`](show-tables.md) 语句可以查看当前正在使用的数据库中的所有表：

```sql
> SHOW TABLES;
```

```
+----------+
|  Table   |
+----------+
| accounts |
| users    |
+----------+
(2 rows)
```

使用 `SHOW TABLES FROM` 后接数据库的名字，可以查看不活跃数据库的表，例如：

```sql
> SHOW TABLES FROM animals;
```

```
+-----------+
|   Table   |
+-----------+
| aardvarks |
| elephants |
| frogs     |
| moles     |
| pandas    |
| turtles   |
+-----------+
(6 rows)
```

## 向表中插入行

使用 [`INSERT INTO`](insert.md) 后接表名，以及按表中列名顺序排列的所要插入的值，可以在表中插入一行数据，例如：

```sql
> INSERT INTO accounts VALUES (1, 10000.50);
```

如果你想以不同的顺序插入列的值，就需要明确地列出列的顺序并按相应顺序提供列的值，例如：

```sql
> INSERT INTO accounts (balance, id) VALUES
    (25000.00, 2);
```

若要向一张表中插入多行数据，可使用一个列表，以逗号分隔开，每一项都是按列名顺序排列的值，用括号封闭起来，例如：

```sql
> INSERT INTO accounts VALUES
    (3, 8100.73),
    (4, 9400.10);
```

当你跳过列名所对应的值或者明确要求使用默认值时，该值将使用 [默认值](default-value.md)。比如，下面两条语句都将插入一行，列 `balance` 的值为默认值，在这里为 `NULL`：

```sql
> INSERT INTO accounts (id, balance) VALUES
    (5);
```

```sql
> INSERT INTO accounts (id, balance) VALUES
    (6, DEFAULT);
```

```sql
> SELECT * FROM accounts WHERE id in (5, 6);
```

```
+----+---------+
| id | balance |
+----+---------+
|  5 | NULL    |
|  6 | NULL    |
+----+---------+
(2 rows)
```

## 创建索引

[索引](indexes.md)帮助定位数据，而不需要查看表中的每一行数据。索引会为表中的[主键](primary-key.md)以及任意带有[唯一约束](unique.md)的列自动创建。

使用 [`CREATE INDEX`](create-index.html) 后接可选的索引名和一个标明表及列索引的 `ON` 子句，为未带有唯一约束的列创建索引。对于每一列，可以选择升序（`ASC`）或者降序（`DESC`）排列。

```sql
> CREATE INDEX balance_idx ON accounts (balance DESC);
```

也可以在创建表的同时创建索引，只需要加上 `INDEX` 关键字后接可选的索引名及要索引的列名：

```sql
> CREATE TABLE accounts (
    id INT PRIMARY KEY,
    balance DECIMAL,
    INDEX balance_idx (balance)
);
```

##查看表的索引

使用 [`SHOW INDEX FROM`](show-index.md) 后接表名，可查看表的索引：

```sql
> SHOW INDEX FROM accounts;
```

```
+----------+-------------+--------+-----+---------+-----------+---------+----------+
|  Table   |    Name     | Unique | Seq | Column  | Direction | Storing | Implicit |
+----------+-------------+--------+-----+---------+-----------+---------+----------+
| accounts | primary     | true   |   1 | id      | ASC       | false   | false    |
| accounts | balance_idx | false  |   1 | balance | DESC      | false   | false    |
| accounts | balance_idx | false  |   2 | id      | ASC       | false   | true     |
+----------+-------------+--------+-----+---------+-----------+---------+----------+
(3 rows)
```

## 查询表

查询表，使用 [`SELECT`](select.md) 后接一个用逗号分隔的将返回的列名的列表，和要检索的表：

```sql
> SELECT balance FROM accounts;
```

```
+----------+
| balance  |
+----------+
| 10000.50 |
| 25000.00 |
|  8100.73 |
|  9400.10 |
| NULL     |
| NULL     |
+----------+
(6 rows)
```

使用 `*` 通配符可以检索所有列：

```sql
> SELECT * FROM accounts;
```

```
+----+----------+
| id | balance  |
+----+----------+
|  1 | 10000.50 |
|  2 | 25000.00 |
|  3 |  8100.73 |
|  4 |  9400.10 |
|  5 | NULL     |
|  6 | NULL     |
+----+----------+
(6 rows)
```

使用 `WHERE` 子句标明要过滤的列和值，可以过滤查询结果：

```sql
> SELECT id, balance FROM accounts WHERE balance > 9000;
```

```
+----+---------+
| id | balance |
+----+---------+
|  2 |   25000 |
|  1 | 10000.5 |
|  4 |  9400.1 |
+----+---------+
(3 rows)
```

使用 `ORDER BY` 子句标明要排序的列，可对查询结果排序。对于每一列，可以选择升序（`ASC`）或者降序（`DESC`）排列：

```sql
> SELECT id, balance FROM accounts ORDER BY balance DESC;
```

```
+----+---------+
| id | balance |
+----+---------+
|  2 |   25000 |
|  1 | 10000.5 |
|  4 |  9400.1 |
|  3 | 8100.73 |
|  5 | NULL    |
|  6 | NULL    |
+----+---------+
(6 rows)
```

## 更新表中的行

使用 [`UPDATE`](update.md) 后接表名，和一个 `SET` 子句标明要更新的列和新值，以及 `WHERE` 子句标明要更新的行，可以更新表中的行：

```sql
> UPDATE accounts SET balance = balance - 5.50 WHERE balance < 10000;
```

```sql
> SELECT * FROM accounts;
```

```
+----+----------+
| id | balance  |
+----+----------+
|  1 | 10000.50 |
|  2 | 25000.00 |
|  3 |  8095.23 |
|  4 |  9394.60 |
|  5 | NULL     |
|  6 | NULL     |
+----+----------+
(6 rows)
```

如果表有主键，可以在 `WHERE` 子句中使用主键可靠地更新特定的行；否则，匹配 `WHERE` 子句的每一行都将被更新。如果没有 `WHERE` 子句，表中的所有行都将被更新。

## 删除表中的行

使用 [`DELETE FROM`](delete.md) 后接表名以及 `WHERE` 子句标明所要删除的行，可以删除表中的行：

```sql
> DELETE FROM accounts WHERE id in (5, 6);
```

```sql
> SELECT * FROM accounts;
```

```
+----+----------+
| id | balance  |
+----+----------+
|  1 | 10000.50 |
|  2 | 25000.00 |
|  3 |  8095.23 |
|  4 |  9394.60 |
+----+----------+
(4 rows)
```

正如 `UPDATE` 语句一样，如果表有主键，在 `WHERE` 子句中使用主键可以可靠地删除特定的行；否则，满足 `WHERE` 子句的每一行都将被删除。如果没有 `WHERE` 子句，表中所有行都将被删除。

## 下一步

-   探索所有的 [SQL 语句](sql-statements.md)
-   [使用内建的数据库客户端](use-the-built-in-sql-client.md) 可以使我们在脚本中或者直接在命令行执行语句
-   为你最喜欢的语言 [安装客户端驱动程序](install-client-drivers.md) 并 [构建一个App](build-an-app-with-cockroachdb.md)
-   [探索 CockroachDB 核心功能](demo-data-replication.md)，如：自动复制，再平衡以及容错