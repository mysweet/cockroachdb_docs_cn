# SQL FAQs

##如何增大插入 CockroachDB 的数据？

一般情况下，你可以批量使用 [`INSERT`](insert.html) 语句增大插入的数据，每次插入的数据不超过几MB。行的大小决定你可以使用多少，不过通常情况下1000~10000行是最好的选择。详情请查看 [导入数据](https://www.cockroachlabs.com/docs/import-data.html)。

## 在 CockroachDB 中，如何自动得生成唯一的行 ID？

请看 [自动生成唯一ID](faq/auto-generate-unique-ids.html).

## 如何获取插入表中的最后的项的 ID 或者 序列号？

CockroachDB 中没有函数能返回最后插入的数据，但是可以使用 `INSERT` 语句的 [`RETURNING` 子句](insert.html#insert-and-return-values)。

比如，使用 `RETURNING` 返回自动生成的 [`序列号`](serial.html)：

```sql
> CREATE TABLE users (id SERIAL, name STRING);

> INSERT INTO users (name) VALUES ('mike') RETURNING id;
```

## CockroachDB 是否支持 `JOIN` ？

CockroachDB 对 `JOIN` 有基本的、未经优化的支持，我们正在努力改善他的性能。

详情请查看我们的关于 CockroachDB 的 `JOIN` 的博客：

-   [简单朴素: CockroachDB 的 JOIN](https://www.cockroachlabs.com/blog/cockroachdbs-first-join/).
-   [更好的使用 JOIN](https://www.cockroachlabs.com/blog/better-sql-joins-in-cockroachdb/)

## 如何使用交叉表？

[交叉表](interleave-in-parent.html) 通过优化密切相关的表的键值结构来改善查询性能，如果可能要同时读和写，就会试图将数据保存在相同的键值范围内。

## CockroachDB 是否支持  JSON/Protobuf 数据类型？

当前还不行, 但是 [我们正在计划提供 JSON/Protobuf 的数据类型](https://github.com/cockroachdb/cockroach/issues/2969)。

## CockroachDB 将会选择哪个索引来进行查询？

查看 CockroachDB 正在使用哪个索引来进行查询，你可以使用 [`EXPLAIN`](explain.html) 语句，他将会输出这个查询计划，包含任何被用到的索引：

```sql
> EXPLAIN SELECT col1 FROM tbl1;
```

如果想要让查询规划器使用某个索引，可以使用一些特殊的语法索引提示：

```sql
> SELECT col1 FROM tbl1@idx1;
```

## 如何记录 SQL 查询？

对于生产集群，记录 SQL 查询的最佳方法是通过 [整个集群范围的设置](cluster-settings.html) `sql.trace.log_statement_execute`：

```sql
> SET CLUSTER SETTING sql.trace.log_statement_execute = true;
```

打开了以后，集群的每个节点会将所有执行的 SQL 查询写入他的日志文件。当你不再需要记录查询，你可以关闭此设置：

```sql
> SET CLUSTER SETTING sql.trace.log_statement_execute = false;
```

你也可以选择本地测试 CockroachDB，通过一个特定的节点来记录执行的查询，即当启动一个节点时，在 [`cockroach start`](start-a-node.html) 命令后面加上 `--vmodule=executor=2`。比如，本地启动一个节点，记录所有执行的 SQL 查询，你可以这样做：

```shell
$ cockroach start --insecure --host=localhost --vmodule=executor=2
```

## CockroachDB 支持 UUID 类型吗？

当前还不行，但是在一个字节类型的列里存储一个16字节的数组会更好。

##当未使用 `ORDER BY` 时，如何让 CockroachDB 对结果进行排序？

当在一个 `SELECT` 查询语句中未使用 [`ORDER BY`](select.html#sorting-retrieved-values) 子句，检索的行没有按任何一致的标准进行排序。相反，CockroachDB 作为协调节点接收他们，返回他们。

## 参考

-   [产品常见问题](frequently-asked-questions.html)
-   [操作常见问题](operational-faqs.html)