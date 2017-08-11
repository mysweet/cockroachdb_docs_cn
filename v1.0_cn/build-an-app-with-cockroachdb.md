# 用 CockroachDB 构建一个 App

这一章节将会向你介绍 CockroachDB 如何使用兼容性的 PostgreSQL 客户端驱动程序和 ORMs(我们已经测试过，可推荐使用) 来创建一个 App。

如果你在使用所推荐的 ORM 时有问题，请 [开任务单](https://github.com/cockroachdb/cockroach/issues/new) 并详细说明。如果你想让我们支持额外的 ORMs，请 [写在这里](https://forum.cockroachlabs.com/t/orm-compatibility/49) 让我们知道。

| App 语言  | 特定的驱动程序                                  | 特定的 ORM                                  |
| ------- | ---------------------------------------- | ---------------------------------------- |
| Go      | [pq](build-a-go-app-with-cockroachdb.html) | [GORM](build-a-go-app-with-cockroachdb-gorm.html) |
| Python  | [psycopg2](build-a-python-app-with-cockroachdb.html) | [SQLAlchemy](build-a-python-app-with-cockroachdb-sqlalchemy.html) |
| Ruby    | [pg](build-a-ruby-app-with-cockroachdb.html) | [ActiveRecord](build-a-ruby-app-with-cockroachdb-activerecord.html) |
| Java    | [jdbc](build-a-java-app-with-cockroachdb.html) | [Hibernate](build-a-java-app-with-cockroachdb-hibernate.html) |
| Node.js | [pg](build-a-nodejs-app-with-cockroachdb.html) | [Sequelize](build-a-nodejs-app-with-cockroachdb-sequelize.html) |
| C++     | [libpqxx](build-a-c++-app-with-cockroachdb.html) | No ORMs tested                           |
| Clojure | [java.jdbc](build-a-clojure-app-with-cockroachdb.html) | No ORMs tested                           |
| PHP     | [php-pgsql](build-a-php-app-with-cockroachdb.html) | No ORMs tested                           |
| Rust    | [postgres](build-a-rust-app-with-cockroachdb.html) | No ORMs tested                           |

