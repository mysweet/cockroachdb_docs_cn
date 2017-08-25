# 使用 CockroachDB 构建一个 Node.js App

- [使用 pg]()
- 使用 Sequelize

本教程介绍了怎样使用 CockroachDB 的 PostgreSQL 兼容的驱动程序或 ORM 构建一个简单的 Node.js 应用，我们已经测试并推荐 [Node.js pg 驱动程序](https://www.npmjs.com/package/pg) 和 [Sequelize ORM](http://docs.sequelizejs.com/en/v3/) 库，所以在这里介绍它们。

## 准备

确保你已经[安装了 CockroachDB](install-cockroachdb.md)。

## 第一步：安装 Sequelize ORM 库

运行以下命令来安装 Sequelize 和 [CockroachDB Node.js 包](https://github.com/cockroachdb/sequelize-cockroachdb)，以消除 CockroachDB 和 PostgreSQL 之间的一些细微差别:

~~~ shell
$ npm install sequelize sequelize-cockroachdb
~~~

## 第二步：启动一个集群

要达到本教程的目的，你只需要有一个运行在非安全模式下的 CockroachDB 节点：

```shell
$ cockroach start --insecure \
--store=hello-1 \
--host=localhost
```

但是就像你在[启动一个本地集群](start-a-local-cluster.md)教程中看到的，如果你想模拟一个真实的集群，启动和增加另外的节点都非常容易。

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

## 第三步：创建用户 

在一个新的终端窗口中，作为 `root` 用户，使用 [`cockroach user`](create-and-manage-users.md) 命令创建一个新用户，`maxroach`。

```shell
$ cockroach user set maxroach --insecure
```

## 第四步：新建数据库并授权 

作为 `root` 用户，使用[内建 SQL 客户端](use-the-built-in-sql-client.md)创建数据库 `bank`：

```shell
$ cockroach sql --insecure -e 'CREATE DATABASE bank'
```

为 `maxroach` 用户授权：

```shell
$ cockroach sql --insecure -e 'GRANT ALL ON DATABASE bank TO maxroach'
```

## 第五步：运行 Node.js 代码

以下代码使用 [Sequelize](http://docs.sequelizejs.com/en/v3/) ORM 映射特定于 Node.js 的对象到 SQL 操作。`Account.sync({force: true})` 语句基于 Account 模型创建了 `accounts` 表 (或者当表已经存在时删除原表并创建新表)，`Account.bulkCreate([...])` 向表中插入行，`Account.findAll()` 从表中进行查询并打印账户结余。

复制代码或直接[下载](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/app/sequelize-basic-sample.js):

~~~ js
var Sequelize = require('sequelize-cockroachdb');

// Connect to CockroachDB through Sequelize.
var sequelize = new Sequelize('bank', 'maxroach', '', {
  dialect: 'postgres',
  port: 26257,
  logging: false
});

// Define the Account model for the "accounts" table.
var Account = sequelize.define('accounts', {
  id: { type: Sequelize.INTEGER, primaryKey: true },
  balance: { type: Sequelize.INTEGER }
});

// Create the "accounts" table.
Account.sync({force: true}).then(function() {
  // Insert two rows into the "accounts" table.
  return Account.bulkCreate([
    {id: 1, balance: 1000},
    {id: 2, balance: 250}
  ]);
}).then(function() {
  // Retrieve accounts.
  return Account.findAll();
}).then(function(accounts) {
  // Print out the balances.
  accounts.forEach(function(account) {
    console.log(account.id + ' ' + account.balance);
  });
  process.exit(0);
}).catch(function(err) {
  console.error('error: ' + err.message);
  process.exit(1);
});
~~~

运行代码：

~~~ shell
$ node sequelize-basic-sample.js
~~~

输出应该是：

~~~ shell
1 1000
2 250
~~~

可以使用[内建 SQL 客户端](use-the-built-in-sql-client.md)验证表和新行被成功创建:

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

学习更多关于使用 [Sequelize ORM](http://docs.sequelizejs.com/en/v3/) 的资料，或者在我们的 [`examples-orms`](https://github.com/cockroachdb/examples-orms) 代码库查找 CockroachDB 使用 Sequelize 更实用的实现。

你也许有兴趣于使用一个本地集群探索下面的 CockroachDB 的核心功能：

- [数据复制](demo-data-replication.md)
- [容错和恢复](demo-fault-tolerance-and-recovery.md)
- [自动再平衡](demo-automatic-rebalancing.md)
- [自动云迁移](demo-automatic-cloud-migration.md)