# 使用 CockroachDB 开发一个 Python 应用程序

- 使用 psycopg2 
- [使用 SQLAlchemy ](build-a-python-app-with-cockroachdb-sqlalchemy.md)

本教程将向你介绍如何使用与 PostgreSQL 兼容的驱动程序或 ORM，使用CockroachDB，构建一个简单的 Python 应用程序。我们已经测试过并且可以推荐使用 [Python psycopg2 驱动程序](http://initd.org/psycopg/docs/) 和 [SQLAIchemy ORM](https://docs.sqlalchemy.org/en/latest/)，所以它们在这里介绍。

## 准备

确保你已经安装了 [CockroachDB](https://www.cockroachlabs.com/docs/stable/install-cockroachdb.html)。

## 第一步：安装 pdycopg2 驱动程序

运行以下命令来安装 Python psycopg2 驱动程序：

```sh
$ pip install psycopg2
```

安装 psycopg2 的其他方式，请参照[官方文档](http://initd.org/psycopg/docs/install.html)。

## 第二步：启动集群

为了本教程的目的，你只需要一个在非安全模式下运行的 CockroachDB 节点：

```sh
$ cockroach start --insecure \
--store=hello-1 \
--host=localhost
```

但是就像在[启动一个本地集群](start-a-local-cluster.md)教程中看到过的，如果你想要模拟一个真正的集群，启动和加入附加节点是非常容易的。

在一个新的终端，启动节点 2：

```sh
$ cockroach start --insecure \
--store=hello-2 \
--host=localhost \
--port=26258 \
--http-port=8081 \
--join=localhost:26257
```

在一个新的终端，启动节点 3：

```sh
$ cockroach start --insecure \
--store=hello-3 \
--host=localhost \
--port=26259 \
--http-port=8082 \
--join=localhost:26257
```

## 第三步：创建用户

在一个新的终端，以 `root` 用户身份使用 [cockroach user](create-and-manage-users.md) 命令创建一个新用户 `maxroach`。

```sh
$ cockroach user set maxroach --insecure
```

## 第四步：创建数据库并授权

以 `root`用户身份使用[内建 SQL 客户端](use-the-built-in-sql-client.md)创建数据库 `bank`。

```sh
$ cockroach sql --insecure -e 'CREATE DATABASE bank'
```

然后[授权](grant.md)给 `maxroach` 用户。

```sh
$ cockroach sql --insecure -e 'GRANT ALL ON DATABASE bank TO maxroach'
```

## 第五步：运行 Python 代码

现在，有了数据库和用户，你将运行代码来创建一张表并插入一些行，然后，你将运行代码作为[原子事务](transactions.md)读取并更新值。

### 基本声明

首先，用下面的代码以用户 `maxroach` 的身份连接并且执行一些基本的 SQL 语句，创建一张表并插入一些行，然后读取并打印这些行。

下载 [basic-sample.py](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/app/basic-sample.py) 文件，或者，自己创建一个文件然后把代码复制过去。

```python
# Import the driver.
import psycopg2

# Connect to the "bank" database.
conn = psycopg2.connect(database='bank', user='maxroach', host='localhost', port=26257)

# Make each statement commit immediately.
conn.set_session(autocommit=True)

# Open a cursor to perform database operations.
cur = conn.cursor()

# Create the "accounts" table.
cur.execute("CREATE TABLE IF NOT EXISTS accounts (id INT PRIMARY KEY, balance INT)")

# Insert two rows into the "accounts" table.
cur.execute("INSERT INTO accounts (id, balance) VALUES (1, 1000), (2, 250)")

# Print out the balances.
cur.execute("SELECT id, balance FROM accounts")
rows = cur.fetchall()
print('Initial balances:')
for row in rows:
    print([str(cell) for cell in row])

# Close the database connection.
cur.close()
conn.close()

```

然后运行代码：

```sh
$ python basic-sample.py
```

输出应该是：

```
Initial balances:
['1', '1000']
['2', '250']
```

### 事务(带重试逻辑)

接下来，使用下面的代码再次以用户 `maxroach` 的身份连接，但是这次作为一个原子事务执行一组语句，将资金从一个账户转移到另一个账户，其中所有包含的语句都被提交或者退出。

下载 [txn-sample.py](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/app/txn-sample.py) 文件，或者，自己创建一个文件然后把代码复制过去。

> **注意：**

> 使用默认的 `SERIALIZABLE` 隔离级别，CockeoachDB 可能会要求客户端在读写竞争的情况下[重试事务](transactions.md#transaction-retries)。CockroachDB 提供了一个通用的*重试函数*，在事务中运行，并在需要时重试。你可以复制粘贴这里的重试函数到你的代码中去。

```python
# Import the driver.
import psycopg2
import psycopg2.errorcodes

# Connect to the cluster.
conn = psycopg2.connect(database='bank', user='maxroach', host='localhost', port=26257)


def onestmt(conn, sql):
    with conn.cursor() as cur:
        cur.execute(sql)


# Wrapper for a transaction.
# This automatically re-calls "op" with the open transaction as an argument
# as long as the database server asks for the transaction to be retried.
def run_transaction(conn, op):
    with conn:
        onestmt(conn, "SAVEPOINT cockroach_restart")
        while True:
            try:
                # Attempt the work.
                op(conn)

                # If we reach this point, commit.
                onestmt(conn, "RELEASE SAVEPOINT cockroach_restart")
                break

            except psycopg2.OperationalError as e:
                if e.pgcode != psycopg2.errorcodes.SERIALIZATION_FAILURE:
                    # A non-retryable error; report this up the call stack.
                    raise e
                # Signal the database that we'll retry.
                onestmt(conn, "ROLLBACK TO SAVEPOINT cockroach_restart")


# The transaction we want to run.
def transfer_funds(txn, frm, to, amount):
    with txn.cursor() as cur:

        # Check the current balance.
        cur.execute("SELECT balance FROM accounts WHERE id = " + str(frm))
        from_balance = cur.fetchone()[0]
        if from_balance < amount:
            raise "Insufficient funds"

        # Perform the transfer.
        cur.execute("UPDATE accounts SET balance = balance - %s WHERE id = %s",
                    (amount, frm))
        cur.execute("UPDATE accounts SET balance = balance + %s WHERE id = %s",
                    (amount, to))


# Execute the transaction.
run_transaction(conn, lambda conn: transfer_funds(conn, 1, 2, 100))


with conn:
    with conn.cursor() as cur:
        # Check account balances.
        cur.execute("SELECT id, balance FROM accounts")
        rows = cur.fetchall()
        print('Balances after transfer:')
        for row in rows:
            print([str(cell) for cell in row])

# Close communication with the database.
conn.close()

```

然后运行代码：

```sh
$ python txn-sample.py
```

输出应该是：

```
Balances after transfer:
['1', '900']
['2', '350']
```

然而，如果你想验证资金已经从一个账户转移到了另一个账户，使用[内建 SQL 客户端](use-the-built-in-sql-client.md)：

```sh
$ cockroach sql --insecure -e 'SELECT id, balance FROM accounts' --database=bank
```

```
+----+---------+
| id | balance |
+----+---------+
|  1 |     900 |
|  2 |     350 |
+----+---------+
(2 列)
```

## 下一步

阅读更多关于使用 [Python psycopg driver](http://initd.org/psycopg/docs/)。

你也可能有兴趣使用本地集群来探索以下 CoreroachDB 核心功能:

- [数据备份](demo-data-replication.md)
- [容错 & 修复](demo-fault-tolerance-and-recovery.md)
- [自动化再平衡](demo-automatic-rebalancing.md)

