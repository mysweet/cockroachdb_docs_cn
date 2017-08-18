---
title:ADD CONSTRAINT
summary: Use the ADD CONSTRAINT statement to add constraints to columns.
toc: false
---

`ADD CONSTRAINT`[语句](sql-staements.md)是`ALTER TABLE`的一部分，用于向列中添加以下[约束](constrains.html):

- [检查](check.html)
- [外键](foreign-key.html)
- [唯一](unique.html)

{{site.data.alerts.callout_info}}
<a href="primary-key.html">主键约束</a>和<a href="not-null.html">非空约束</a>只能通过<a href="create-table.html"><code>CREATE TABLE</code></a>添加。<a href="default-value.html">默认约束</a>则通过<a href="alter-column.html"><code>ALTER COLUMN</code></a>来管理。{{site.data.alerts.end}}

<div id="toc"></div>

## 简介

{% include sql/{{ page.version.version }}/diagrams/add_constraint.html %}

## 权限

用户必须对数据表有 `CREATE` 的[权限](privileges.html) 。

## 参数

| 参数           | 描述           |
| ------------ | ------------ |
| `table_name` | 包含想要添加限制的列的数据表的名称 |
| `name`       | 所添加限制的名称，对于其表必须是唯一的，并遵循这些[标识符规则](keywords-and-identifiers.html#identifiers)。
| `constraint_elem` | 想要添加的[检查](check.html), [外键](foreign-key.html), [唯一](unique.html)限制。<br/><br/>添加/更改默认约束是通过[`ALTER COLUMN`](alter-column.html)。<br/><br/>添加/更改表中的主键约束不能通过 `ALTER TABLE`实现;它只能在[数据表创建](create-table.html#create-a-table-primary-key-defined)期间被指定。 |

## 示例

### 添加唯一约束

添加[唯一约束](unique.html)要求所有列的值彼此不同(*NULL* 值除外)。

``` sql
> ALTER TABLE orders ADD CONSTRAINT id_customer_unique UNIQUE (id, customer);
```

### 添加检查约束

添加[检查约束](check.html)要求所有列的值对于布尔值表达式来说均为`TRUE`。

``` sql
> ALTER TABLE orders ADD CONSTRAINT total_0_check CHECK (total > 0);
```

### 添加外键约束

在将[外键约束](foreign-key.html)添加到列之前，这些列必须已经被编入索引。如果没有被编入索引，使用[`CREATE INDEX`](create-index.html) 对其进行索引，然后使用`ADD CONSTRAINT`语句将外键索引加入到列中。

例如，假设你有两个简单的列表，`订单`和`客户`。

~~~ sql
> SHOW CREATE TABLE customers;
~~~

~~~
+-----------+-------------------------------------------------+
|   Table   |                   CreateTable                   |
+-----------+-------------------------------------------------+
| customers | CREATE TABLE customers (␤                       |
|           |     id INT NOT NULL,␤                           |
|           |     "name" STRING NOT NULL,␤                    |
|           |     address STRING NULL,␤                       |
|           |     CONSTRAINT "primary" PRIMARY KEY (id ASC),␤ |
|           |     FAMILY "primary" (id, "name", address)␤     |
|           | )                                               |
+-----------+-------------------------------------------------+
(1 row)
~~~

~~~ sql
> SHOW CREATE TABLE orders;
~~~

~~~
+--------+-------------------------------------------------------------------------------------------------------------+
| Table  |                                                 CreateTable                                                 |
+--------+-------------------------------------------------------------------------------------------------------------+
| orders | CREATE TABLE orders (␤                                                                                      |
|        |     id INT NOT NULL,␤                                                                                       |
|        |     customer_id INT NULL,␤                                                                                  |
|        |     status STRING NOT NULL,␤                                                                                |
|        |     CONSTRAINT "primary" PRIMARY KEY (id ASC),␤                                                             |
|        |     FAMILY "primary" (id, customer_id, status),␤                                                            |
|        |     CONSTRAINT check_status CHECK (status IN ('open':::STRING, 'complete':::STRING, 'cancelled':::STRING))␤ |
|        | )                                                                                                           |
+--------+-------------------------------------------------------------------------------------------------------------+
(1 row)
~~~

为了确保`orders.customer_id`列中的每个值都与`orders.id`列中的唯一值匹配,需要要将Foreign Key约束添加到`orders.customer_id`。所以首先需要在`orders.customer_id`上创建一个索引。

~~~ sql
> CREATE INDEX ON orders (customer_id);
~~~

然后再添加外键约束：

~~~ sql
> ALTER TABLE orders ADD CONSTRAINT customer_fk FOREIGN KEY (customer_id) REFERENCES customers (id);
~~~

如果在创建索引前添加这个约束的话，将会得到一个报错：

~~~
pq: foreign key requires an existing index on columns ("customer_id")
~~~

## 参考

- [约束](constraints.html)
- [`ALTER COLUMN`](alter-column.html)
- [`CREATE TABLE`](create-table.html)
- [`ALTER TABLE`](alter-table.html)
