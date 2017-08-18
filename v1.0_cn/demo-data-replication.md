# 数据复制实例

本节将会向你介绍一个 CockroachDB 如何复制及分配数据的简单实例。启动一个单节点本地集群开始，你可以写入一些数据，然后添加2个节点接着就可以查看数据是如何自动复制的。然后你可以通过复制下面前5个步骤，再加上另外两个节点，接着再次查看已经存在的所有复制品是如何复制出新的节点的。

## 开始之前

确保你已经 [安装 CockroachDB](install-cockroachdb.html).

## 步骤1 启动一个单节点集群

```shell
$ cockroach start --insecure \
--store=repdemo-node1 \
--host=localhost
```

## 步骤2 写入数据

开启一个新的终端，使用 [`cockroach gen`](generate-cockroachdb-resources.html) 命令来生成一个 `intro` 的示例数据库：

```shell
$ cockroach gen example-data intro | cockroach sql --insecure
```

```sql
CREATE DATABASE
SET
DROP TABLE
CREATE TABLE
INSERT 1
INSERT 1
INSERT 1
INSERT 1
...
```

在同一个终端，打开 [内置的 SQL 命令行](use-the-built-in-sql-client.html) 核实新的 `intro` 数据库是否已经创建并新建了一个表 `mytable`：

```shell
$ cockroach sql --insecure
```

```
# Welcome to the cockroach SQL interface.
# All statements must be terminated by a semicolon.
# To exit: CTRL + D.
```

```sql
> SHOW DATABASES;
```

```
+--------------------+
|      Database      |
+--------------------+
| information_schema |
| pg_catalog         |
| intro              |
| system             |
+--------------------+
(4 rows)
```

```sql
> SHOW TABLES FROM intro;
```

```
+---------+
|  Table  |
+---------+
| mytable |
+---------+
(1 row)
```

```sql
> SELECT * FROM intro.mytable WHERE (l % 2) = 0;
```

```
+----+-----------------------------------------------------+
| l  |                          v                          |
+----+-----------------------------------------------------+
|  0 | !__aaawwmqmqmwwwaas,,_        .__aaawwwmqmqmwwaaa,, |
|  2 | !"VT?!"""^~~^"""??T$Wmqaa,_auqmWBT?!"""^~~^^""??YV^ |
|  4 | !                    "?##mW##?"-                    |
|  6 | !  C O N G R A T S  _am#Z??A#ma,           Y        |
|  8 | !                 _ummY"    "9#ma,       A          |
| 10 | !                vm#Z(        )Xmms    Y            |
| 12 | !              .j####mmm#####mm#m##6.               |
| 14 | !   W O W !    jmm###mm######m#mmm##6               |
| 16 | !             ]#me*Xm#m#mm##m#m##SX##c              |
| 18 | !             dm#||+*$##m#mm#m#Svvn##m              |
| 20 | !            :mmE=|+||S##m##m#1nvnnX##;     A       |
| 22 | !            :m#h+|+++=Xmm#m#1nvnnvdmm;     M       |
| 24 | ! Y           $#m>+|+|||##m#1nvnnnnmm#      A       |
| 26 | !  O          ]##z+|+|+|3#mEnnnnvnd##f      Z       |
| 28 | !   U  D       4##c|+|+|]m#kvnvnno##P       E       |
| 30 | !       I       4#ma+|++]mmhvnnvq##P`       !       |
| 32 | !        D I     ?$#q%+|dmmmvnnm##!                 |
| 34 | !           T     -4##wu#mm#pw##7'                  |
| 36 | !                   -?$##m####Y'                    |
| 38 | !             !!       "Y##Y"-                      |
| 40 | !                                                   |
+----+-----------------------------------------------------+
(21 rows)
```

然后推迟 SQL 命令行：

```sql
> \q
```

## 步骤3 添加两个节点

在一个新的终端，添加节点2：

```shell
$ cockroach start --insecure \
--store=repdemo-node2 \
--host=localhost \
--port=26258 \
--http-port=8081 \
--join=localhost:26257
```

在另一个新的终端，添加节点3：

```shell
$ cockroach start --insecure \
--store=repdemo-node3 \
--host=localhost \
--port=26259 \
--http-port=8082 \
--join=localhost:26257
```

## 步骤4 查看数据是否被复制到新的节点

打开管理界面 `http://localhost:8080`，在右侧点击 **View nodes list** ，你将会看到列表中有3个节点。一开始，副品计数将会少于节点2和节点3。很快，副品计数在所有3个节点上都是相同的，意味着集群中的所有数据已经被复制了3次，每个节点中都有每一条数据的副品。

![replication1](images/replication1.png)

##步骤5 增加复制因数

正如你看到的，CockroachDB 默认复制3次数据。现在，在内置 SQL 命令行或者在一个新的终端，编辑默认的 [replication zone](configure-replication-zones.html) 可以复制5次数据：

```shell
$ echo 'num_replicas: 5' | cockroach zone set .default --insecure -f -
```

```
range_min_bytes: 1048576
range_max_bytes: 67108864
gc:
  ttlseconds: 86400
num_replicas: 5
constraints: []
```

## 步骤6 添加另外两个节点

在一个新的终端，添加节点4：

```shell
$ cockroach start --insecure \
--host=localhost \
--store=repdemo-node4 \
--port=26260 \
--http-port=8083 \
--join=localhost:26257
```

在另一个新的节点，添加节点5：

```shell
$ cockroach start --insecure \
--host=localhost \
--store=repdemo-node5 \
--port=26261 \
--http-port=8084 \
--join=localhost:26257
```

## 步骤7 查看数据是否被复制到新的节点

进入管理界面，你将会看到列表中有5个节点。同样的，一开始，副品计数将少于节点4和节点5。但是因为你将默认的复制因数改成了5，很快，副品计数在所有5个节点上都是相同的，意味着集群中的所有数据已经被复制了3次.

![replication2](images/replication2.png)

## 步骤8 关闭集群

在完成测试集群后，通过切换到每个节点本身的终端并使用 **CTRL + C** 来关闭每一个节点。

对于最后两个节点，关闭进程将会需要更长时间（大概1分钟），最终会强制杀死所有节点。这是因为，如果只留2个节点仍然在线，大多数副品就不再可使用（3 / 5），接着导致集群不能操作。为了加快关闭进程，可在节点终端长按 **CTRL + C** 1秒。

如果你不再计划重新启动这个集群，你可能想要删除节点中的数据缓存：

```shell
$ rm -rf repdemo-node1 repdemo-node2 repdemo-node3 repdemo-node4 repdemo-node5
```

## 下一节

使用一个本地集群来探索其他的 CockroachDB 核心功能：

-   [容错和恢复](demo-fault-tolerance-and-recovery.html)
-   [自动再平衡](demo-automatic-rebalancing.html)
-   [自动云迁移](demo-automatic-cloud-migration.html)

