## 创建数据库

CockroachDB 带着一个单一、默认的系统数据库，其中包含着 CockroachDB 的元数据并且该数据库还是只读的。若要新建一个数据库，可以使用 [`CREATE DATABASE`](create-database.html) 后接数据库的名字，例如：

```sql
> CREATE DATABASE bank;
```

数据库的名字必须遵守这些 [命名规范](keywords-and-identifiers.html#identifiers)。为了避免数据库已经存在的情况，可以加上 `IF NOT EXISTS`，例如：

```sql
> CREATE DATABASE IF NOT EXISTS bank;
```

当你不再需要一个数据库时，可使用 `DROP DATABASE` 后接数据库的名字，该操作将会移除该数据库以及他的所有对象，例如：

```sql
> DROP DATABASE bank;
```

## 展示数据库

使用 `SHOW DATABASES` 语句可查看所有数据库：

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

使用 `SET` 语句可设置默认数据库，例如：

```sql
> SET DATABASE = bank;
```

当你正在使用默认数据库的时候，使用的时候就不用在语句中明确地提及。若想查看当前使用的默认数据库，可使用 `SHOW DATABASE` 语句（注意是单数形式）：

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

使用 `CREATE TABLE` 后接表名，列名、数据类型及约束，可新建一个表，例如：

```sql
> CREATE TABLE accounts (
    id INT PRIMARY KEY,
    balance DECIMAL
);
```

表名和列名必须遵守这些 [命名规范](keywords-and-identifiers.html#identifiers)。而且，当你没有明确地定义一个主键时，CockroachDB 会自动地添加一个隐藏的 `rowid` 列作为主键。

为了避免表已经存在的情况，可以加上 `IF NOT EXISTS`，例如：

```sql
> CREATE TABLE IF NOT EXISTS accounts (
    id INT PRIMARY KEY,
    balance DECIMAL
);
```

使用 [`SHOW COLUMNS FROM`](show-columns.html) 后接表名可查看表中的所有列，例如：

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

当你不再需要一个表时，使用  [`DROP TABLE`](drop-table.html) 后接表名可移除该表以及他的所有数据，例如：

```sql
> DROP TABLE accounts;
```

## 展示表

使用  [`SHOW TABLES`](show-tables.html) 语句可查看当前正在使用的数据库中的所有表：

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

使用 `SHOW TABLES FROM` 后接数据库的名字，可以查看当前未使用的数据库的表，例如：

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

## 在表中插入行

使用 [`INSERT INTO`](insert.html) 后接表名，以及按表中列名顺序排列的所要插入的值，可在表中插入一行数据，例如：

```sql
> INSERT INTO accounts VALUES (1, 10000.50);
```

如果你想通过不同的顺序插入所对应的列名的值，就需要明确地指出列名顺序以及按此顺序排列的值，例如：

```sql
> INSERT INTO accounts (balance, id) VALUES
    (25000.00, 2);
```

若想要在一个表中插入多行的数据，可使用一个列表，以逗号分隔开，每一项都是按列名顺序排列的值，用括号封闭起来，例如：

```sql
> INSERT INTO accounts VALUES
    (3, 8100.73),
    (4, 9400.10);
```

当你跳过列名所对应的值或者明确要求使用默认值时，该值将使用 [默认值](default-value.html)。比如，下面两个语句将插入一行 列 `balance` 的值为默认值的数据，在这里 `balance` 的默认值为 `NULL`：

```sql
> INSERT INTO accounts (id, balance) VALUES
    (5);
```

```sql
> INSERT INTO accounts (id, balance) VALUES
    (6, DEFAULT);
```

当我们查看时：

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

## 创建一个索引

索引可以帮助我们定位数据，而不需要去查看表中的每一行数据。这些索引会根据表中的主键以及任意带有唯一约束的列而自动创建。

为了给那些未带有唯一约束的列创建索引，我们可以使用 [`CREATE INDEX`](create-index.html) 后接索引名(可选)和一个标识表及列索引的 `ON` 子句。对于每一列，你可以选择升序排列还是降序排列，例如：

```sql
> CREATE INDEX balance_idx ON accounts (balance DESC);
```

你也可以在创建表的同时创建索引，只需要加上 `INDEX` 关键字后接索引名(可选)及索引所对应的列名，例如：

```sql
> CREATE TABLE accounts (
    id INT PRIMARY KEY,
    balance DECIMAL,
    INDEX balance_idx (balance)
);
```

##展示表的索引

使用 [`SHOW INDEX FROM`](show-index.html) 后接表名，可展示表的索引，例如：

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

使用 [`SELECT`](select.html) 后接一个列表，每一项都是列名，用逗号分隔开，就可以检索我们想要查询的列，例如：

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

使用 `*` 通配符可以检索所有列，例如：

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

使用 `WHERE` 子句标识列及用来过滤的对应的值，可过滤查询到的结果。

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

使用 `ORDER BY` 子句标识想要排序的列，可对查询到的结果进行排序。对每一列，你可以选择是升序排列还是降序排列，例如：

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

## 更新表中行数据

使用 [`UPDATE`](update.html) 后接表名，和一个 `SET` 子句标识想要更新的列以及对应的新的值，还有一个 `WHERE` 子句标识所要更新的行，可更新表中行的数据，例如：

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

如果表中含有主键，`UPDATE` 配合 `WHERE` 子句便可可靠地更新特定的行；否则，满足 `WHERE` 子句的每一行的数据都将会更新。当未使用 `WHERE` 子句时，表中的所有行的数据都将会更新。

## 删除表中的行

使用 [`DELETE FROM`](delete.html) 后接表名以及 `WHERE` 子句标识所要删除的行，可删除表中的行，例如：

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

正如 `UPDATE` 语句一样，如果表中含有主键，`UPDATE` 配合 `WHERE` 子句便可可靠地删除特定的行；否则，满足 `WHERE` 子句的每一行的数据都将会被删除。当未使用 `WHERE` 子句时，表中的所有行的数据都将会被删除。

## 下一节

-   探索所有的 [SQL 语句](sql-statements.html)
-   [使用内置的数据库客户端](use-the-built-in-sql-client.html) 可以使我们从一个脚本或者直接从命令行执行语句
-   为你最喜欢的语言 [安装客户端驱动程序](install-client-drivers.html) 然后 [构建一个APP](build-an-app-with-cockroachdb.html)
-   [探索 CockroachDB 核心功能](demo-data-replication.html)，如：自动复制，再平衡以及容错