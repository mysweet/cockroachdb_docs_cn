# 使用 Sequelize 构建一个 Node.js 应用

本节介绍了怎样使用 CockroachDB 的 PostgreSQL 兼容驱动或 ORM 构建一个简单的 Node.js 应用，我们已经推荐了 [Node.js pg 驱动](https://www.npmjs.com/package/pg) 和 [Sequelize ORM](http://docs.sequelizejs.com/en/v3/) 库，在这里将详细介绍。

## 开始之前

确保你已经安装了 [CockroachDB](https://www.cockroachlabs.com/docs/stable/install-cockroachdb.html)。

## Step 1. 安装 Sequelize ORM 库

运行以下命令来安装  [CockroachDB Node.js package](https://github.com/cockroachdb/sequelize-cockroachdb) :

~~~ shell
$ npm install sequelize sequelize-cockroachdb
~~~

## Step 2. 搭建一个集群

要达到本教程的目的，你只需要有一个运行在非安全模式下的节点，但如果你想增加节点的话可以查看 [搭建一个本地集群](https://www.cockroachlabs.com/docs/stable/start-a-local-cluster.html)：

```shell
$ cockroach start --insecure \
--store=hello-1 \
--host=localhost
```

## Step 3. 创建用户 

在作为 root 用户的前提下，使用 `cockroach user` 命令创建一个新用户，`maxroach`

```shell
$ cockroach user set maxroach --insecure
```

## Step 4. 新建一个数据库并进行授权 

以 root 用户身份使用内置 SQL 客户端创建 `bank` 数据库：

```shell
$ cockroach sql --insecure -e 'CREATE DATABASE bank'
```

为 `maxroach` 用户授权：

```shell
$ cockroach sql --insecure -e 'GRANT ALL ON DATABASE bank TO maxroach'
```

## Step 5. 运行 Node.js 代码

以下代码使用 [Sequelize](http://docs.sequelizejs.com/en/v3/) ORM 库来映射 Node.js 具体对象到 SQL 操作。`Account.sync({force: true})` 语句基于用户模板创建了一个 `accounts` 表 (或者当表已经存在时删除原表并创建新表)，`Account.bulkCreate([...])` 语句向表中插入了一条记录， `Account.findAll()` 语句从表中进行查询并输出。

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

输出：

~~~ shell
1 1000
2 250
~~~

可以使用内置 SQL 客户端验证表和新记录是否被创建:

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

学习更多关于 [Sequelize ORM](http://docs.sequelizejs.com/en/v3/) 的内容或在我们的 [`examples-orms`](https://github.com/cockroachdb/examples-orms) 代码仓库查找 CockroachDB 的 Sequelize 实现。
