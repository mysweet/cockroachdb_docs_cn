# 用 CockroachDB做一个Node.js的App

了解怎样在一个使用了Node.js pg 驱动程序的简单Node.js应用程序中使用CockroachDB

[使用pg](https://www.cockroachlabs.com/docs/stable/build-a-nodejs-app-with-cockroachdb.html) [使用Sequelize](https://www.cockroachlabs.com/docs/stable/build-a-nodejs-app-with-cockroachdb-sequelize.html)

本教程将向你介绍如何使用与PostgreSQL兼容的驱动程序或ORM构建一个简单的使用CockroachDB的Node.js应用程序。我们已经测试过并且可以推荐使用Node.js pg驱动程序和Sequelize ORM，所以这些在这里起重要作用。

## 开始之前

确保你已经安装[CockroachDB](https://www.cockroachlabs.com/docs/stable/install-cockroachdb.html)

## 第一步：安装Node.js安装包

安装[Node.js pg driver](https://www.npmjs.com/package/pg)来使你的应用程序与CockroachDB通讯

```sh
$ npm install pg
```

在你页面上的样例app同样需要[async](https://www.npmjs.com/package/async)：

```sh
$ npm install async
```

## 第二步：搭建一个集群

为了本教程的目的，你只需要运行一个非可靠的CockroachDB节点

```sh
$ cockroach start --insecure \
--store=hello-1 \
--host=localhost
```

但是就像你在[搭建一个本地集群](https://www.cockroachlabs.com/docs/stable/start-a-local-cluster.html)的教程中看到过，如果你想要模拟一个真正的集群，搭建和加入附加节点是非常容易的。

在一个新的终端，搭建节点2： 

```sh
$ cockroach start --insecure \
--store=hello-2 \
--host=localhost \
--port=26258 \
--http-port=8081 \
--join=localhost:26257
```

在一个新的终端，搭建节点3：

```sh
$ cockroach start --insecure \
--store=hello-3 \
--host=localhost \
--port=26259 \
--http-port=8082 \
--join=localhost:26257
```

## 第三步：创建一个用户

在一个新的终端，作为`root`用户，使用[cockroach user](https://www.cockroachlabs.com/docs/stable/create-and-manage-users.html)命令来创建一个新的用户`maxroach`

```sh
$ cockroach user set maxroach --insecure
```

## 第四步：创建一个数据库和授权

作为`root`用户，使用[内置 SQL 客户端](https://www.cockroachlabs.com/docs/stable/use-the-built-in-sql-client.html)来创建一个`bank`数据库

```sh
$ cockroach sql --insecure -e 'CREATE DATABASE bank'
```

然后[授权](https://www.cockroachlabs.com/docs/stable/grant.html)给`maxroach`用户

```sh
$ cockroach sql --insecure -e 'GRANT ALL ON DATABASE bank TO maxroach'
```

## 第五步： 运行Node.js代码

现在你有一个数据库和一个用户，你将运行代码来创建一个表格并添加一些行，然后你将运行代码来读取并更新这些值来作为一个原子事务

### 基本声明

首先，用下面的代码作为`maxroach`用户去连接并且执行一些基本的SQL声明，创建一个表格并添加一些行，然后读取并输出这些行

下载[basic-sample.js](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/app/basic-sample.js)文件,或者你自己创建一个文件然后把代码赋值过去

```js
var async = require('async');

// Require the driver.
var pg = require('pg');

// Connect to the "bank" database.
var config = {
  user: 'maxroach',
  host: 'localhost',
  database: 'bank',
  port: 26257
};

// Create a pool.
var pool = new pg.Pool(config);

pool.connect(function (err, client, done) {
  // Closes communication with the database and exits.
  var finish = function () {
    done();
    process.exit();
  };

  if (err) {
    console.error('could not connect to cockroachdb', err);
    finish();
  }
  async.waterfall([
    function (next) {
      // Create the "accounts" table.
      client.query('CREATE TABLE IF NOT EXISTS accounts (id INT PRIMARY KEY, balance INT);', next);
    },
    function (results, next) {
      // Insert two rows into the "accounts" table.
      client.query('INSERT INTO accounts (id, balance) VALUES (1, 1000), (2, 250);', next);
    },
    function (results, next) {
      // Print out the balances.
      client.query('SELECT id, balance FROM accounts;', next);
    },
  ],
  function (err, results) {
    if (err) {
      console.error('error inserting into and selecting from accounts', err);
      finish();
    }

    console.log('Initial balances:');
    results.rows.forEach(function (row) {
      console.log(row);
    });

    finish();
  });
});
```

然后运行代码

```sh
$ node basic-sample.js
```

输出的应该是

```
Initial balances:
{ id: '1', balance: '1000' }
{ id: '2', balance: '250' }
```

### 事务(重启逻辑)

接下来，使用下面的代码再次作为`maxroach`用户去连接，但是这次执行一批声明作为一个原子事务来将储存的东西从一个账户转移到另一个账户，然后读取这些更新过后的值，其中所有包含的语句都被提交或者中止

下载[txn-sample.js](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/app/txn-sample.js)文件，或者你自己创建一个文件然后把代码复制过去

> **注意：**<br>使用默认的`SERIALIZABLE`隔离层级，CockeoachDB可能会要求客户端在读写争用的情况下重启事务。CockroachDB提供了一个通用的在事务中运行的重启函数，并会在需要的时候重启。你可以在这里复制这个重启函数然后粘贴到你的代码中去。

```js
var async = require('async');

// Require the driver.
var pg = require('pg');

// Connect to the cluster.
var config = {
  user: 'maxroach',
  host: 'localhost',
  database: 'bank',
  port: 26257
};

// Wrapper for a transaction.
// This automatically re-calls "op" with the client as an argument as
// long as the database server asks for the transaction to be retried.
function txnWrapper(client, op, next) {
  client.query('BEGIN; SAVEPOINT cockroach_restart', function (err) {
    if (err) {
      return next(err);
    }

    var released = false;
    async.doWhilst(function (done) {
      var handleError = function (err) {
        // If we got an error, see if it's a retryable one and, if so, restart.
        if (err.code === '40001') {
          // Signal the database that we'll retry.
          return client.query('ROLLBACK TO SAVEPOINT cockroach_restart', done);
        }
        // A non-retryable error; break out of the doWhilst with an error.
        return done(err);
      };

      // Attempt the work.
      op(client, function (err) {
        if (err) {
          return handleError(err);
        }
        var opResults = arguments;

        // If we reach this point, release and commit.
        client.query('RELEASE SAVEPOINT cockroach_restart', function (err) {
          if (err) {
            return handleError(err);
          }
          released = true;
          return done.apply(null, opResults);
        });
      });
    },
    function () {
      return !released;
    },
    function (err) {
      if (err) {
        client.query('ROLLBACK', function () {
          next(err);
        });
      } else {
        var txnResults = arguments;
        client.query('COMMIT', function(err) {
          if (err) {
            return next(err);
          } else {
            return next.apply(null, txnResults);
          }
        });
      }
    });
  });
}

// The transaction we want to run.
function transferFunds(client, from, to, amount, next) {
  // Check the current balance.
  client.query('SELECT balance FROM accounts WHERE id = $1', [from], function (err, results) {
    if (err) {
      return next(err);
    } else if (results.rows.length === 0) {
      return next(new Error('account not found in table'));
    }

    var acctBal = results.rows[0].balance;
    if (acctBal >= amount) {
      // Perform the transfer.
      async.waterfall([
        function (next) {
          // Subtract amount from account 1.
          client.query('UPDATE accounts SET balance = balance - $1 WHERE id = $2', [amount, from], next);
        },
        function (updateResult, next) {
          // Add amount to account 2.
          client.query('UPDATE accounts SET balance = balance + $1 WHERE id = $2', [amount, to], next);
        }, function (updateResult, next) {
          // Fetch account balances after updates.
          client.query('SELECT id, balance FROM accounts', function (err, selectResult) {
            next(err, selectResult ? selectResult.rows : null);
          });
        }
      ], next);
    } else {
      next(new Error('insufficient funds'));
    }
  });
}

// Create a pool.
var pool = new pg.Pool(config);

pool.connect(function (err, client, done) {
  // Closes communication with the database and exits.
  var finish = function () {
    done();
    process.exit();
  };

  if (err) {
    console.error('could not connect to cockroachdb', err);
    finish();
  }

  // Execute the transaction.
  txnWrapper(client,
  function (client, next) {
    transferFunds(client, 1, 2, 100, next);
  },
  function (err, results) {
    if (err) {
      console.error('error performing transaction', err);
      finish();
    }

    console.log('Balances after transfer:');
    results.forEach(function (result) {
      console.log(result);
    });

    finish();
  });
});
```

然后运行代码

```sh
$ node txn-sample.js
```

输出的应该是

```
Balances after transfer:
{ id: '1', balance: '900' }
{ id: '2', balance: '350' }
```

当然，如果你想要证明储存的资源已经从一个账户转到了另一个账户，使用[内置 SQL 客户端](https://github.com/TechCatsLab/cockroachdb_docs_cn/blob/master/v1.0/use-the-built-in-sql-client.html):

```sh
$ cockroach sql --insecure -e 'SELECT id, balance FROM accounts' --database=bank
```

```
+----+---------+
| id | balance |
+----+---------+
|  1 |     900 |
|  2 |     350 |
+----+---------+
(2 列)
```

## 接下来做什么？

阅读更多关于使用[Node.js pg driver](https://www.npmjs.com/package/pg)

您可能也有兴趣使用本地集群来探索以下CoreroachDB核心功能:

* [数据备份](https://www.cockroachlabs.com/docs/stable/demo-data-replication.html)
* [容错 & 修复](https://www.cockroachlabs.com/docs/stable/demo-fault-tolerance-and-recovery.html)
* [自动化再平衡](https://www.cockroachlabs.com/docs/stable/demo-automatic-rebalancing.html)
* [自动化云迁移](https://www.cockroachlabs.com/docs/stable/demo-automatic-cloud-migration.html)

