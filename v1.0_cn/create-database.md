# 创建数据库

`CREATE DATABASE` [语句](https://www.cockroachlabs.com/docs/stable/sql-statements.html)用来创建新的 CockroachDB 数据库。  

## 权限要求

只有 `root` 用户可以创建数据库。

## 简介

![](https://github.com/TechCatsLab/cockroachdb_docs_cn/blob/master/images/create-database-synopsis.png)

## 参数

| 参数              | 描述                                       |
| --------------- | ---------------------------------------- |
| `IF NOT EXISTS` | 只有当该名字指向的数据库不存在时才创建新的数据库，如果存在也不返回错误。     |
| `name`          | 创建的数据库的名字，[必须是唯一的](https://www.cockroachlabs.com/docs/stable/create-database.html#create-fails-name-already-in-use) 且符合 [标识符规则](https://www.cockroachlabs.com/docs/stable/keywords-and-identifiers.html#identifiers)。 |
| `encoding`      | `CREATE DATABASE` 语句接收 `ENCODING` 选项来和 PostgreSQL 数据库兼容，但是只支持 `UTF-8` 编码。用别名 `UTF8` 和 `UNICODE` 也是可以的。值必须用单引号包含，且不区分大小写。 例子: `CREATE DATABASE bank ENCODING = 'UTF-8'`. |

## 例子


### 创建数据库 reate a Database
~~~ sql
> CREATE DATABASE bank;

> SHOW DATABASES;
~~~
~~~
+----------+
| Database |
+----------+
| bank     |
| system   |
+----------+
~~~


### 创建错误（创建的名字已被使用）

~~~ sql
> SHOW DATABASES;
~~~
~~~
+----------+
| Database |
+----------+
| bank     |
| system   |
+----------+
~~~
~~~ sql
> CREATE DATABASE bank;
~~~
~~~
pq: database "bank" already exists
~~~
~~~ sql
> SHOW DATABASES;
+----------+
| Database |
+----------+
| bank     |
| system   |
+----------+
~~~

~~~ sql
> CREATE DATABASE IF NOT EXISTS bank;
~~~

SQL 语句不会生成错误，相反即使没有创建数据库也会响应创建数据库。

~~~ sql
> SHOW DATABASES;
~~~

~~~
+----------+
| Database |
+----------+
| bank     |
| system   |
+----------+
~~~

## 查看更多

- [`SHOW DATABASES`](https://www.cockroachlabs.com/docs/stable/show-databases.html)
- [`RENAME DATABASE`](https://www.cockroachlabs.com/docs/stable/rename-database.html)
- [`SET DATABASE`](https://www.cockroachlabs.com/docs/stable/set-vars.html)
- [`DROP DATABASE`](https://www.cockroachlabs.com/docs/stable/drop-database.html)
- [查看更多语句](https://www.cockroachlabs.com/docs/stable/sql-statements.html)
