# 创建表

`CREATE TABLE` [语句](https://www.cockroachlabs.com/docs/stable/sql-statements.html)用来在数据库中创建新表。

## 权限要求

必须拥有对数据库的 `CREATE` [权限](https://www.cockroachlabs.com/docs/stable/privileges.html)。

## 简介

## 基本版

![](https://github.com/TechCatsLab/cockroachdb_docs_cn/blob/master/images/create-table-synopsis-basic.png)

## 扩展版

![](https://github.com/TechCatsLab/cockroachdb_docs_cn/blob/master/images/create-table-synopsis-expanded.png)

> **提示：**若要根据 `SELECT` 语句的结果来创建表，请查看 [`CREATE TABLE AS`](https://www.cockroachlabs.com/docs/stable/create-table-as.html)。

## 参数

| 参数                 | 描述                                       |
| ------------------ | ---------------------------------------- |
| `IF NOT EXISTS`    | 只当数据库中不存在同名表的情况下才创建表，如果该表已存在也不会返回错误。<br><br>注意 `IF NOT EXISTS`  语句只检查表的名字；不检查现有表是否具有新表的相同列、索引、约束等。 |
| `any_name`         | 创建的表的名字，该名字必须在数据库中是唯一的且符合[标识符规则](https://www.cockroachlabs.com/docs/stable/keywords-and-identifiers.html#identifiers)。当没有选中数据库时，必须用 `database.name` 来指明表名。<br><br>[`UPSERT`](https://www.cockroachlabs.com/docs/stable/upsert.html) 和 [`INSERT ON CONFLICT`](https://www.cockroachlabs.com/docs/stable/insert.html) 语句使用叫作 `excluded` 的临时表来处理过程中的唯一性冲突。 因此，不建议对任何表使用已排除的名称。 |
| `column_def`       | 使用逗号分隔的列定义列表。每一列都必须指定 [名字/标识符](https://www.cockroachlabs.com/docs/stable/keywords-and-identifiers.html#identifiers) 和 [数据类型](https://www.cockroachlabs.com/docs/stable/data-types.html)；可以选择性地指定列级约束。在表中列名必须是唯一的但是可以使用相同的名字作为索引和限制。<br><br>在列级别定义的任何主键、唯一和检查约束都被移动到表级别，作为表创建的一部分。使用 [`SHOW CREATE TABLE`](show-create-table.html) 语句可以在表级别显示它们。 |
| `index_def`        | 一个可选的，逗号分隔的[索引定义](https://www.cockroachlabs.com/docs/stable/indexes.html)列表。 对于每个索引，必须指定索引的列；或者可选地指定名称。 索引名字必须表内唯一且符合 [标识符规则](https://www.cockroachlabs.com/docs/stable/keywords-and-identifiers.html#identifiers)。查看[创建具有次级索引的表](https://www.cockroachlabs.com/docs/stable/create-table.html#create-a-table-with-secondary-indexes)了解更多。<br><br>[`CREATE INDEX`](https://www.cockroachlabs.com/docs/stable/create-index.html) 语句可用于创建与表创建分开的索引。 |
| `family_def`       | 一个可选的、以逗号分隔的[列表列族的定义](https://www.cockroachlabs.com/docs/stable/column-families.html)。列名称在表中必须是唯一的，但可以与列、约束或索引具有相同的名称。<br><br>列族是在底层键值存储中作为单个键值对存储的一组列。cockroachdb自动组列为有保障和高效的存储。但是，有些情况下可能希望手动将列分配给族。有关详细信息，请参见[列族](https://www.cockroachlabs.com/docs/stable/column-families.html)。 |
| `table_constraint` | 一个可选的、逗号分隔的[表级约束列表](https://www.cockroachlabs.com/docs/stable/constraints.html)。约束名称在表中必须是唯一的，但可以具有与列、列族或索引相同的名称。 |
| `opt_interleave`   | 可以通过[交叉表](https://www.cockroachlabs.com/docs/stable/interleave-in-parent.html)查询进行性能优化，从而改变cockroachdb存储数据的方式。 |

## 表级复制

默认情况下，表是在默认复制区中创建的，但可以放在特定的复制区中。请参阅对于一个表复制区的更多[创建信息](https://www.cockroachlabs.com/docs/stable/configure-replication-zones.html#create-a-replication-zone-for-a-table)。

## 例子

### 创建一个表 (默认没有主键)

在CockroachDB，每一个表都需要主键。如果没有明确的定义，一个叫做 `rowid`，类型为 `int` 的列将被自动添加为主键，以确保新的行总是具有独特的 `rowid` 值可执行 `unique_rowid()` 功能。主键自动索引。

> **注意：**严格来说，主键的唯一索引其实不会被创建；它是根据存储数据的键派生的，因此不占用额外的空间。但是，使用显示索引之类的命令时，它似乎是一个正常的唯一索引。

~~~ sql
> CREATE TABLE logon (
    user_id INT,
    logon_date DATE
);

> SHOW COLUMNS FROM logon;
~~~

~~~
+------------+------+------+---------+---------+
|   Field    | Type | Null | Default | Indices |
+------------+------+------+---------+---------+
| user_id    | INT  | true | NULL    | {}      |
| logon_date | DATE | true | NULL    | {}      |
+------------+------+------+---------+---------+
(2 rows)
~~~

~~~ sql
> SHOW INDEX FROM logon;
~~~

~~~
+-------+---------+--------+-----+--------+-----------+---------+----------+
| Table |  Name   | Unique | Seq | Column | Direction | Storing | Implicit |
+-------+---------+--------+-----+--------+-----------+---------+----------+
| logon | primary | true   |   1 | rowid  | ASC       | false   | false    |
+-------+---------+--------+-----+--------+-----------+---------+----------+
(1 row)
~~~

### 创建一个表 (定义主键)

在这个例子中，我们创建了一个有三列的表。一个列是[主键](https://www.cockroachlabs.com/docs/stable/primary-key.html)，另一列是[唯一约束](https://www.cockroachlabs.com/docs/stable/unique.html)，而第三列则没有约束。具有唯一约束的主键和列将自动索引。

~~~ sql
> CREATE TABLE logoff (
    user_id INT PRIMARY KEY,
    user_email STRING UNIQUE,
    logoff_date DATE
);

> SHOW COLUMNS FROM logoff;
~~~

~~~
+-------------+--------+-------+---------+---------------------------------+
|    Field    |  Type  | Null  | Default |             Indices             |
+-------------+--------+-------+---------+---------------------------------+
| user_id     | INT    | false | NULL    | {primary,logoff_user_email_key} |
| user_email  | STRING | true  | NULL    | {logoff_user_email_key}         |
| logoff_date | DATE   | true  | NULL    | {}                              |
+-------------+--------+-------+---------+---------------------------------+
(3 rows)
~~~

~~~ sql
> SHOW INDEX FROM logoff;
~~~

~~~
+--------+-----------------------+--------+-----+------------+-----------+---------+----------+
| Table  |         Name          | Unique | Seq |   Column   | Direction | Storing | Implicit |
+--------+-----------------------+--------+-----+------------+-----------+---------+----------+
| logoff | primary               | true   |   1 | user_id    | ASC       | false   | false    |
| logoff | logoff_user_email_key | true   |   1 | user_email | ASC       | false   | false    |
| logoff | logoff_user_email_key | true   |   2 | user_id    | ASC       | false   | true     |
+--------+-----------------------+--------+-----+------------+-----------+---------+----------+
(3 rows)
~~~

### 创建具有次级索引的表

在本例中，我们在表创建期间创建了两个辅助索引。辅助索引允许使用主键以外的键对数据进行有效访问。这个例子还演示了一些列级和表级[约束](https://www.cockroachlabs.com/docs/stable/constraints.html)。

~~~ sql
> CREATE TABLE product_information (
    product_id           INT PRIMARY KEY NOT NULL,
    product_name         STRING(50) UNIQUE NOT NULL,
    product_description  STRING(2000),
    category_id          STRING(1) NOT NULL CHECK (category_id IN ('A','B','C')),
    weight_class         INT,
    warranty_period      INT CONSTRAINT valid_warranty CHECK (warranty_period BETWEEN 0 AND 24),
    supplier_id          INT,
    product_status       STRING(20),
    list_price           DECIMAL(8,2),
    min_price            DECIMAL(8,2),
    catalog_url          STRING(50) UNIQUE,
    date_added           DATE DEFAULT CURRENT_DATE(),
    CONSTRAINT price_check CHECK (list_price >= min_price),
    INDEX date_added_idx (date_added),
    INDEX supp_id_prod_status_idx (supplier_id, product_status)
);

> SHOW INDEX FROM product_information;
~~~

~~~
+---------------------+--------------------------------------+--------+-----+----------------+-----------+---------+----------+
|        Table        |                 Name                 | Unique | Seq |     Column     | Direction | Storing | Implicit |
+---------------------+--------------------------------------+--------+-----+----------------+-----------+---------+----------+
| product_information | primary                              | true   |   1 | product_id     | ASC       | false   | false    |
| product_information | product_information_product_name_key | true   |   1 | product_name   | ASC       | false   | false    |
| product_information | product_information_product_name_key | true   |   2 | product_id     | ASC       | false   | true     |
| product_information | product_information_catalog_url_key  | true   |   1 | catalog_url    | ASC       | false   | false    |
| product_information | product_information_catalog_url_key  | true   |   2 | product_id     | ASC       | false   | true     |
| product_information | date_added_idx                       | false  |   1 | date_added     | ASC       | false   | false    |
| product_information | date_added_idx                       | false  |   2 | product_id     | ASC       | false   | true     |
| product_information | supp_id_prod_status_idx              | false  |   1 | supplier_id    | ASC       | false   | false    |
| product_information | supp_id_prod_status_idx              | false  |   2 | product_status | ASC       | false   | false    |
| product_information | supp_id_prod_status_idx              | false  |   3 | product_id     | ASC       | false   | true     |
+---------------------+--------------------------------------+--------+-----+----------------+-----------+---------+----------+
(10 rows)
~~~

我们还有索引的其它资源：

- 使用 [CREATE INDEX](https://www.cockroachlabs.com/docs/stable/create-index.html) 对已存在的表创建索引。
- [查看更多索引相关内容](https://www.cockroachlabs.com/docs/stable/indexes.html).

### 创建一个表，自动生成唯一的行标识

自动生成唯一的行标识，使用可执行 `unique_rowid()` 功能的默认值的 `INT` 类型 `SERIAL` 类型。

```sql
> CREATE TABLE test (id SERIAL PRIMARY KEY, name STRING);
```

在插入操作中，`unique_rowid()` 函数会以执行插入功能的时间戳和 ID 生成一个默认的值，除非一种全局唯一的组合，每个节点每秒生成大量IDS（100000 +）的极端情况。在这种情况下，你应该使用一个字节的列与uuid_v4()作为默认值代替：

```sql
> CREATE TABLE test (id BYTES PRIMARY KEY DEFAULT uuid_v4(), name STRING);
```

因为字节值是128位，比64位的 int 值大得多，所以几乎没有机会生成非唯一值。

IDs 在关键值级别的分布也可能是一个考虑因素。当使用字节 uuid_v4() 作为默认值，生成连续的IDS将分布在不同的关键值范围（因此可能在不同的节点），而使用 int unique_rowid() 作为默认值时，连续生成的ID可以结束在同一关键值范围。

### 创建一个具有外键的表

[外键](https://www.cockroachlabs.com/docs/stable/foreign-key.html)保证一个列只使用它引用的列中已经存在的值，该值必须来自另一个表。此约束强制在两个表之间引用完整性。

有很多管理外键的规则，但以下两点是最重要的：

- 当使用`索引`、`主键`或`唯一值`创建表时，必须对外键列进行索引。
- 引用的列必须包含唯一值。这意味着引用子句必须使用与[主键](https://www.cockroachlabs.com/docs/stable/primary-key.html)或[唯一约束](https://www.cockroachlabs.com/docs/stable/unique.html)完全相同的列。

在本例中，我们将使用不同格式的外键显示一系列表。

~~~ sql
> CREATE TABLE customers (id INT PRIMARY KEY, email STRING UNIQUE);

> CREATE TABLE products (sku STRING PRIMARY KEY, price DECIMAL(9,2));

> CREATE TABLE orders (
  id INT PRIMARY KEY,
  product STRING NOT NULL REFERENCES products,
  quantity INT,
  customer INT NOT NULL CONSTRAINT valid_customer REFERENCES customers (id),
  CONSTRAINT id_customer_unique UNIQUE (id, customer),
  INDEX (product),
  INDEX (customer)
);

> CREATE TABLE reviews (
  id INT PRIMARY KEY,
  product STRING NOT NULL REFERENCES products,
  customer INT NOT NULL,
  "order" INT NOT NULL,
  body STRING,
  CONSTRAINT order_customer_fk FOREIGN KEY ("order", customer) REFERENCES orders (id, customer),
  INDEX (product),
  INDEX (customer),
  INDEX ("order", customer)
);
~~~

### 创建 key-value 存储的表

cockroachdb 是分布式 SQL 数据库并建立在强业务和强兼容性的键值对存储系统。虽然无法直接访问密钥值存储区，但可以使用两列的 “simple” 镜像来直接访问，其中一组作为主键：

```sql
> CREATE TABLE kv (k INT PRIMARY KEY, v BYTES);
```

当这样一个 “simple” 表没有索引或外键， [`INSERT`](https://www.cockroachlabs.com/docs/stable/insert.html)/[`UPSERT`](https://www.cockroachlabs.com/docs/stable/upsert.html)/[`UPDATE`](https://www.cockroachlabs.com/docs/stable/update.html)/[`DELETE`](https://www.cockroachlabs.com/docs/stable/delete.html) 语句以最小的开销翻译关键值操作（个位数的百分比下降）。例如，以下的`UPSERT` 将插入或更换行将转化成一个单一的 键-值对操作：

```sql
> UPSERT INTO kv VALUES (1, b'hello')
```

这个SQL表方法还提供了一个定义良好的查询语言、一个已知的事务模型，以及在需要时向表中添加更多列的灵活性。

### 使用 `SELECT` 语句创建表

你可以使用 [`CREATE TABLE AS`](https://www.cockroachlabs.com/docs/stable/create-table-as.html) 语句从 `SELECT` 语句的执行结果来创建一个新表，例如：

~~~ sql
> SELECT * FROM customers WHERE state = 'NY';
~~~
~~~
+----+---------+-------+
| id |  name   | state |
+----+---------+-------+
|  6 | Dorotea | NY    |
| 15 | Thales  | NY    |
+----+---------+-------+
~~~
~~~ sql
> CREATE TABLE customers_ny AS SELECT * FROM customers WHERE state = 'NY';

> SELECT * FROM customers_ny;
~~~
~~~
+----+---------+-------+
| id |  name   | state |
+----+---------+-------+
|  6 | Dorotea | NY    |
| 15 | Thales  | NY    |
+----+---------+-------+
~~~

### 显示表的定义

若要显示表的定义，可以使用 [SHOW CREATE TABLE](https://www.cockroachlabs.com/docs/stable/show-create-table.html) 语句。响应 `CreateTable` 列的内容是一个嵌入了换行符的字符串，并产生格式化输出。

~~~ sql
> SHOW CREATE TABLE logoff;
~~~

~~~
+--------+----------------------------------------------------------+
| Table  |                       CreateTable                        |
+--------+----------------------------------------------------------+
| logoff | CREATE TABLE logoff (
                                   |
|        |     user_id INT NOT NULL,
                               |
|        |     user_email STRING(50) NULL,
                         |
|        |     logoff_date DATE NULL,
                              |
|        |     CONSTRAINT "primary" PRIMARY KEY (user_id),
         |
|        |     UNIQUE INDEX logoff_user_email_key (user_email),
    |
|        |     FAMILY "primary" (user_id, user_email, logoff_date)
 |
|        | )                                                        |
+--------+----------------------------------------------------------+
(1 row)
~~~

## 查看更多

- [`INSERT`](insert.html)
- [`ALTER TABLE`](alter-table.html)
- [`DELETE`](delete.html)
- [`DROP TABLE`](drop-table.html)
- [`RENAME TABLE`](rename-table.html)
- [`SHOW TABLES`](show-tables.html)
- [`SHOW COLUMNS`](show-columns.html)
- [列族](https://www.cockroachlabs.com/docs/stable/column-families.html)
- [表级复制区](https://www.cockroachlabs.com/docs/stable/configure-replication-zones.html#create-a-replication-zone-for-a-table)
