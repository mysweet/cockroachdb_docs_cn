
# 使用 CockroachDB 构建一个 Java App

- 使用jdbc
- [使用Hibernate](build-a-java-app-with-cockroachdb-hibernate.md)

本教程将向你介绍如何用与 PostgreSQL 兼容的驱动程序或 ORM 使用CockroachDB 构建一个简单的 Java 应用程序。我们已经测试并且可以推荐使用 [Java jdbc](https://jdbc.postgresql.org/) 驱动程序和 [Hibernate ORM](http://hibernate.org/)，所以在这里使用它们。

## 准备

确认你已经[安装了 CockroachDB](install-cockroachdb.md)

## 第一步：安装Java jdbc 驱动程序

按照[官方文档](https://jdbc.postgresql.org/documentation/head/setup.html)描述下载并安装 Java jdbc 驱动程序。

## 第二步：启动一个集群

为了本教程的目的，你只需要在非安全模式下运行一个 CockroachDB 节点：

```sh
$ cockroach start --insecure \
--store=hello-1 \
--host=localhost
```

但是就像你在[启动一个本地集群](start-a-local-cluster.md)的教程中看到过的，如果你想要模拟一个真正的集群，启动和加入附加节点非常容易。

在一个新的终端窗口，启动节点 2：

```sh
$ cockroach start --insecure \
--store=hello-2 \
--host=localhost \
--port=26258 \
--http-port=8081 \
--join=localhost:26257
```

在一个新的终端窗口，启动节点 3：

```sh
$ cockroach start --insecure \
--store=hello-3 \
--host=localhost \
--port=26259 \
--http-port=8082 \
--join=localhost:26257
```

## 第三步：创建一个用户

在一个新的终端窗口，作为 `root` 用户，使用 [cockroach user](create-and-manage-users.md) 命令创建一个新用户`maxroach`。

```sh
$ cockroach user set maxroach --insecure
```

## 第四步：创建数据库并授权

作为 `root` 用户，使用[内建 SQL 客户端](use-the-built-in-sql-client.md)创建`bank`数据库。

```sh
$ cockroach sql --insecure -e 'CREATE DATABASE bank'
```

然后[授权](grant.md)给 `maxroach` 用户。

```sh
$ cockroach sql --insecure -e 'GRANT ALL ON DATABASE bank TO maxroach'
```

## 第五步：运行 Java 代码

现在你有了一个数据库和一个用户，你将运行代码创建一张表并插入几行数据，然后运行代码作为一个原子事务读取并更新这些值。

### 基本语句

首先，用下面的代码作为 `maxroach` 用户连接并且执行一些基本的 SQL 语句，创建一张表并添加一些行，然后读取并输出这些行。

下载 [BasicSample.java](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/app/BasicSample.java) 文件，或者自己创建一个文件然后把代码复制进去。

```java

import java.sql.*;

/*
You can compile and run this example with a command like:
  javac BasicSample.java && java -cp .:~/path/to/postgresql-9.4.1208.jar BasicSample
You can download the postgres JDBC driver jar from https://jdbc.postgresql.org.
*/
public class BasicSample {
    public static void main(String[] args) throws ClassNotFoundException, SQLException {
        // Load the postgres JDBC driver.
        Class.forName("org.postgresql.Driver");

        // Connect to the "bank" database.
        Connection db = DriverManager.getConnection("jdbc:postgresql://127.0.0.1:26257/bank?sslmode=disable", "maxroach", "");

        try {
            // Create the "accounts" table.
            db.createStatement().execute("CREATE TABLE IF NOT EXISTS accounts (id INT PRIMARY KEY, balance INT)");

            // Insert two rows into the "accounts" table.
            db.createStatement().execute("INSERT INTO accounts (id, balance) VALUES (1, 1000), (2, 250)");

            // Print out the balances.
            System.out.println("Initial balances:");
            ResultSet res = db.createStatement().executeQuery("SELECT id, balance FROM accounts");
            while (res.next()) {
                System.out.printf("\taccount %s: %s\n", res.getInt("id"), res.getInt("balance"));
            }
        } finally {
            // Close the database connection.
            db.close();
        }
    }
}


可以用下面的命令编译并运行这个例子：

  javac BasicSample.java && java -cp .:~/path/to/postgresql-9.4.1208.jar BasicSample

可以从 https://jdbc.postgresql.org. 下载 postgres JDBC 驱动程序 jar。

*/

public class BasicSample {

    public static void main(String[] args) throws ClassNotFoundException, SQLException {

        // Load the postgres JDBC driver.

        Class.forName("org.postgresql.Driver");

        // Connect to the "bank" database.

        Connection db = DriverManager.getConnection("jdbc:postgresql://127.0.0.1:26257/bank?sslmode=disable", "maxroach", "");
        try {
            // Create the "accounts" table.
            db.createStatement().execute("CREATE TABLE IF NOT EXISTS accounts (id INT PRIMARY KEY, balance INT)");

            // Insert two rows into the "accounts" table.

            db.createStatement().execute("INSERT INTO accounts (id, balance) VALUES (1, 1000), (2, 250)");



            // Print out the balances.

            System.out.println("Initial balances:");

            ResultSet res = db.createStatement().executeQuery("SELECT id, balance FROM accounts");

            while (res.next()) {

                System.out.printf("\taccount %s: %s\n", res.getInt("id"), res.getInt("balance"));

            }

        } finally {

            // Close the database connection.

            db.close();

        }

    }

}
```

### 事务(带重试逻辑)

接下来，使用下面的代码再次作为 `maxroach` 用户连接，但是这次作为一个原子事务执行一组命令将资金从一个账户转移到另一个账户，其中所有包含的语句或者都被提交或者都被退出。

下载 [TxnSample.java](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/app/TxnSample.java) 文件，或者自己创建一个文件然后把代码复制进去。

> **注意：**

> 使用默认的 `SERIALIZABLE` 隔离层级，在读写竞争的情况下，CockeoachDB 可能会要求[客户端重试事务](transactions.md#transaction-retries)。CockroachDB 提供了一个通用的在事务中运行的**重试函数**，并会在需要的时候重试。你可以在这里复制这个重试函数然后粘贴到你的代码中去。

```java

import java.sql.*;

/*
  可以用下面的命令编译并运行这个例子：
  javac TxnSample.java && java -cp .:~/path/to/postgresql-9.4.1208.jar TxnSample
  可以从 https://jdbc.postgresql.org. 下载 postgres JDBC 驱动程序 jar。
*/

class InsufficientBalanceException extends Exception {}
class AccountNotFoundException extends Exception {
    public int account;
    public AccountNotFoundException(int account) {
        this.account = account;
    }
}

// A simple interface that provides a retryable lambda expression.
interface RetryableTransaction {
    public void run(Connection conn)
        throws SQLException, InsufficientBalanceException, AccountNotFoundException;
}

public class TxnSample {
    public static RetryableTransaction transferFunds(int from, int to, int amount) {
        return new RetryableTransaction() {
            public void run(Connection conn)
                throws SQLException, InsufficientBalanceException, AccountNotFoundException {
                // Check the current balance.
                ResultSet res = conn.createStatement().executeQuery("SELECT balance FROM accounts WHERE id = " + from);
                if(!res.next()) {
                    throw new AccountNotFoundException(from);
                }
                int balance = res.getInt("balance");
                if(balance < from) {
                    throw new InsufficientBalanceException();
                }
                // Perform the transfer.
                conn.createStatement().executeUpdate("UPDATE accounts SET balance = balance - " + amount + " where id = " + from);
                conn.createStatement().executeUpdate("UPDATE accounts SET balance = balance + " + amount + " where id = " + to);
            }
        };
    }

    public static void retryTransaction(Connection conn, RetryableTransaction tx)
        throws SQLException, InsufficientBalanceException, AccountNotFoundException {
        Savepoint sp = conn.setSavepoint("cockroach_restart");
        while(true) {
            try {
                // Attempt the transaction.
                tx.run(conn);

                // If we reach this point, commit the transaction,
                // which implicitly releases the savepoint.
                conn.commit();
                break;
            } catch(SQLException e) {
                // Check if the error code indicates a SERIALIZATION_FAILURE.
                if(e.getErrorCode() == 40001) {
                    // Signal the database that we will attempt a retry.
                    conn.rollback(sp);
                } else {
                    // This is a not a serialization failure, pass it up the chain.
                    throw e;
                }
            }
        }
    }

    public static void main(String[] args) throws ClassNotFoundException, SQLException {
        // Load the postgres JDBC driver.
        Class.forName("org.postgresql.Driver");

        // Connect to the "bank" database.
        Connection db = DriverManager.getConnection("jdbc:postgresql://127.0.0.1:26257/bank?sslmode=disable", "maxroach", "");
            try {
                // We need to turn off autocommit mode to allow for
                // multi-statement transactions.
                db.setAutoCommit(false);
                // Perform the transfer. This assumes the table has
                // already been set up as in the "Build a Test App"
                // tutorial.
                RetryableTransaction transfer = transferFunds(1, 2, 100);
                retryTransaction(db, transfer);
                // Check balances after transfer.
                ResultSet res = db.createStatement().executeQuery("SELECT id, balance FROM accounts");
                while (res.next()) {
                    System.out.printf("\taccount %s: %s\n", res.getInt("id"), res.getInt("balance"));
                }
            } catch(InsufficientBalanceException e) {
                System.out.println("Insufficient balance");
            } catch(AccountNotFoundException e) {
                System.out.println("No users in the table with id " + e.account);
            } catch(SQLException e) {
                System.out.println("SQLException encountered:" + e);
            } finally {
                // Close the database connection.
                db.close();
            }
    }
}
```

运行过代码之后，使用[内建 SQL 客户端](use-the-built-in-sql-client.md)验证资金已经从一个账户转移到了另一个账户。

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

阅读更多关于使用[Java jdbc 驱动程序](https://jdbc.postgresql.org/)。

你可能也有兴趣使用本地集群来探索以下 CoreroachDB 核心功能:

- [数据复制](demo-data-replication.md)

- [容错和恢复](demo-fault-tolerance-and-recovery.md)

- [自动再平衡](demo-automatic-rebalancing.md)

- [自动云迁移](demo-automatic-cloud-migration.md)


