# 使用 GORM 库构建 GO 语言的 CockroachDB 应用程序

这一节主要讲述如何使用 GORM 构建一个 CockroachDB 应用。

## 开始之前

确定你已经安装了 [CockroachDB](https://www.cockroachlabs.com/docs/stable/install-cockroachdb.html).

## 第一步：安装 GORM ORM

运行一下程序安装 [GORM](http://jinzhu.me/gorm/)

~~~ shell
$ go get -u github.com/jinzhu/gorm
~~~

## 第二步：部署一个集群

只需要一个在非安全模式下的节点就能完成此次测试

```sh
$ cockroach start --insecure \
--store=hello-1 \
--host=localhost
```

如果你需要模拟一个真正的集群，可以查看[部署一个集群](https://www.cockroachlabs.com/docs/stable/start-a-local-cluster.html)

再打开一个终端，开启节点2：

```sh
cockroach start --insecure \
--store=hello-2 \
--host=localhost \
--port=26258 \
--http-port=8081 \
--join=localhost:26257
```

再打开一个终端，开启节点3：

```sh
cockroach start --insecure \
--store=hello-3 \
--host=localhost \
--port=26259 \
--http-port=8082 \
--join=localhost:26257
```

## 第三步：创建一个用户

打开一个新终端，以 root 用户身份使用 [cockroach user](https://www.cockroachlabs.com/docs/stable/create-and-manage-users.html) 命令来创建一个新用户 `maxroach` ：

```sh
cockroach user set maxroach --insecure
```

## 第四步：创建数据库并授予权限

以 root 用户使用 [内置 SQL 客户端](https://www.cockroachlabs.com/docs/stable/use-the-built-in-sql-client.html)创建 `bank` 数据库：

```sh
cockroach sql --insecure -e 'CREATE DATABASE bank'
```

接下来为 `maxroach` 用户[授予权限](https://www.cockroachlabs.com/docs/stable/grant.html)：

```sh
cockroach sql --insecure -e 'GRANT ALL ON DATABASE bank TO maxroach'
```

## 第五步：运行 GO 程序

一下代码使用 [GORM](http://jinzhu.me/gorm/) ORM 来进行 SQL 操作。比如，`db.AutoMigrate(&Account{})` 基于账户模板创建了一个 `accounts` 表， `db.Create(&Account{})` 向这个表中插入了一列，`db.Find(&accounts)` 检索数据并输出。

复制以下代码或[直接复制](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/app/gorm-basic-sample.go)。

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

y运行以下代码：
~~~ shell
$ go run gorm-basic-sample.go
~~~

输出结果：

~~~ shell
Initial balances:
1 1000
2 250
~~~

可以使用[内置 SQL 客户端](https://www.cockroachlabs.com/docs/stable/use-the-built-in-sql-client.html)核实数据库表和数据是否被成功创建：

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

阅读 [GORM ORM](http://jinzhu.me/gorm/) 使用说明并在 [examples-orms](https://github.com/cockroachdb/examples-orms) 库里尝试更多实现。

你也许更感兴趣于使用一个本地集群来探索更多 CockroachDB 的核心特性：

- [数据自我复制](https://www.cockroachlabs.com/docs/stable/demo-data-replication.html)
- [容错和恢复](https://www.cockroachlabs.com/docs/stable/demo-fault-tolerance-and-recovery.html)
- [自动再平衡](https://www.cockroachlabs.com/docs/stable/demo-automatic-rebalancing.html)
- [自动云迁移](https://www.cockroachlabs.com/docs/stable/demo-automatic-cloud-migration.html)
