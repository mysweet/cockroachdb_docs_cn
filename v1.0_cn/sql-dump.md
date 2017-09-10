# SQL 转储(导出)

`cockroach dump` [命令](cockroach-commands.md) 输出用于重建一张或多张表和它们的行的 SQL 语句(也被称为*转储*)。这条命令可被用于备份或导出集群中的每个数据库。输出经最小的调整，也应该适于导入到其他的关系数据库。

> 提示：
> 
> CockroachDB <a href="https://www.cockroachlabs.com/pricing/">企业授权</a>用户也能够使用<a href="backup.md"><code>BACKUP</code></a>备份他们的集群数据。

在 `cockroach dump` 被执行时：

- 表模式和数据在命令被启动的那一刻的形态被转储。任何在命令启动之后发生的变化将不会出现在转储中。
- 如果转储花费的时间多于复制为表设置的 [`ttlseconds`](configure-replication-zones.md) (默认为 24 小时)，转储可能失败。
- 读、写和模式变化可能发生在转储进行中，但不会影响转储的输出。

> 注：
> 
> 用户必须拥有目标表的 <code>SELECT</code> 权限。

## 概要

~~~ shell
# Dump the schema and data of specific tables to stdout:
$ cockroach dump <database> <table> <table...> <flags>

# Dump just the data of specific tables to stdout:
$ cockroach dump <database> <table> <table...> --dump-mode=data <other flags>

# Dump just the schema of specific tables to stdout:
$ cockroach dump <database> <table> <table...> --dump-mode=schema <other flags>

# Dump the schema and data of all tables in a database to stdout:
$ cockroach dump <database> <flags>

# Dump just the schema of all tables in a database to stdout:
$ cockroach dump <database> --dump-mode=schema <other flags>

# Dump just the data of all tables in a database to stdout:
$ cockroach dump <database> --dump-mode=data <other flags>

# Dump to a file:
$ cockroach dump <database> <table> <flags> > dump-file.sql

# View help:
$ cockroach dump --help
~~~

## 标志

