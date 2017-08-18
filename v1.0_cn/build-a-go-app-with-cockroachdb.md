# 使用 CockroachDB 构建一个 Go App

- 使用 pq 
- [使用 GORM](build-a-go-app-with-cockroachdb-gorm.md)

本教程将向你介绍，如何使用与 PostgreSQL 兼容的驱动程序或者 ORM，用 CockroachDB  构建一个简单的 Go 应用程序。我们已经测试过 [Go pq 驱动程序](https://godoc.org/github.com/lib/pq) 和 [GORM ORM ](http://jinzhu.me/gorm/)，并能推荐给你们使用，所以它们才会出现在这里。

## 准备

确保你已经 [安装了 CockroachDB](install-cockroachdb.md)

## 第一步：安装 Go pq 驱动程序

使用下面的命令可以[安装 Go pq 驱动程序](https://godoc.org/github.com/lib/pq)：

```shell
$ go get -u github.com/lib/pq
```

## 第二步：启动一个集群

这一步，你只需要在非安全模式下运行这条 CockroachDB 命令：

```shell
$ cockroach start --insecure \
--store=hello-1 \
--host=localhost
```

但是，正如你看过的 [启动一个本地集群](start-a-local-cluster.md) 教程，如果你想模拟一个真实的集群，那么启动和加入附加代码也是非常容易的。

在新的终端窗口，启动节点 2：

```shell
$ cockroach start --insecure \
--store=hello-2 \
--host=localhost \
--port=26258 \
--http-port=8081 \
--join=localhost:26257
```

在新的终端，启动节点 3：

```shell
$ cockroach start --insecure \
--store=hello-3 \
--host=localhost \
--port=26259 \
--http-port=8082 \
--join=localhost:26257
```

## 第三步：创建用户

在新的终端窗口，`root` 用户可以使用  [`cockroach user`](create-and-manage-users.md) 命令来创建一个新用户 `maxroach`：

```shell
$ cockroach user set maxroach --insecure
```

## 第四步：创建数据库并授权

`root` 用户可以使用[内建的 SQL 客户端](use-the-built-in-sql-client.md)创建一个 `bank` 的数据库：

```shell
$ cockroach sql --insecure -e 'CREATE DATABASE bank'
```

然后给 `maxroach` 用户授权：

```shell
$ cockroach sql --insecure -e 'GRANT ALL ON DATABASE bank TO maxroach'
```

## 第五步：运行 Go 代码

现在已经有一个数据库和一个用户，你可以运行代码来创建表并插入一些数据，然后也可以运行代码作为原子[事务](transactions.md) 来读取和更新数据。

### 基本语句

首先，使用下面的代码可以作为 `maxroach` 用户连接数据库并执行一些基本的 SQL 语句来创建表、插入行数据以及读取并打印这些行。

下载 [<code>basic-sample.go</code>](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/app/basic-sample.go) 文件，或者新建一个文件然后将代码复制进去。

```go
package main

import (
    "database/sql"
    "fmt"
    "log"

    _ "github.com/lib/pq"
)

func main() {
    // Connect to the "bank" database.
    db, err := sql.Open("postgres", "postgresql://maxroach@localhost:26257/bank?sslmode=disable")
    if err != nil {
        log.Fatal("error connecting to the database: ", err)
    }

    // Create the "accounts" table.
    if _, err := db.Exec(
        "CREATE TABLE IF NOT EXISTS accounts (id INT PRIMARY KEY, balance INT)"); err != nil {
        log.Fatal(err)
    }

    // Insert two rows into the "accounts" table.
    if _, err := db.Exec(
        "INSERT INTO accounts (id, balance) VALUES (1, 1000), (2, 250)"); err != nil {
        log.Fatal(err)
    }

    // Print out the balances.
    rows, err := db.Query("SELECT id, balance FROM accounts")
    if err != nil {
        log.Fatal(err)
    }
    defer rows.Close()
    fmt.Println("Initial balances:")
    for rows.Next() {
        var id, balance int
        if err := rows.Scan(&id, &balance); err != nil {
            log.Fatal(err)
        }
        fmt.Printf("%d %d\n", id, balance)
    }
}
```

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

下一步，使用下面的代码可以作为 `maxroach` 用户重连数据库，但是这次将会执行一组语句，作为原子事务的语句将资金从一个帐户转移到另一个帐户，在这里所有包含的语句或者被提交或者被退出。

下载 [<code>txn-sample.go</code>](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/app/txn-sample.go) 文件，或者自己新建一个文件然后将代码复制进去。

```go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "log"

    "github.com/cockroachdb/cockroach-go/crdb"
)

func transferFunds(tx *sql.Tx, from int, to int, amount int) error {
    // Read the balance.
    var fromBalance int
    if err := tx.QueryRow(
        "SELECT balance FROM accounts WHERE id = $1", from).Scan(&fromBalance); err != nil {
        return err
    }

    if fromBalance < amount {
        return fmt.Errorf("insufficient funds")
    }

    // Perform the transfer.
    if _, err := tx.Exec(
        "UPDATE accounts SET balance = balance - $1 WHERE id = $2", amount, from); err != nil {
        return err
    }
    if _, err := tx.Exec(
        "UPDATE accounts SET balance = balance + $1 WHERE id = $2", amount, to); err != nil {
        return err
    }
    return nil
}

func main() {
    db, err := sql.Open("postgres", "postgresql://maxroach@localhost:26257/bank?sslmode=disable")
    if err != nil {
        log.Fatal("error connecting to the database: ", err)
    }

    // Run a transfer in a transaction.
    err = crdb.ExecuteTx(context.Background(), db, nil, func(tx *sql.Tx) error {
        return transferFunds(tx, 1 /* from acct# */, 2 /* to acct# */, 100 /* amount */)
    })
    if err == nil {
        fmt.Println("Success")
    } else {
        log.Fatal("error: ", err)
    }
}
```

因为带有默认的 `SERIALIZABLE` 隔离层，当万一出现读写竞争的情况，CockroachDB 可能要求 [客户端重试一个事务](transactions.md#transaction-retries)。CockroachDB 会提供一个通用的 **重试** 函数，它运行在事务里面，必要时候就会重试。对 Go 来说，这个 CockroachDB 重试函数在 CockroachDB 的 Go 客户端的 `crdb` 包里。正如下面所做的， 克隆这个库到你的 `$GOPATH` 里：

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

然而，如果你想要核实资金已经从一个账户转移到了另一个账户，可以使用 [内建的 SQL 客户端](use-the-built-in-sql-client.md)：

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

## 下一步

阅读更多关于使用 [Go pq 驱动程序](https://godoc.org/github.com/lib/pq)的资料。

你可能对使用本地集群来探索 CockroachDB 的核心功能感兴趣：

-   [数据复制](demo-data-replication.md)
-   [容错和恢复](demo-fault-tolerance-and-recovery.md)
-   [自动化再平衡](demo-automatic-rebalancing.md)
-   [自动化云迁移](demo-automatic-cloud-migration.md)