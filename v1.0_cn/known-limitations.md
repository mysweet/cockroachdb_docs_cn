# 已知限制

本页描述了我们在 [CockroachDB 1.0](../releases_cn/v1.0.md) 发布版中发现的限制，并计划调查并可能在将来的发布版中解决。

## 从大的数据表中移除所有的行

当使用 [`TRUNCATE`](truncate.md) 语句或者 [`DELETE`](delete.md#delete-all-rows) 语句而没有 `WHERE` 子句, 从一张表中移除所有的行时，CockroachDB 将整个操作作为一个单一[事务](transactions.md)批量运行。对于大的表，可能造成包含表数据的节点由于增高的内存和 CPU 使用量而崩溃或者性能表现不佳。

当你需要移除大表中的所有行时，一个解决办法是：

1. 使用 [`SHOW CREATE TABLE`](show-create-table.md) 得到表的模式。
2. 使用 [`DROP TABLE`](drop-table.md) 删除表。
3. 使用 [`CREATE TABLE`](create-table.md) 和我第一步中的输出重建表。

>**注意：**

> 已在 v1.1 中解决，见 [#17016](https://github.com/cockroachdb/cockroach/pull/17016)。

## 数据库模式在事务中改变

在一个单一[事务](transactions.md)中：

- DDL 语句不能在语句之后。一个解决办法是，将 DML 语句 放在 DDL 语句之前，或者将语句分到不同的事务中。
- 包含 [`FOREIGN KEY`](foreign-key.md) 或 [`INTERLEAVE`](interleave-in-parent.md) 子句的 [`CREATE TABLE`](create-table.md) 语句的后面不能有引用新表的语句。
- 一张表在删除后不能以同样的名字重新创建。这在单一事务中是不可能的，因为 `DROP TABLE` 不是立刻删除表的名字。一个解决办法是，将 [`DROP TABLE`](drop-table.md) 和 [`CREATE TABLE`](create-table.md) 语句分到不同的事务中。

## 在恢复备份到新的机器时连接标志

在[部署教程](manual-deployment.md)中，在启动一个集群的第一个节点时，`--join` 标志应该是空的，但是当启动所有的其他节点时，`--join` 标志应该被设置为节点 1 的地址。这个方法保证了所有的节点访问第一个键-值域的拷贝，它是元索引的一部分，元索引确定所有域副本在哪里存储，以及哪些节点要求初始化它们本身并开始接受进来的连接。

保证所有的节点访问第一个键-值域的拷贝，在从整个集群的备份恢复到与备份的集群有不同的 IP 地址时更加困难。在这种情况下，`--join` 标志必须形成一个全连接的有向图。最容易的办法是将所有新节点的地址放入每个节点的 `--join` 标志，这保证了所有节点能够用第一个键-值域的拷连接一个节点。

## `INSERT ON CONFLICT` vs. `UPSERT`

在插入/更新一张表的所有列时，而且表没有第二个索引，我们推荐使用 [`UPSERT`](upsert.md) 语句而不是相当的 [`INSERT ON CONFLICT`](insert.md) 语句。其中 `INSERT ON CONFLICT` 总是执行一个读以决定必要的写，`UPSERT` 语句写不需要读，使得它更快。

这个问题在使用一张具有两列的简单的 SQL 表This issue is particularly relevant when using a simple SQL table of two columns to [模拟直接 KV 访问](frequently-asked-questions.md#can-i-use-cockroachdb-as-a-key-value-store)时特别有关。在这种情况下，确保使用 `UPSERT` 语句。

## 在 SQL shell 历史中重复或合并的命令

[内建 SQL shell](use-the-built-in-sql-client.md) 在 shell 的历史中存储以前执行过的命令。在某些情况下，这些命令竟然被重复了。

而且，在一些终端中，例如不用 `tmux` 的 `st` 或者 `xterm`，以前执行过的命令在shell 的历史中被组合成了一条命令。

## 在 SQL shell 中使用 `\|` 执行大的输入

在[内建 SQL shell](use-the-built-in-sql-client.md) 中，使用 [`\|`](use-the-built-in-sql-client.md#sql-shell-commands) 操作符从一个文件中执行大量的输入，能造成服务器关闭连接。这是因为 `\|` 作为一个单一的查询发送整个文件到服务器，可能超出了服务器从客户端接收的包的大小的上限 (16MB)。

一个解决办法是，使用 `cat data.sql | cockroach sql` [在命令行执行文件](use-the-built-in-sql-client.md#execute-sql-statements-from-a-file)，而不是在交互式 shell 内执行文件。

## 在 `ALTER TABLE ADD COLUMN` 期间由表达式 `DEFAULT` 产生的新值 

在执行带有 [`DEFAULT`](default-value.md) 表达式的 [`ALTER TABLE ADD COLUMN`](add-column.md) 语句时，新的值被产生：

- 使用默认的[搜索路径](sql-name-resolution.md#search-path)，而不用在当前会话中通过 `SET SEARCH_PATH` 配置的搜索路径。
- 使用 UTC 时区，而不用在当前会话中通过 [`SET TIME ZONE`](set-vars.md) 配置的时区。
- 没有默认的数据库，而不用在当前会话中通过 [`SET DATABASE`](set-vars.md) 配置的默认数据库，所以，你必须指定任何表引用的数据库。
- 为 `statement_timestamp()` 函数使用事务时间戳，而不用 `ALTER` 语句被提交的时间。

## 在非平均延迟部署中基于负载的租约再平衡

在节点使用[`--locality`](start-a-node.md#flags) 标志启动时，CockroachDB 试图将副本租约持有者（客户端请求被转发到的副本）放在最接近请求的节点上。这意味着当客户端请求在地理位置上移动时，副本租约持有者也会移动。

然而，你可能见到在下列情况中由在数据中心之间持续的高租约转移率造成的增加的延迟：

- 你的集群运行在几个相互距离非常不同的数据中心。
- 每个节点用一个单一层的 `--locality`，即， `--locality=datacenter=a` 启动。
- 大部分客户端请求被发送到单一的数据中心，因为那里是你所有的应用流量所在。

检测这是否在发生，打开 [Admin UI](explore-the-admin-ui.md)，选择 **Queues** 仪表板，光标悬停在 **Replication Queue** 图上，检查 **Leases Transferred / second** 数据点。如果值持续大于 0，你应该考虑关闭每个节点，并用额外的本地层启动它们，以提高请求延迟。

例如，假设从数据中心 A 中的节点到数据中心 B 中的节点的延迟是 10ms，但是从数据中心 A 中的节点到数据中心 C 中的节点的延迟是 100ms。为了保证 A 和 B 的相对接近被考虑在租约持有者再平衡之内，你可以用一个共同的区域 `--locality=region=foo,datacenter=a` 和 `--locality=region=foo,datacenter=b` 重启数据中心 A 和 B 中的节点，而用一个不用的区域 `--locality=region=bar,datacenter=c` 重启数据中心 C 中的节点。

## 到 `STRING` 的往返没有遵守 `:::` 和 `-` 的优先级

对值为 2**-63 的常数的查询可能被非正确地拒绝，例如：

~~~ sql
> CREATE TABLE t (i int PRIMARY KEY);

> INSERT INTO t VALUES (1), (2), (3);

> SELECT (-9223372036854775808) ::: INT;

> SELECT i FROM t WHERE (i, i) < (1, -9223372036854775808);
~~~

~~~
pq: ($0, $0) < (1, - 9223372036854775808:::INT): tuples ($0, $0), (1, - 9223372036854775808:::INT) are not comparable at index 2: numeric constant out of int64 range
~~~

## 对被整理字符串的过载分解

很多字符串操作对[整理的字符串](collate.md) 没有正确地过载，例如：

~~~ sql
> SELECT 'string1' || 'string2';
~~~

~~~
+------------------------+
| 'string1' || 'string2' |
+------------------------+
| string1string2         |
+------------------------+
(1 row)
~~~

~~~ sql
> SELECT ('string1' collate en) || ('string2' collate en);
~~~

~~~
pq: unsupported binary operator: <collatedstring{en}> || <collatedstring{en}>
~~~

## 引用包含大写字符的整理语言环境

引用一个包含大写字母的[整理](collate.md)语言环境导致错误，例如：

~~~ sql
> CREATE TABLE a (b STRING COLLATE "DE");
~~~

~~~
invalid syntax: statement ignored: invalid locale "DE": language: tag is not well-formed at or near ")"
CREATE TABLE a (b STRING COLLATE "DE");
                                     ^
~~~

一个解决办法是，将语言环境小写或者去掉引号，例如：

~~~ sql
> CREATE TABLE a (b STRING COLLATE "de");

> CREATE TABLE b (c STRING COLLATE DE);
~~~

>**注意：**

> 已在 [v1.0.1](../releases_cn/v1.0.1.md) 中解决，见 [#15917](https://github.com/cockroachdb/cockroach/pull/15917)。

## 用数组类型创建视图

因为数组不被支持，试图在 `SELECT` 查询中用数组[创建视图](create-view.md)会使得接受请求的节点崩溃。

>**注意：**

> 已在 [v1.0.1](../releases_cn/v1.0.1.md) 中解决，见 [#15913](https://github.com/cockroachdb/cockroach/pull/15913)。

## 删除包含视图的数据库

在一个[视图](views.md)查询多张表或者一张表多次时（即，通过 [`UNION`](select.md#combine-multiple-selects-union-intersect-except)），删除包含表的数据库会失败而没有提示。

>**注意：**

> 已在 [v1.0.1](../releases_cn/v1.0.1.md) 中解决，见 [#15983](https://github.com/cockroachdb/cockroach/pull/15983)。

## 修饰来自视图的列

不可能完全修饰一个来自视图的列，因为视图被一个匿名子查询取代，例如：

~~~ sql
> CREATE TABLE test (a INT, b INT);

> CREATE VIEW Caps AS SELECT a, b FROM test;

> SELECT sum(Caps.a) FROM Caps GROUP BY b;
~~~

~~~
pq: source name "caps" not found in FROM clause
~~~

>**注意：**

> 已在 [v1.0.1](../releases_cn/v1.0.1.md) 中解决，见 [#15984](https://github.com/cockroachdb/cockroach/pull/15984)。


## 对单一事务的写和更新范围

单一事务能容纳最多 100,000 个写操作（即，对各个列的修改）和最多 64MB 的合并更新。当一个事务超出了这些限制，会被退出。`INSERT INTO .... SELECT FROM ...` 查询通常需要这里限制。

如果你需要提高这些限制，你可以更新[集群范围的设置](cluster-settings.md) `kv.transaction.max_intents` 和 `kv.raft.command.max_size`。特别对于 `INSERT INTO .. SELECT FROM` 查询，另一个解决办法是使用分开的事务手动将你要插入的数据分页。

## 单一列族的最大尺寸

在创建或更新一行时，如何单一[列族](column-families.md)中值的组合的大小超过了表的最大范围尺寸（默认为 64MB），操作可能失败，或者集群性能下降。

一个解决办法是，你或者可以[手动将表的列分为多个列族](column-families.md#manual-override)，或者你可以用增大的最大范围尺寸[创建一个特定于表的区配置](configure-replication-zones.md#create-a-replication-zone-for-a-table)。

## 在单一节点上的同步的客户端连接并运行查询

在一个节点的客户端连接数和运行的查询数都很高时，节点可能由于耗尽内存而崩溃。这是因为CockroachDB 没有基于节点的可用内存量准确地限制客户端和查询的数量。

为防止内存耗尽，监视每个节点的内存用量并保证在最大的 CockroachDB 内存用量和可用系统内存之间有一些余量。关于 CockroachDB 中内存用量的更多细节，见[博文](https://www.cockroachlabs.com/blog/memory-usage-cockroachdb/)。

## SQL 子表达式和内存使用

很多 SQL 子表达式（即，`ORDER BY`，`UNION`/`INTERSECT`/`EXCEPT`，`GROUP BY`，子查询）在处理查询的节点上在内存里积累中间结果。如果操作符试图处理多余可以放入内存的行，节点或者崩溃或者报告一个内存容量问题。关于 CockroachDB 中内存用量的更多细节，见[博文](https://www.cockroachlabs.com/blog/memory-usage-cockroachdb/)。

## 计数数据表中不同的行

在使用 `count(DISTINCT a.*)` 基于列的一个子集计数一张表中不同的行时，与 `count(*)` 相反，结果几乎总是不正确，例如：

~~~ sql
> CREATE TABLE t (a INT, b INT);

> INSERT INTO t VALUES (1, 2), (1, 3), (2, 1);

> SELECT count(DISTINCT t.*) FROM t;
~~~

~~~
+---------------------+
| count(DISTINCT t.*) |
+---------------------+
|                   1 |
+---------------------+
(1 row)
~~~

一个解决办法是，明确列出列，例如：

~~~ sql
> SELECT count(DISTINCT (t.a, t.b)) FROM t;
~~~

~~~
+----------------------------+
| count(DISTINCT (t.a, t.b)) |
+----------------------------+
|                          3 |
+----------------------------+
(1 row)
~~~

## 在 Windows 上作为非管理员用户运行

CockroachDB 默认定期轮换它写入日志的文件，并用一个符号链接指向当前使用的文件。然而，在 Windows 上，非管理员用户不能创建符号链接，这阻止了 CockroachDB 启动，因为它不能创建日志。

为了解决这个问题，非管理员用户必须通过将`--log-dir=` （带空值）传递给`cockroach start` 命令，向 `stdout` （而不是文件）写日志，即：

~~~ shell
$ cockroach.exe start --log-dir= --insecure
~~~

> 注：
> 
> 在 [v1.0.1](../releases_cn/v1.0.1.md) 中已解决。见 [#15916](https://github.com/cockroachdb/cockroach/pull/15916)。

## `OR` 表达式的查询规划

对于一个像 `SELECT * FROM foo WHERE a > 1 OR b > 2` 的查询，即使有合适的索引同时满足 `a > 1` 和 `b > 2`，查询规划器执行一个全表或索引扫描，因为它不能同时使用两个条件。

## `DELETE` 和 `UPDATE` 的权限

每条 [`DELETE`](delete.md) 或 [`UPDATE`](update.md) 语句构造了一条 `SELECT` 语句，即使没有涉及 `WHERE` 字句。其结果是，执行 `DELETE` 或 `UPDATE` 的用户要求在表上有 `DELETE` 和 `SELECT` 或者 `UPDATE` 和 `SELECT` [权限](privileges.md)。

## 数据库模式在准备好的语句执行之间改变

当一条准备好的语句的目标表的模式在准备好的语句执行之前改变了，CockroachDB 应该返回一个错误。反之，CockroachDB 非正确地允许准备好的语句基于改变了的表模式返回结果，例如：

~~~ sql
> CREATE TABLE users (id INT PRIMARY KEY);

> PREPARE prep1 AS SELECT * FROM users;

> ALTER TABLE users ADD COLUMN name STRING;

> INSERT INTO users VALUES (1, 'Max Roach');

> EXECUTE prep1;
~~~

~~~
+----+-----------+
| id |   name    |
+----+-----------+
|  1 | Max Roach |
+----+-----------+
(1 row)
~~~

而且，一条准备好的 [`INSERT`](insert.md)，[`UPSERT`](upsert.md) 或者 [`DELETE`](delete.md) 语句当被写入的表的模式在准备好的语句执行之前改变时，它们的行为是不一致的：

- 如果列数增加了，准备好的语句返回错误，但尽管如此还是写入了数据。
- 如果列数不变但类型变了，准备好的语句写入数据，而且不返回错误。

## 删除在同一张表上与另一个索引交错的索引

> **注意：**
> 
> 在 1.1 版中解决了。见 <a href="https://github.com/cockroachdb/cockroach/pull/17860">#17860</a>。

在不太可能的情况下，你[交错](interleave-in-parent.md)一个索引到同一张表的另一个索引，而且随后[删除](drop-index.md)了被交错的索引，将来在表上的 DDL 操作会失败。

例如：

~~~ sql
> CREATE TABLE t1 (id1 INT PRIMARY KEY, id2 INT, id3 INT);
~~~

~~~ sql
> CREATE INDEX c ON t1 (id2)
      STORING (id1, id3)
      INTERLEAVE IN PARENT t1 (id2);
~~~

~~~ sql
> SHOW INDEXES FROM t1;
~~~

~~~
+-------+---------+--------+-----+--------+-----------+---------+----------+
| Table |  Name   | Unique | Seq | Column | Direction | Storing | Implicit |
+-------+---------+--------+-----+--------+-----------+---------+----------+
| t1    | primary | true   |   1 | id1    | ASC       | false   | false    |
| t1    | c       | false  |   1 | id2    | ASC       | false   | false    |
| t1    | c       | false  |   2 | id1    | N/A       | true    | false    |
| t1    | c       | false  |   3 | id3    | N/A       | true    | false    |
+-------+---------+--------+-----+--------+-----------+---------+----------+
(4 rows)
~~~

~~~ sql
> DROP INDEX t1@c;
~~~

~~~ sql
> DROP TABLE t1;
~~~

~~~
pq: invalid interleave backreference table=t1 index=3: index-id "3" does not exist
~~~

~~~ sql
> TRUNCATE TABLE t1;
~~~

~~~
pq: invalid interleave backreference table=t1 index=3: index-id "3" does not exist
~~~

~~~ sql
> ALTER TABLE t1 RENAME COLUMN id3 TO id4;
~~~

~~~
pq: invalid interleave backreference table=t1 index=3: index-id "3" does not exist
~~~