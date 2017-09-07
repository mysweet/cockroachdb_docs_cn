# 创建索引

`CREATE INDEX` [语句](https://www.cockroachlabs.com/docs/stable/sql-statements.html)用来给表创建索引。 [索引](https://www.cockroachlabs.com/docs/stable/indexes.html) 通过帮助 SQL 查找数据而不必查看表的每一行，从而提高数据库性能。

> **注意：**对于表的 [`PRIMARY KEY`](https://www.cockroachlabs.com/docs/stable/primary-key.html) 和 [`UNIQUE`](https://www.cockroachlabs.com/docs/stable/unique.html) 字段，索引是自动创建的。
>
> 当查找一个表时，CockroachDB 使用最快的索引。查看更多，请看 [`CockroachDB 中的索引选择`](https://www.cockroachlabs.com/blog/index-selection-cockroachdb-2/)。

## 全选要求

用户必须拥有表的 `CREATE` 权限。

## 简介

![](https://github.com/TechCatsLab/cockroachdb_docs_cn/blob/master/images/create-index-synopsis.png)

## 参数

| 参数              | 描述                                       |
| --------------- | ---------------------------------------- |
| `UNIQUE`        | 将[唯一约束](https://www.cockroachlabs.com/docs/stable/unique.html)应用到索引列中。<br><br>这将导致系统检查索引创建时存在的重复值。它还应用表级上的唯一约束，因此系统在插入或更新数据时检查重复的值。 |
| `IF NOT EXISTS` | 仅当尚未存在同名索引时才创建新索引；如果确实存在，也不返回错误。         |
| `index_name`    | 创建的索引的名字，在表中[必须是唯一的](https://www.cockroachlabs.com/docs/stable/create-database.html#create-fails-name-already-in-use)且符合[标识符规则](https://www.cockroachlabs.com/docs/stable/keywords-and-identifiers.html#identifiers)。<br><br>如果不具体说明名字， CockroachDB 将使用 `<table>_<columns>_key/idx`. `key` 格式来创建索引；`idx` 表示它没有。例如: `accounts_balance_idx` |
| `table_name`    | 你想创建索引的标的 [`限定名称`](sql-grammar.html#qualified_name) 。 |
| `column_name`   | 你想索引的列名。                                 |
| `ASC` or `DESC` | 升序排序 (`ASC`) 或降序排序 (`DESC`) 的索引顺序。列的排序方式会影响查询结果，特别是在使用限制时。<br><br>__默认值:__ `ASC` |
| `STORING ...`   | 存储 (但不排序) 你包含名字的每一列。<br><br>查看何时使用 `STORING`, 请看 [存储列](https://www.cockroachlabs.com/docs/stable/create-index.html#store-columns).<br><br>别名 `COVERING` 和 `STORING` 是相同的。 |

## 例子

### 创建索引

为了创建最有效的索引，我们建议进行评审：

- [索引: 最佳实践](https://www.cockroachlabs.com/docs/stable/indexes.html#best-practices)
- [CockroachDB 中的索引选择](https://www.cockroachlabs.com/blog/index-selection-cockroachdb-2)

#### 单列索引

单个列索引对单个列的值进行排序。

~~~ sql
> CREATE INDEX ON products (price);
~~~

因为每次查询只能使用一个索引，所以单列索引通常不像多个列索引那样有用。

#### 多列索引

多个列索引按列的顺序对列进行排序。

~~~ sql
> CREATE INDEX ON products (price, stock);
~~~

为了创建最有用的多列索引，我们建议回顾我们的[最佳实践](https://www.cockroachlabs.com/docs/stable/indexes.html#best-practices)。

#### 唯一索引

唯一索引不允许列中的重复值。

~~~ sql
> CREATE UNIQUE INDEX ON products (name, manufacturer_id);
~~~

这也适用于表级的唯一约束，类似于更改表。上面的例子相当于:

~~~ sql
> ALTER TABLE products ADD CONSTRAINT products_name_manufacturer_id_key UNIQUE (name, manufacturer_id);
~~~

### 存储列

存储一个列可以提高检索（但不过滤）其值的查询的性能。

~~~ sql
> CREATE INDEX ON products (price) STORING (name);
~~~

但是，要使用存储的列，查询必须过滤同一索引中的另一列。例如，SQL只能在查询的 `WHERE` 子句过滤`价格`时索引检索名称值。

### 更改列排序顺序

要对列进行降序排序，必须在创建索引时显式地设置选项。（默认值是升序）

~~~ sql
> CREATE INDEX ON products (price DESC, stock);
~~~

列如何排序会影响查询使用索引返回的行的顺序，这会影响使用限制的查询。

### 查询特定索引

通常情况下，CockroachDB选择的指标为计算出最少的行扫描。但是，您可以重写该选项，并指定要使用的索引的名称。找到名字， use [`SHOW INDEX`](show-index.html) 语句。

~~~ sql
> SHOW INDEX FROM products;
~~~
~~~
+----------+--------------------+--------+-----+--------+-----------+---------+----------+
|  Table   |        Name        | Unique | Seq | Column | Direction | Storing | Implicit |
+----------+--------------------+--------+-----+--------+-----------+---------+----------+
| products | primary            | true   |   1 | id     | ASC       | false   | false    |
| products | products_price_idx | false  |   1 | price  | ASC       | false   | false    |
| products | products_price_idx | false  |   2 | id     | ASC       | false   | true     |
+----------+--------------------+--------+-----+--------+-----------+---------+----------+
(3 rows)
~~~
~~~ sql
> SELECT name FROM products@products_price_idx WHERE price > 10;
~~~

## 查看更多

- [索引](https://www.cockroachlabs.com/docs/stable/indexes.html)
- [`SHOW INDEX`](https://www.cockroachlabs.com/docs/stable/show-index.html)
- [`DROP INDEX`](https://www.cockroachlabs.com/docs/stable/drop-index.html)
- [`RENAME INDEX`](https://www.cockroachlabs.com/docs/stable/rename-index.html)
- [其他 SQL 语句](https://www.cockroachlabs.com/docs/stable/sql-statements.html)
