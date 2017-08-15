# 用 CockroachDB 构建一个 Go App

本节将会向你介绍 CockroachDB 如何使用可兼容的 PostgreSQL 驱动程序或者 ORM 来构建一个简单的 Go App。[Go pq driver](https://godoc.org/github.com/lib/pq) 和 [GORM ORM ](http://jinzhu.me/gorm/)我们已经测试过，觉得推荐给你们使用，所以他们才会出现在这里。

## 开始之前

确保你已经 [安装 CockroachDB](install-cockroachdb.html)

## 第一步 安装 Go pq 驱动程序

使用下面的命令可以安装 Go pq 驱动程序：

```shell
$ go get -u github.com/lib/pq
```

## 第二步 启动一个集群

这一步你只需要在非安全模式下运行这个 CockroachDB 命令：

```shell
$ cockroach start --insecure \
--store=hello-1 \
--host=localhost
```

但是，正如你看过的 [启动一个本地集群](start-a-local-cluster.html) 教程，如果你想模拟一个真正的集群，那么启动和加入附加代码也是非常容易的。

在新的终端，运行代码：

```shell
$ cockroach start --insecure \
--store=hello-2 \
--host=localhost \
--port=26258 \
--http-port=8081 \
--join=localhost:26257
```

在新的终端，运行代码：

```shell
$ cockroach start --insecure \
--store=hello-3 \
--host=localhost \
--port=26259 \
--http-port=8082 \
--join=localhost:26257
```

## 第三步 添加用户

在新的终端，root 用户可以使用  [`cockroach user`](https://www.cockroachlabs.com/docs/stable/create-and-manage-users.html) 命令来添加一个新的用户 `maxroach`：

```shell
$ cockroach user set maxroach --insecure
```

## 第四步 创建数据库并授予权限

root 用户可以使用内置的 SQL 客户端创建一个 `bank` 的数据库：

```shell
$ cockroach sql --insecure -e 'CREATE DATABASE bank'
```

然后给 `maxroach` 赋予权限：

```shell
$ cockroach sql --insecure -e 'GRANT ALL ON DATABASE bank TO maxroach'
```

## 第五步 运行 Go 命令

现在已经有一个数据库和一个用户，你可以运行代码来创建表并插入行数据，然后也可以把数据当做一个 [原子事务](transactions.html) 来读取和更新。

### 基本语句

首先，使用下面的代码可以让你作为 `maxroach` 用户连接数据库并执行一些基本的 SQL 语句来创建表、插入行数据以及读取并输出这些行数据。

下载 <a href="https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/app/basic-sample.go" download><code>basic-sample.go</code></a> 文件，或者新建一个文件然后将代码复制进去。

接着运行代码：

```shell
$ go run basic-sample.go
```

输出应该是：

```shell
Initial balances:
1 1000
2 250
```

### 事务（带有重试逻辑）

其次，使用下面的代码可以让你作为 `maxroach` 用户重连数据库，但是这次将会执行一些可被当做原子事务的语句把基金从一个用户转移到另一个用户，在这里所有包含的语句不是提交就是已经中止。

下载 <a href="https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/app/txn-sample.go" download><code>txn-sample.go</code></a> 文件，或者自己新建一个文件然后将代码复制进去。

因为带有默认的可串行化的隔离层，当万一出现读写争用的情况，CockroachDB 可能要求 [客户端重试一个事务](transactions.html#transaction-retries)。CockroachDB 会提供一个通用的 **重试** 函数，它运行在事务里面，必要时候就会重试。对 Go 来说，这个 CockroachDB 重试函数在 CockroachDB 的 Go 客户端的 `crdb` 包里。正如下面所做的， 克隆这个库到你的 `$GOPATH` 里：

```shell
$ mkdir -p $GOPATH/github.com/cockroachdb
```

```shell
$ cd $GOPATH/github.com/cockroachdb
```

```shell
$ git clone git@github.com:cockroachdb/cockroach-go.git
```

然后运行代码：

```shell
$ go run txn-sample.go
```

输出应该是：

```shell
Success
```

然而，如果你想要核实这个已经从一个账户转移到另一个账户的基金，可以使用 [内置 SQL 客户端](use-the-built-in-sql-client.html)：

```shell
$ cockroach sql --insecure -e 'SELECT id, balance FROM accounts' --database=bank
```

```
+----+---------+
| id | balance |
+----+---------+
|  1 |     900 |
|  2 |     350 |
+----+---------+
(2 rows)
```

## 下一节

如果想了解更多，可以使用 [Go pq driver](https://godoc.org/github.com/lib/pq)

你可能对使用本地集群来探索核心的 CockroachDB 特点很感兴趣：

-   [数据复制](demo-data-replication.html)
-   [容错和恢复](demo-fault-tolerance-and-recovery.html)
-   [自动的再平衡](demo-automatic-rebalancing.html)
-   [自动的云迁移](demo-automatic-cloud-migration.html)