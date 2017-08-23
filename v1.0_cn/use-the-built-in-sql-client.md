# 使用内建的 SQL 客户端

CockroachDB 内建了一个客户端用，在交互式的 shell 或直接在命令行中运行 SQL 语句。

下面介绍了运行 Cockroach SQL [命令](cockroach-commands.md)来使用这个客户端。

要退出交互式的 shell，按 **CTRL + D**, **CTRL + C**, 或 `\q`。

## 简介

~~~ shell
# 运行交互式的 SQL shell:
$ cockroach sql <flags>

# 从命令行运行 SQL:
$ cockroach sql --execute="<sql statement>;<sql statement>" --execute="<sql-statement>" <flags>
$ echo "<sql statement>;<sql statement>" | cockroach sql <flags>
$ cockroach sql <flags> < file-containing-statements.sql

# 查看帮助:
$ cockroach sql --help
~~~

## 标志

`sql` 命令支持 [普通用法](#通用) 和 [日志](#日志) 标志。

### 通用

- 打开一个交互式的 SQL shell，带上适当的命令标志运行 `cockroach sql` 命令或仅使用 `--url` 标志，它将包含连接的细节。

- 在命令行用 `--execute` 标志运行 SQL 语句。

| 标志                | 描述                                       |
| :------------------- | ---------------------------------------- |
| `--certs-dir`        | [证书目录](create-security-certificates.md)所在的路径。如果以安全模式运行，这个目录必须包含有效的证书。<br><br>**环境变量：** `COCKROACH_CERTS_DIR`<br>**默认值：** `${HOME}/.cockroach-certs/` |
| `--database`<br>`-d` | 需要连接的数据库。<br><br>**环境变量：** `COCKROACH_DATABASE` |
| `--execute`<br>`-e`  | 不打开 shell，直接在命令行运行 SQL 语句。这个标志可以使用多次，每一个实例都可以包含一条或多条语句并且用分号隔开。任何一条语句发生错误时，命令将退出并输出一个非零的状态码，其他命令不再执行。命令执行的结果将在标准输出设备上输出 (查看 `--pretty` 格式化选项)。<br><br>更多示例请看[例子](use-the-built-in-sql-client.md#execute-sql-statements-from-the-command-line) 。 |
| `--host`             | 连接的服务器主机，可以是集群里任何一个节点的地址。<br><br>**环境变量：** `COCKROACH_HOST`<br>**默认值：** `localhost` |
| `--insecure`         | 以非安全模式运行。如果没有设置这个标志，`--certs-dir` 必须指向有效的证书。<br><br>**环境变量：** `COCKROACH_INSECURE`<br>**默认值：** `false` |
| `--port`<br>`-p`     | 连接的服务器端口。<br><br>**环境变量：** `COCKROACH_PORT`<br>**默认值：** `26257` |
| `--pretty`           | 使用 ASCII 艺术将打印到标准输出的表行格式化，并禁用特殊字符的转义。<br><br>当使用 `--pretty=false` 来禁用这个功能或当标准输出不是终端时，表行将以制表符分隔的值输出，特殊字符被转义。这样易于输出内容被其他程序解析。<br><br>**默认值：** 当输出设备是终端时为 `true` 否则为 `false`。 |
| `--url`              | 连接的 URL。 如果使用了这个标志，就不要再设置其他的连接标识符。<br><br>对于不安全的连接，URL 的格式是这样的：<br>`--url=postgresql://<user>@<host>:<port>/<database>?sslmode=disable`<br><br>对于安全安全模式下的连接，URL 的格式是：<br>`--url=postgresql://<user>@<host>:<port>/<database>`<br>在查询语句中带上如下的参数：<br>`sslcert=<path-to-client-crt>`<br>`sslkey=<path-to-client-key>`<br>`sslmode=verify-full`<br>`sslrootcert=<path-to-ca-crt>` <br><br>**环境变量：** `COCKROACH_URL` |
| `--user`<br>`-u`     | 连接的[用户](create-and-manage-users.md)。用户必须有操作的[权限](privileges.md)。<br><br>**环境变量：** `COCKROACH_USER`<br>**默认值：** `root` |

### 日志

默认情况下， `sql` 命令记录错误日志到 `stderr`。

如果你需要对这条命令的行为拍错，可以改变其[日志](debug-and-error-logs.md)。

## SQL 终端命令

下面的命令可以在交互式 SQL shell 中使用：

| 命令                                   | 用法                                       |
| ------------------------------------ | ---------------------------------------- |
| `\q`<br>**CTRL + D**<br>**CTRL + C** | 退出 shell。                                |
| `\!`                                 | 运行外部命令并且将结果输出到 `stdout`。查看下面的[例子](#在 SQL shell 中运行外部命令)。 |
| `\|`                                 | 将外部命令的输出作为 SQL 语句的参数。查看[例子](#在 SQL shell 中运行外部命令)。 |
| `\set <option>`                      | 设置一个客户端选项可用。具体查看下面的参数列表。<br><br>使用不带参数的 `\set` 查看最新的设置。 |
| `\unset <option>`                    | 禁用客户端选项。具体查看下面的参数列表。                     |
| `\?`<br>`help`                       | 在 shell 中查看帮助。                                    |

| 客户端选项            | 描述                                       |
| ------------------- | ---------------------------------------- |
| `CHECK_SYNTAX`      | SQL 语句发送到服务器之前在客户端进行语法验证，这保证了用户输入的错字或错误不会退出在交互 shell 中之前启动而且正在运行的事务。<br><br>这个选项默认是打开的，要关闭它，运行 `\unset CHECK_SYNTAX`。 |
| `NORMALIZE_HISTORY` | 在 shell 历史中存储规范化的语法，例如大写关键字、标准化间距、以及将多行语句作为单行语句再调用。<br><br>这个参数默认是打开的。但是在 `CHECK_SYNTAX` 打开时才打开。要禁用这个选项，运行 `\unset NORMALIZE_HISTORY`。 |
| `ERREXIT`           | 在遇到错误时退出 SQL shell。<br><br>这个命令默认是关闭。运行 `\set ERREXIT`开启。 |

## SQL shell 快捷方式

SQL shell 支持很多快捷方式，比如 **CTRL + R** 搜索历史。更多细节查看[快捷方式](https://github.com/chzyer/readline/blob/master/doc/shortcut.md)。

## 例子

### 启动 SQL shell

在下面的例子中，我们将连接 SQL shell 到一个安全集群。

~~~ shell
# 使用标准连接标志：
$ cockroach sql \
--certs-dir=certs \
--user=maxroach \
--host=12.345.67.89 \
--port=26257 \
--database=critterdb

# 使用 --url 标志连接:
$ cockroach sql \
--url="postgresql://maxroach@12.345.67.89:26257/critterdb?sslcert=certs/client.maxroach.crt&sslkey=certs/client.maxroach.key&sslmode=verify-full&sslroot=certs/ca.crt"
~~~

在下面的例子中，我们将连接 SQL shell 到一个安全集群。

~~~ shell
# 使用标准连接标志：
$ cockroach sql --insecure \
--user=maxroach \
--host=12.345.67.89 \
--port=26257 \
--database=critterdb

# 使用 --url 标志连接:
$ cockroach sql \
--url="postgresql://maxroach@12.345.67.89:26257/critterdb?sslmode=disable"
~~~

### 在 SQL shell 中运行外部命令

这个例子假设我们已经启动了一个 SQL shell（见上面的例子）。

~~~ sql
> CREATE TABLE animals (id SERIAL PRIMARY KEY, name STRING);

> INSERT INTO animals (name) VALUES ('bobcat'), ('🐢 '), ('barn owl');

> SELECT * FROM animals;
~~~
~~~
+--------------------+----------+
|         id         |   name   |
+--------------------+----------+
| 148899952591994881 | bobcat   |
| 148899952592060417 | 🐢        |
| 148899952592093185 | barn owl |
+--------------------+----------+
~~~

### 在命令行运行 SQL 语句

在这些例子中，我们在命令行里使用 `--execute` 标志运行来自命令行的语句。

~~~ shell
# 只有一个 --execute 标志:
$ cockroach sql --insecure \
--execute="CREATE TABLE roaches (name STRING, country STRING); INSERT INTO roaches VALUES ('American Cockroach', 'United States'), ('Brownbanded Cockroach', 'United States')" \
--user=maxroach \
--host=12.345.67.89 \
--port=26257 \
--database=critterdb
~~~

~~~
CREATE TABLE
INSERT 2
~~~

~~~ shell
# 带有多个 --execute 标志:
$ cockroach sql --insecure \
--execute="CREATE TABLE roaches (name STRING, country STRING)" \
--execute="INSERT INTO roaches VALUES ('American Cockroach', 'United States'), ('Brownbanded Cockroach', 'United States')" \
--user=maxroach \
--host=12.345.67.89 \
--port=26257 \
--database=critterdb
~~~

~~~
CREATE TABLE
INSERT 2
~~~

在这个例子中，我们使用 `echo` 命令运行来自命令行的语句。

~~~ shell
# 带有 echo 命令的语句:
$ echo "SHOW TABLES; SELECT * FROM roaches;" | cockroach sql --insecure --user=maxroach --host=12.345.67.89 --port=26257 --database=critterdb
+----------+
|  Table   |
+----------+
| roaches  |
+----------+
+-----------------------+---------------+
|         name          |    country    |
+-----------------------+---------------+
| American Cockroach    | United States |
| Brownbanded Cockroach | United States |
+-----------------------+---------------+
~~~

### 美观和不美观的输出

在这些例子中，我们以美观和不美观方式打印了表和特殊字符的输出。当美观的输出方式被打开，表将以 ASCII 艺术的形式输出，而特殊字符不会被转义，易于被人接受。当美观的输出方式被禁用，表行将以制表符分隔的值被输出，而特殊字符将被转义；这种方式易于被其他程序解析。

当标准输出是终端时，美观的输出方式将被默认启用，但是你可以明确禁用它 `--pretty=false`：

~~~ shell
# 使用默认的美观输出：
$ cockroach sql --insecure \
--pretty \
--execute="SELECT '🐥' AS chick, '🐢' AS turtle" \
--user=maxroach \
--host=12.345.67.89 \
--port=26257 \
--database=critterdb
~~~

~~~
+-------+--------+
| chick | turtle |
+-------+--------+
| 🐥    | 🐢     |
+-------+--------+
~~~

~~~ shell
# 禁用美观输出：
$ cockroach sql --insecure \
--pretty=false \
--execute="SELECT '🐥' AS chick, '🐢' AS turtle" \
--user=maxroach \
--host=12.345.67.89 \
--port=26257 \
--database=critterdb
~~~

~~~
1 row
chick turtle
"\U0001f425"  "\U0001f422"
~~~

当输出到其他的程序或者文件时，默认值是相反的。美观输出默认不可用， 但是你可以明确启用它 `--pretty`:

~~~ shell
# 使用默认的非优美输出:
$ cockroach sql --insecure \
--execute="SELECT '🐥' AS chick, '🐢' AS turtle" > out.txt \
--user=maxroach \
--host=12.345.67.89 \
--port=26257 \
--database=critterdb

$ cat out.txt
~~~

~~~
1 row
chick turtle
"\U0001f425"  "\U0001f422"
~~~

~~~ shell
# 明确启用美观输出：
$ cockroach sql --insecure \
--pretty \
--execute="SELECT '🐥' AS chick, '🐢' AS turtle" > out.txt \
--user=maxroach \
--host=12.345.67.89 \
--port=26257 \
--database=critterdb

$ cat out.txt
~~~

~~~
+-------+--------+
| chick | turtle |
+-------+--------+
| 🐥    | 🐢     |
+-------+--------+
~~~

如果 `--pretty` 没有用 `--execute ` 标志，它将应用于交互的  SQL shell 中每个表结果的输出格式上。

### 从文件中运行 SQL 语句

在这个例子中，我们显示并运行一个包含 SQL 语句的文件：

~~~ shell
$ cat statements.sql
~~~

~~~
CREATE TABLE roaches (name STRING, country STRING);
INSERT INTO roaches VALUES ('American Cockroach', 'United States'), ('Brownbanded Cockroach', 'United States');
~~~

~~~ shell
$ cockroach sql --insecure \
--user=maxroach \
--host=12.345.67.89 \
--port=26257 \
--database=critterdb \
< statements.sql
~~~

~~~
CREATE TABLE
INSERT 2
~~~

### 在 SQL shell 中运行外部命令

在这个例子中，我们使用  `\!` 在创建表之前查看 CSV 文件中的行，并使用 `\|` 向表中插入这些行。

> **注意：**
>
>这个例子只在当 CSV 文件中的值都是数字时才有效。对于其他格式的值，使用一个在线的 CSV-to-SQL 转换器，或创建你自己的输入程序。

~~~ sql
> \! cat test.csv
~~~

~~~
12, 13, 14
10, 20, 30
~~~

~~~ sql
> CREATE TABLE csv (x INT, y INT, z INT);

> \| IFS=","; while read a b c; do echo "insert into csv values ($a, $b, $c);"; done < test.csv;

> SELECT * FROM csv;
~~~

~~~
+----+----+----+
| x  | y  | z  |
+----+----+----+
| 12 | 13 | 14 |
| 10 | 20 | 30 |
+----+----+----+
~~~

在这个例子中，我们创建了一张表并随后使用 `\|` 以编程的形式插入值：

~~~ sql
> CREATE TABLE for_loop (x INT);

> \| for ((i=0;i<10;++i)); do echo "INSERT INTO for_loop VALUES ($i);"; done

> SELECT * FROM for_loop;
~~~

~~~
+---+
| x |
+---+
| 0 |
| 1 |
| 2 |
| 3 |
| 4 |
| 5 |
| 6 |
| 7 |
| 8 |
| 9 |
+---+
~~~

## 另见

- [其他 Cockroach 命令](cockroach-commands.md)
- [SQL 参数](sql-statements.md)
- [学习 CockroachDB SQL](learn-cockroachdb-sql.md)
- [导入数据](import-data.md)
