---
语句: ADD COLUMN
说明: 使用 ADD COLUMN 语句向数据表中添加一列。
是否可重复: 否
---

`ADD COLUMN` [语句](sql-statements.html)是 `ALTER TABLE` 的一部分，用于向数据表中添加一列。

<div id="toc"></div>

## 概要

{% 包含 sql 语句/{{ 页.版本.版本 }}/图标/add_column.html %}

## 权限

用户需要具有 `CREATE` 数据表的[权限](privileges.html) 。

## 参数

| 参数                  | 描述                                       |
| ------------------- | ---------------------------------------- |
| `table_name`        | 所操作的数据表的表名                               |
| `name`              | 需要添加的列名。列名必须符合[标识符规则](keywords-and-identifiers.html#identifiers)且在表中是唯一的，但是可以使用唯一的名字作为索引和约束条件 |
| `typename`          | 新增的列中允许的[数据类型](data-types.html)          |
| `col_qualification` | 列定义的说明列表，内容包括[列级约束](constraints.html)、[校验](collate.html)、或[列族分配](column-families.html)。<br><br>注意不要为列添加[外键](foreign-key.html)约束。一种解决方法是，添加一个没有约束的列，然后使用 [`CREATE INDEX`](create-index.html) 语句为该列添加索引，再用 [`ADD CONSTRAINT`](add-constraint.html) 语句为该列添加外键约束。 |

## 示例

### 插入一列

~~~ sql
> ALTER TABLE accounts ADD COLUMN names STRING;
~~~

~~~ sql
> SHOW COLUMNS FROM accounts;
~~~

~~~
+-----------+-------------------+-------+---------+-----------+
|   Field   |       Type        | Null  | Default |  Indices  |
+-----------+-------------------+-------+---------+-----------+
| id        | INT               | false | NULL    | {primary} |
| balance   | DECIMAL           | true  | NULL    | {}        |
| names     | STRING            | true  | NULL    | {}        |
+-----------+-------------------+-------+---------+-----------+
~~~

### 插入多列

~~~ sql
> ALTER TABLE accounts ADD COLUMN location STRING, ADD COLUMN amount DECIMAL;
~~~

~~~ sql
> SHOW COLUMNS FROM accounts;
~~~

~~~
+-----------+-------------------+-------+---------+-----------+
|   Field   |       Type        | Null  | Default |  Indices  |
+-----------+-------------------+-------+---------+-----------+
| id        | INT               | false | NULL    | {primary} |
| balance   | DECIMAL           | true  | NULL    | {}        |
| names     | STRING            | true  | NULL    | {}        |
| location  | STRING            | true  | NULL    | {}        |
| amount    | DECIMAL           | true  | NULL    | {}        |
+-----------+-------------------+-------+---------+-----------+

~~~

### 插入一个具有默认值的非空列

~~~ sql
> ALTER TABLE accounts ADD COLUMN interest DECIMAL NOT NULL DEFAULT (DECIMAL '1.3');
~~~

~~~ sql
> SHOW COLUMNS FROM accounts;
~~~
~~~
+-----------+-------------------+-------+---------------------------+-----------+
|   Field   |       Type        | Null  |          Default          |  Indices  |
+-----------+-------------------+-------+---------------------------+-----------+
| id        | INT               | false | NULL                      | {primary} |
| balance   | DECIMAL           | true  | NULL                      | {}        |
| names     | STRING            | true  | NULL                      | {}        |
| location  | STRING            | true  | NULL                      | {}        |
| amount    | DECIMAL           | true  | NULL                      | {}        |
| interest  | DECIMAL           | false | ('1.3':::STRING::DECIMAL) | {}        |
+-----------+-------------------+-------+---------------------------+-----------+
~~~

### 插入一个具有唯一值的非空列

~~~ sql
> ALTER TABLE accounts ADD COLUMN cust_number DECIMAL UNIQUE NOT NULL;
~~~

### 插入一个具有校验功能的列

~~~ sql
> ALTER TABLE accounts ADD COLUMN more_names STRING COLLATE en;
~~~

### 插入一列并将其分配给一个列族

#### 插入一列并将其分配给一个新列族
~~~ sql
> ALTER TABLE accounts ADD COLUMN location1 STRING CREATE FAMILY new_family;
~~~

#### 插入一列并将其分配给一个已存在的列族
~~~ sql
> ALTER TABLE accounts ADD COLUMN location2 STRING FAMILY existing_family;
~~~

#### 插入一列并将其分配给一个列族，如果该列族不存在则创建该列族

~~~ sql
> ALTER TABLE accounts ADD COLUMN new_name STRING CREATE IF NOT EXISTS FAMILY f1;
~~~


## 文档查阅
- [`ALTER TABLE`](alter-table.html) 语句
- [列级约束](constraints.html)
- [校验](collate.html)
- [列族](column-families.html)
