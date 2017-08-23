# 使用 CockroachDB 构建一个 Go App

- [使用 pq](build-a-go-app-with-cockroachdb.md) 
- 使用 GORM

本教程将向你介绍，如何使用与 PostgreSQL 兼容的驱动程序或者 ORM，用 CockroachDB  构建一个简单的 Go 应用程序。我们已经测试过 [Go pq 驱动程序](https://godoc.org/github.com/lib/pq) 和 [GORM ORM ](http://jinzhu.me/gorm/)，并能推荐给你们使用，所以它们才会出现在这里。

> **提示：**
> 
> 对于更实际的 GORM 和 CockroachDB 使用，见 [examples-orms](https://github.com/cockroachdb/examples-orms) 代码库。

## 准备

确定你已经[安装了 CockroachDB](install-cockroachdb.md)。

## 第一步：安装 GORM ORM

运行以下命令安装 [GORM](http://jinzhu.me/gorm/)：

~~~ shell
$ go get -u github.com/jinzhu/gorm
~~~

## 第二步：启动一个集群

本教程只需要一个在运行在非安全模式下的 CockroachDB 节点。

```sh
$ cockroach start --insecure \
--store=hello-1 \
--host=localhost
```

如果需要模拟一个真正的集群，可以查看[启动一个本地集群](start-a-local-cluster.md)教程，启动和加入另外的节点非常容易。

打开一个新的终端窗口，启动节点 2：

```sh
cockroach start --insecure \
--store=hello-2 \
--host=localhost \
--port=26258 \
--http-port=8081 \
--join=localhost:26257
```

再打开一个新的终端窗口，启动节点 3：

```sh
cockroach start --insecure \
--store=hello-3 \
--host=localhost \
--port=26259 \
--http-port=8082 \
--join=localhost:26257
```

## 第三步：创建一个用户

打开一个新的终端窗口，以 `root` 用户身份使用 [cockroach user](create-and-manage-users.md) 命令创建一个新用户 `maxroach`。

```sh
cockroach user set maxroach --insecure
```

## 第四步：创建数据库并授权

以 `root` 用户使用 [内建 SQL 客户端](use-the-built-in-sql-client.md)创建 `bank` 数据库。

```sh
cockroach sql --insecure -e 'CREATE DATABASE bank'
```

接下来为 `maxroach` 用户[授权](grant.html)。

```sh
cockroach sql --insecure -e 'GRANT ALL ON DATABASE bank TO maxroach'
```

## 第五步：运行 GO 代码

以下代码使用 [GORM](http://jinzhu.me/gorm/) ORM 将特定于 Go 的对象映射为 SQL 操作。比如，`db.AutoMigrate(&Account{})` 基于 Account 模型创建了一张 `accounts` 表， `db.Create(&Account{})` 向这个表中插入了几行，`db.Find(&accounts)` 检索数据并输出余额。

复制以下代码或[直接下载](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/app/gorm-basic-sample.go)。

~~~ go
package main

import (
    "fmt"
    "log"

    // Import GORM-related packages.
    "github.com/jinzhu/gorm"
    _ "github.com/jinzhu/gorm/dialects/postgres"
)

// Account is our model, which corresponds to the "accounts" database table.
type Account struct {
    ID      int `gorm:"primary_key"`
    Balance int
}

func main() {
    // Connect to the "bank" database as the "maxroach" user.
    const addr = "postgresql://maxroach@localhost:26257/bank?sslmode=disable"
    db, err := gorm.Open("postgres", addr)
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    // Automatically create the "accounts" table based on the Account model.
    db.AutoMigrate(&Account{})

    // Insert two rows into the "accounts" table.
    db.Create(&Account{ID: 1, Balance: 1000})
    db.Create(&Account{ID: 2, Balance: 250})

    // Print out the balances.
    var accounts []Account
    db.Find(&accounts)
    fmt.Println("Initial balances:")
    for _, account := range accounts {
        fmt.Printf("%d %d\n", account.ID, account.Balance)
    }
}
~~~

然后运行代码：
~~~ shell
$ go run gorm-basic-sample.go
~~~

输出应该是：

~~~ shell
Initial balances:
1 1000
2 250
~~~

可以使用[内建 SQL 客户端](use-the-built-in-sql-client.html)验证数据库表和行被成功创建：

~~~ shell
$ cockroach sql --insecure -e 'SHOW TABLES' --database=bank
~~~

~~~
+----------+
|  Table   |
+----------+
| accounts |
+----------+
(1 row)
~~~

~~~ shell
$ cockroach sql --insecure -e 'SELECT id, balance FROM accounts' --database=bank
~~~

~~~
+----+---------+
| id | balance |
+----+---------+
|  1 |    1000 |
|  2 |     250 |
+----+---------+
(2 rows)
~~~

## 下一步：

阅读 [GORM ORM](http://jinzhu.me/gorm/) 使用说明并在 [examples-orms](https://github.com/cockroachdb/examples-orms) 库里尝试更多真实的 CockroachDB GORM 实现。

你也许有兴趣于使用一个本地集群探索下面的 CockroachDB 的核心功能：

- [数据复制](demo-data-replication.md)
- [容错和恢复](demo-fault-tolerance-and-recovery.md)
- [自动再平衡](demo-automatic-rebalancing.md)
- [自动云迁移](demo-automatic-cloud-migration.md)