`dump` 命令支持下列[通用](#通用)和[日志](#日志)标志。

### 通用

标志 | 描述
-----|------------
`--as-of` | 转储在指定[时间戳](timestamp.md)的表模式和/或数据。示例见这个[例子](#dump-table-data-as-of-a-specific-time)。<br><br>注意，历史数据仅在垃圾收集窗口内可用，这是由表的复制设置 [`ttlseconds`](configure-replication-zones.md) 决定的（默认为 24 小时）。如果这个时间戳早于窗口，转储将失败。<br><br>**默认值:** 当前时间
`--certs-dir` | [证书目录](create-security-certificates.md)的路径。如果运行在安全模式，目录必须包含有效证书。<br><br>**环境变量:** `COCKROACH_CERTS_DIR`<br>**默认值:** `${HOME}/.cockroach-certs/`
`--dump-mode` | 是否转储表模式、表数据，或者都转储。<br><br>只转储表模式，设为 `schema`。只转储表数据，设为 `data`。转储表模式和数据，不设置这个标志或者设置为 `both`。<br><br>**默认值:** `both`
`--host` | 要连接的服务器主机。可以是集群中任何节点的地址。<br><br>**环境变量:** `COCKROACH_HOST`<br>**默认值:** `localhost`
`--insecure` | 运行在非安全模式。如果这个标志没有被设置， `--certs-dir` 标志必须指向有效的证书。<br><br>**环境变量:** `COCKROACH_INSECURE`<br>**默认值:** `false`
`--port`<br>`-p` | 要连接的服务器端口。 <br><br>**环境变量:** `COCKROACH_PORT`<br>**默认值:** `26257`
`--url` | 连接 URL。如果你使用这个标志，不要设置任何其他连接标志。<br><br>对于非安全的连接，URL 格式是： <br>`--url=postgresql://<user>@<host>:<port>/<database>?sslmode=disable`<br><br>对于安全的连接，URL 格式是：<br>`--url=postgresql://<user>@<host>:<port>/<database>`<br>以及下列在查询串中的参数：<br>`sslcert=<path-to-client-crt>`<br>`sslkey=<path-to-client-key>`<br>`sslmode=verify-full`<br>`sslrootcert=<path-to-ca-crt>` <br><br>**环境变量:** `COCKROACH_URL`
`--user`<br>`-u` | 执行 `dump` 命令的[用户](create-and-manage-users.md)。用户必须拥有对目标数据库的 `SELECT` 权限。<br><br>**默认值:** `root`

### 日志

`dump` 命令默认将错误输出到 `stderr`。

如果你需要对这条命令的行为排错，你可以改变它的[日志行为](debug-and-error-logs.md)。

## 例子

> 注：
> 
> 这些例子使用我们的startrek 数据库，你可以通过<a href="generate-cockroachdb-resources.md#generate-example-data"><code>cockroach gen</code></a> 命令加入集群。而且，例子假设 <code>maxroach</code> 用户已经在所有目标表 <a href="grant.md">被授予</a> <code>SELECT</code> 权限。 

### 转储表的模式和数据

~~~ shell
$ cockroach dump startrek episodes --insecure --user=maxroach > backup.sql
~~~

~~~ shell
$ cat backup.sql
~~~

~~~
CREATE TABLE episodes (
    id INT NOT NULL,
    season INT NULL,
    num INT NULL,
    title STRING NULL,
    stardate DECIMAL NULL,
    CONSTRAINT "primary" PRIMARY KEY (id),
    FAMILY "primary" (id, season, num),
    FAMILY fam_1_title (title),
    FAMILY fam_2_stardate (stardate)
);

INSERT INTO episodes (id, season, num, title, stardate) VALUES
    (1, 1, 1, 'The Man Trap', 1531.1),
    (2, 1, 2, 'Charlie X', 1533.6),
    (3, 1, 3, 'Where No Man Has Gone Before', 1312.4),
    (4, 1, 4, 'The Naked Time', 1704.2),
    (5, 1, 5, 'The Enemy Within', 1672.1),
    (6, 1, 6, e'Mudd\'s Women', 1329.8),
    (7, 1, 7, 'What Are Little Girls Made Of?', 2712.4),
    (8, 1, 8, 'Miri', 2713.5),
    (9, 1, 9, 'Dagger of the Mind', 2715.1),
    (10, 1, 10, 'The Corbomite Maneuver', 1512.2),
    ...
~~~

### 仅转储表的模式

~~~ shell
$ cockroach dump startrek episodes --insecure --user=maxroach --dump-mode=schema > backup.sql
~~~

~~~ shell
$ cat backup.sql
~~~

~~~
CREATE TABLE episodes (
    id INT NOT NULL,
    season INT NULL,
    num INT NULL,
    title STRING NULL,
    stardate DECIMAL NULL,
    CONSTRAINT "primary" PRIMARY KEY (id),
    FAMILY "primary" (id, season, num),
    FAMILY fam_1_title (title),
    FAMILY fam_2_stardate (stardate)
);
~~~

### 仅转储表的数据

~~~ shell
$ cockroach dump startrek episodes --insecure --user=maxroach --dump-mode=data > backup.sql
~~~

~~~ shell
$ cat backup.sql
~~~

~~~
INSERT INTO episodes (id, season, num, title, stardate) VALUES
    (1, 1, 1, 'The Man Trap', 1531.1),
    (2, 1, 2, 'Charlie X', 1533.6),
    (3, 1, 3, 'Where No Man Has Gone Before', 1312.4),
    (4, 1, 4, 'The Naked Time', 1704.2),
    (5, 1, 5, 'The Enemy Within', 1672.1),
    (6, 1, 6, e'Mudd\'s Women', 1329.8),
    (7, 1, 7, 'What Are Little Girls Made Of?', 2712.4),
    (8, 1, 8, 'Miri', 2713.5),
    (9, 1, 9, 'Dagger of the Mind', 2715.1),
    (10, 1, 10, 'The Corbomite Maneuver', 1512.2),
    ...
~~~

### 转储数据库中的所有表

~~~ shell
$ cockroach dump startrek --insecure --user=maxroach > backup.sql
~~~

~~~ shell
$ cat backup.sql
~~~

~~~
CREATE TABLE episodes (
    id INT NOT NULL,
    season INT NULL,
    num INT NULL,
    title STRING NULL,
    stardate DECIMAL NULL,
    CONSTRAINT "primary" PRIMARY KEY (id),
    FAMILY "primary" (id, season, num),
    FAMILY fam_1_title (title),
    FAMILY fam_2_stardate (stardate)
);

CREATE TABLE quotes (
    quote STRING NULL,
    characters STRING NULL,
    stardate DECIMAL NULL,
    episode INT NULL,
    INDEX quotes_episode_idx (episode),
    FAMILY "primary" (quote, rowid),
    FAMILY fam_1_characters (characters),
    FAMILY fam_2_stardate (stardate),
    FAMILY fam_3_episode (episode)
);

INSERT INTO episodes (id, season, num, title, stardate) VALUES
    (1, 1, 1, 'The Man Trap', 1531.1),
    (2, 1, 2, 'Charlie X', 1533.6),
    (3, 1, 3, 'Where No Man Has Gone Before', 1312.4),
    (4, 1, 4, 'The Naked Time', 1704.2),
    (5, 1, 5, 'The Enemy Within', 1672.1),
    (6, 1, 6, e'Mudd\'s Women', 1329.8),
    (7, 1, 7, 'What Are Little Girls Made Of?', 2712.4),
    (8, 1, 8, 'Miri', 2713.5),
    (9, 1, 9, 'Dagger of the Mind', 2715.1),
    (10, 1, 10, 'The Corbomite Maneuver', 1512.2),
    ...

INSERT INTO quotes (quote, characters, stardate, episode) VALUES
    ('"... freedom ... is a worship word..." "It is our worship word too."', 'Cloud William and Kirk', NULL, 52),
    ('"Beauty is transitory." "Beauty survives."', 'Spock and Kirk', NULL, 72),
    ('"Can you imagine how life could be improved if we could do away with jealousy, greed, hate ..." "It can also be improved by eliminating love, tenderness, sentiment -- the other side of the coin"', 'Dr. Roger Corby and Kirk', 2712.4, 7),
    ...
~~~

### 转储失败(用户没有 `SELECT` 权限)

在这个例子中，`dump` 命令对于在 `episodes` 表没有 `SELECT` 权限的用户失败。

~~~ shell
$ cockroach dump startrek episodes --insecure --user=leslieroach > backup.sql
~~~

~~~ shell
Error: pq: user leslieroach has no privileges on table episodes
Failed running "dump"
~~~

### 从备份文件恢复一张表

在这个例子中，用户拥有 `startrek` 数据库的 `CREATE` 权限，基于由 `dump` 命令创建的文件，使用 [`cockroach sql`](use-the-built-in-sql-client.md) 命令重建一张表。

~~~ shell
$ cat backup.sql
~~~

~~~
CREATE TABLE quotes (
    quote STRING NULL,
    characters STRING NULL,
    stardate DECIMAL NULL,
    episode INT NULL,
    INDEX quotes_episode_idx (episode),
    FAMILY "primary" (quote, rowid),
    FAMILY fam_1_characters (characters),
    FAMILY fam_2_stardate (stardate),
    FAMILY fam_3_episode (episode)
);

INSERT INTO quotes (quote, characters, stardate, episode) VALUES
    ('"... freedom ... is a worship word..." "It is our worship word too."', 'Cloud William and Kirk', NULL, 52),
    ('"Beauty is transitory." "Beauty survives."', 'Spock and Kirk', NULL, 72),
    ('"Can you imagine how life could be improved if we could do away with jealousy, greed, hate ..." "It can also be improved by eliminating love, tenderness, sentiment -- the other side of the coin"', 'Dr. Roger Corby and Kirk', 2712.4, 7),
    ...
~~~

~~~ shell
$ cockroach sql --insecure --database=startrek --user=maxroach < backup.sql
~~~

~~~ shell
CREATE TABLE
INSERT 100
INSERT 100
~~~

### 在特定时间转储表数据

在这个例子中，我们假设在 `2017-03-07 19:55:00` 之前和之后有几个插入到表中。

首先，我们使用内建的 SQL 客户端查看当前时间的表：

~~~ shell
$ cockroach sql --insecure --execute="SELECT * FROM db1.dump_test"
~~~

~~~
+--------------------+------+
|         id         | name |
+--------------------+------+
| 225594758537183233 | a    |
| 225594758537248769 | b    |
| 225594758537281537 | c    |
| 225594758537314305 | d    |
| 225594758537347073 | e    |
| 225594758537379841 | f    |
| 225594758537412609 | g    |
| 225594758537445377 | h    |
| 225594991654174721 | i    |
| 225594991654240257 | j    |
| 225594991654273025 | k    |
| 225594991654305793 | l    |
| 225594991654338561 | m    |
| 225594991654371329 | n    |
| 225594991654404097 | o    |
| 225594991654436865 | p    |
+--------------------+------+
(16 rows)
~~~

接下来，我们使用[时间旅行查询](select.md#select-historical-data-time-travel)查看在 `2017-03-07 19:55:00` 表的内容：

~~~ shell
$ cockroach sql --insecure --execute="SELECT * FROM db1.dump_test AS OF SYSTEM TIME '2017-03-07 19:55:00'"
~~~

~~~
+--------------------+------+
|         id         | name |
+--------------------+------+
| 225594758537183233 | a    |
| 225594758537248769 | b    |
| 225594758537281537 | c    |
| 225594758537314305 | d    |
| 225594758537347073 | e    |
| 225594758537379841 | f    |
| 225594758537412609 | g    |
| 225594758537445377 | h    |
+--------------------+------+
(8 rows)
~~~

最后，我们使用 `cockroach dump` 和 `--as-of` 标志转储在 `2017-03-07 19:55:00` 表的内容。

~~~ shell
$ cockroach dump db1 dump_test --insecure --dump-mode=data --as-of='2017-03-07 19:55:00'
~~~

~~~
INSERT INTO dump_test (id, name) VALUES
    (225594758537183233, 'a'),
    (225594758537248769, 'b'),
    (225594758537281537, 'c'),
    (225594758537314305, 'd'),
    (225594758537347073, 'e'),
    (225594758537379841, 'f'),
    (225594758537412609, 'g'),
    (225594758537445377, 'h');
~~~

就像你看到的，转储的结果和早先时间旅行查询是一致的。

## 另见

- [导入数据](import-data.md)
- [使用内建的 SQL 客户端](use-the-built-in-sql-client.md)
- [其他 Cockroach 命令](cockroach-commands.md)
