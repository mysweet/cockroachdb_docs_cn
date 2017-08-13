


# 用CockroachDB做一个Java的App

了解怎样在一个使用了jdbc 驱动程序的简单Java应用程序中使用CockroachDB

[使用jdbc](https://www.cockroachlabs.com/docs/stable/build-a-java-app-with-cockroachdb.html) [使用Hibernate](https://www.cockroachlabs.com/docs/stable/build-a-java-app-with-cockroachdb-hibernate.html)

本教程将向你介绍如何使用与PostgreSQL兼容的驱动程序或ORM构建一个简单的使用CockroachDB的Java应用程序。我们已经测试过并且可以推荐使用[Java jdbc](https://jdbc.postgresql.org/)驱动程序和[Hibernate ORM](http://hibernate.org/)，所以这些在这里起重要作用。

## 开始之前

确保你已经安装[CockroachDB](https://www.cockroachlabs.com/docs/stable/install-cockroachdb.html)

## 第一步：安装Java jdbc 驱动程序

像[官方文档](https://jdbc.postgresql.org/documentation/head/setup.html)描述的那样下载并安装Java jdbc驱动程序

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

## 第五步：运行Java代码

现在你有一个数据库和一个用户，你将运行代码来创建一个表格并添加一些行，然后你将运行代码来读取并更新这些值来作为一个原子事务

### 基本声明

首先，用下面的代码作为`maxroach`用户去连接并且执行一些基本的SQL声明，创建一个表格并添加一些行，然后读取并输出这些行

下载[BasicSample.java](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/app/BasicSample.java)文件,或者你自己创建一个文件然后把代码赋值过去

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
```

### 事务(重启逻辑)

接下来，使用下面的代码再次作为`maxroach`用户去连接，但是这次执行一批声明作为一个原子事务来将储存的东西从一个账户转移到另一个账户，其中所有包含的语句都被提交或者中止

下载[TxnSample.java](https://raw.githubusercontent.com/cockroachdb/docs/master/_includes/app/TxnSample.java)文件，或者你自己创建一个文件然后把代码复制过去

> **注意：**

> 使用默认的`SERIALIZABLE`隔离层级，CockeoachDB可能会要求客户端在读写争用的情况下重启事务。CockroachDB提供了一个通用的在事务中运行的重启函数，并会在需要的时候重启。你可以在这里复制这个重启函数然后粘贴到你的代码中去。

```java

import java.sql.*;

/*
  You can compile and run this example with a command like:
  javac TxnSample.java && java -cp .:~/path/to/postgresql-9.4.1208.jar TxnSample
  You can download the postgres JDBC driver jar from https://jdbc.postgresql.org.
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



运行过代码之后，使用[内置SQL客户端](https://www.cockroachlabs.com/docs/stable/use-the-built-in-sql-client.html)来验证储存的资源已经从一个账户转移到了另一个账户

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

## 接下来做什么？

阅读更多关于使用[Java jdbc driver](https://jdbc.postgresql.org/)

您可能也有兴趣使用本地集群来探索以下CoreroachDB核心功能:

- [数据备份](https://www.cockroachlabs.com/docs/stable/demo-data-replication.html)

- [容错 & 修复](https://www.cockroachlabs.com/docs/stable/demo-fault-tolerance-and-recovery.html)

- [自动化再平衡](https://www.cockroachlabs.com/docs/stable/demo-automatic-rebalancing.html)

- [自动化云迁移](https://www.cockroachlabs.com/docs/stable/demo-automatic-cloud-migration.html)


