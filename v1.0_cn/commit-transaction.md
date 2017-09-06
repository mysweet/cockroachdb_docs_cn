# 提交
`COMMIT` [语句](https://www.cockroachlabs.com/docs/stable/sql-statements.html) 提交最近的 [事务](https://www.cockroachlabs.com/docs/stable/transactions.html)，或者使用 [客户端事务重试](https://www.cockroachlabs.com/docs/stable/transactions.html#client-side-transaction-retries)，切断了开始新事务的连接。

当使用[客户端事务重试](https://www.cockroachlabs.com/docs/stable/transactions.html#client-side-transaction-retries)，发出语句后 [`保存点 cockroach_restart`](https://www.cockroachlabs.com/docs/stable/savepoint.html) 将被提交，当 [`释放保存点cockroach_restart`](https://www.cockroachlabs.com/docs/stable/release-savepoint.html) 将被提交而非 `COMMIT`。然而，你仍然必须发出一个提交语句来清除下一个事务的连接。

对于非重试的事务，如果语句 [生成任何错误](https://www.cockroachlabs.com/docs/stable/transactions.html#error-handling)，`COMMIT` 和 `ROLLBACK` 是等同的，这将中止事务和丢弃其声明的所有更新。

## 简介

![](https://github.com/TechCatsLab/cockroachdb_docs_cn/blob/master/images/commit-synopsis.png)

## 权限

提交事务没有[权限](https://www.cockroachlabs.com/docs/stable/privileges.html)要求。但是，事务中的每个语句都需要权限。

## 别名

在 CockroachDB 中，`END` 是 `COMMIT` 语句的一个别名。

## 例子

### 提交一个事务

如何提交一个事务取决于你的应用如何处理 [事务重试](https://www.cockroachlabs.com/docs/stable/transactions.html#transaction-retries)。

#### 客户端可重试事务

当使用 [客户端事务重试](transactions.html#client-side-transaction-retries)时，语句将被 [`RELEASE SAVEPOINT cockroach_restart`](release-savepoint.html) 提交。`COMMIT` 自身只断开与下一个事务的连接。

~~~ sql
> BEGIN;

> SAVEPOINT cockroach_restart;

> UPDATE products SET inventory = 0 WHERE sku = '8675309';

> INSERT INTO orders (customer, sku, status) VALUES (1001, '8675309', 'new');

> RELEASE SAVEPOINT cockroach_restart;

> COMMIT;
~~~

> **警告**：这个例子在假设你是使用客户端干预来处理事务重试的前提之下成立。

#### 自动重启事务

如果你使用事务，CockroachDB将[自动重试](https://www.cockroachlabs.com/docs/stable/transactions.html#automatic-retries) (也就是说所有语句都被单批提交)，使用 `COMMIT` 提交事务。

~~~ sql
> BEGIN; UPDATE products SET inventory = 100 WHERE = '8675309'; UPDATE products SET inventory = 100 WHERE = '8675310'; COMMIT;
~~~

## 查看更多

- [事务](https://www.cockroachlabs.com/docs/stable/transactions.html)
- [`开始（BEGIN）`](https://www.cockroachlabs.com/docs/stable/begin-transaction.html)
- [`释放保存点（RELEASE SAVEPOINT）`](https://www.cockroachlabs.com/docs/stable/release-savepoint.html)
- [`回滚（ROLLBACK）`](https://www.cockroachlabs.com/docs/stable/rollback-transaction.html)
- [`保存点（SAVEPOINT）`](https://www.cockroachlabs.com/docs/stable/savepoint.html)
