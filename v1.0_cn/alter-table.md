# 修改表

使用 `ALTER TABLE` 语句来修改表的内容。

`ALTER TABLE` [语句](https://www.cockroachlabs.com/docs/stable/sql-statements.html) 用于表的修改。

> **注意：**要了解 CockroachDB 是怎样在没有表请求锁或用户不可见时期修改元素的，请查看 [在 CockroachDB 中修改在线模块](https://www.cockroachlabs.com/blog/how-online-schema-changes-are-possible-in-cockroachdb/)。

以下是 `ALTER TABLE` 语句的使用详情：

| 子命令                                      | 描述                                       |
| ---------------------------------------- | ---------------------------------------- |
| [`ADD COLUMN`](add-column.html)          | 向表中添加一列。                                 |
| [`ADD CONSTRAINT`](add-constraint.html)  | 为列添加限制。                                  |
| [`ALTER COLUMN`](alter-column.html)      | 给数据列添加 [默认值](https://www.cockroachlabs.com/docs/stable/default-value.html) 或添加 [Not Null 限制](https://www.cockroachlabs.com/docs/stable/not-null.html)。 |
| [`DROP COLUMN`](drop-column.html)        | 移除一列。                                    |
| [`DROP CONSTRAINT`](drop-constraint.html) | 移除列限制。                                   |
| [`RENAME COLUMN`](rename-column.html)    | 修改列的名字。                                  |
| [`RENAME TABLE`](rename-table.html)      | 修改表名。                                    |
| `SPLIT AT`                               | *（文档申请）* 通过在键值对层中划分数据的理想位置来提高性能。         |
