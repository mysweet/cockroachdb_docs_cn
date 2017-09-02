# 修改数据库视图（VIEW）

`ALTER VIEW` [语句](https://www.cockroachlabs.com/docs/stable/sql-statements.html)可以用来修改 View 的名字。

> **注意：**目前还无法更改视图执行的 SELECT 语句，除非删除现有视图并创建新视图。另外，目前还不可能重命名其他视图所依赖的视图，但将来可能会添加此功能。

## 所需权限

用户必须拥有该 view 父数据库的 `DROP` [权限](https://www.cockroachlabs.com/docs/stable/privileges.html) 和 `CREATE` 权限。

## 简介

![](https://github.com/TechCatsLab/cockroachdb_docs_cn/blob/master/images/alter-view-synopsis.png)

## 参数

| 参数          | 描述                                       |
| ----------- | ---------------------------------------- |
| `IF EXISTS` | 只当 `view_name` 指定的 view 退出时才可重命名；如果没有退出，也不会返回错误。 |
| `view_name` | 重新命名的 view 名称。使用:<br><br>`SELECT * FROM information_schema.tables WHERE table_type = 'VIEW';` 来查找名称。 |
| `name`      | view 的新 [`名称`](https://www.cockroachlabs.com/docs/stable/sql-grammar.html#name) ，在该数据库中是唯一的且符合 [标识符规则](https://www.cockroachlabs.com/docs/stable/keywords-and-identifiers.html#identifiers)。 |

## 例子

~~~ sql
> SELECT * FROM information_schema.tables WHERE table_type = 'VIEW';
~~~

~~~ 
+---------------+-------------------+--------------------+------------+---------+
| TABLE_CATALOG |   TABLE_SCHEMA    |     TABLE_NAME     | TABLE_TYPE | VERSION |
+---------------+-------------------+--------------------+------------+---------+
| def           | bank              | user_accounts      | VIEW       |       2 |
| def           | bank              | user_emails        | VIEW       |       1 |
+---------------+-------------------+--------------------+------------+---------+
(2 rows)
~~~

~~~ sql
> ALTER VIEW bank.user_emails RENAME TO bank.user_email_addresses;
~~~

~~~
RENAME VIEW
~~~

~~~ sql
> SELECT * FROM information_schema.tables WHERE table_type = 'VIEW';
~~~

~~~
+---------------+-------------------+----------------------+------------+---------+
| TABLE_CATALOG |   TABLE_SCHEMA    |      TABLE_NAME      | TABLE_TYPE | VERSION |
+---------------+-------------------+----------------------+------------+---------+
| def           | bank              | user_accounts        | VIEW       |       2 |
| def           | bank              | user_email_addresses | VIEW       |       3 |
+---------------+-------------------+----------------------+------------+---------+
(2 rows)
~~~

## 查看更多

- [视图 View](https://www.cockroachlabs.com/docs/stable/views.html)
- [`CREATE VIEW`](https://www.cockroachlabs.com/docs/stable/create-view.html)
- [`SHOW CREATE VIEW`](https://www.cockroachlabs.com/docs/stable/show-create-view.html)
- [`DROP VIEW`](https://www.cockroachlabs.com/docs/stable/drop-view.html)
