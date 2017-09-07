# 安装客户端驱动程序

CockroachDB 支持 PostgreSQL 连线协议，所以你可以使用任何可用的 PostgreSQL 客户端驱动程。我们已经测试过并能够推荐下列驱动程序。

- 使用这些驱动程序的代码样本，见[使用 CockroachDB 构建应用](build-an-app-with-cockroachdb.md)教程。
- 如果你用这些驱动程序有问题，或者想就其他驱动程序分享反馈，请[联系](contribute-to-cockroachdb.md)。

语言 | 推荐驱动程序
---------|--------
Go | [pq](https://godoc.org/github.com/lib/pq)
Python | [psycopg2](http://initd.org/psycopg/)
Ruby | [pg](https://rubygems.org/gems/pg)
Java | [jdbc](https://jdbc.postgresql.org)
Node.js | [pg](https://www.npmjs.com/package/pg) 
C | [libpq](http://www.postgresql.org/docs/9.5/static/libpq.html)
C++ | [libpqxx](http://pqxx.org/development/libpqxx/)
Clojure | [java.jdbc](http://clojure-doc.org/articles/ecosystem/java_jdbc/home.html)
PHP | [php-pgsql](http://php.net/manual/en/book.pgsql.php)
Rust | [postgres](https://crates.io/crates/postgres/)