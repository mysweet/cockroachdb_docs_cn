# 使用内置的 SQL 客户端

CockroachDB 内置了一个客户端用来在交互式的 shell 或在命令行中直接运行 SQL 语句。

下面介绍了怎样通过 Cockroach SQL [命令](https://www.cockroachlabs.com/docs/stable/cockroach-commands.html)来使用这个客户端。

最后需要退出时，按 **CTRL + D**, **CTRL + C**, 或 `\q`。

## 简介

~~~ shell
# 运行交互式的 SQL 终端:
$ cockroach sql <flags>

# 从命令行运行 SQL:
$ cockroach sql --execute="<sql statement>;<sql statement>" --execute="<sql-statement>" <flags>
$ echo "<sql statement>;<sql statement>" | cockroach sql <flags>
$ cockroach sql <flags> < file-containing-statements.sql

# 查看帮助:
$ cockroach sql --help
~~~

## 指令

`sql` 命令支持 [普通用法](#普通用法) 和 [日志](#日志) 指令.

### 普通用法

- 打开一个交互式的 SQL 终端，带上适当的命令标识运行 `cockroach sql` 命令或直接使用 `--url` 标识输出命令使用的细节。
- 在命令行中使用 `--execute` 运行 SQL 语句。

| 命令行标识                | 描述                                       |
| :------------------- | ---------------------------------------- |
| `--certs-dir`        | [证书所在的目录](https://www.cockroachlabs.com/docs/stable/create-security-certificates.html)。如果在安全模式，这个目录必须包含有效的安全证书。<br><br>**环境变量：** `COCKROACH_CERTS_DIR`<br>**默认值：** `${HOME}/.cockroach-certs/` |
| `--database`<br>`-d` | 需要连接的数据库。<br><br>**环境变量：** `COCKROACH_DATABASE` |
| `--execute`<br>`-e`  | 不打开终端，直接在命令行运行 SQL 语句。这个标识可以使用多次，每一个实例都可以包含一条或多条语句并且用分好隔开。任何一个语句发生错误时，命令将退出并输出一个非零的状态码，其他命令也不再执行。命令执行的结果将在标准输出设备上输出 (查看 `--pretty` 格式化选项)。<br><br>更多示例请看[例子](https://www.cockroachlabs.com/docs/stable/use-the-built-in-sql-client.html#execute-sql-statements-from-the-command-line) 。 |
| `--host`             | 连接的服务器主机。这可以是集群里任何一个节点的地址。<br><br>**环境变量：** `COCKROACH_HOST`<br>**默认值：** `localhost` |
| `--insecure`         | 在非安全模式下运行。如果没有设置这个标识，`--certs-dir` 必须指向有效的证书。<br><br>**环境变量：** `COCKROACH_INSECURE`<br>**默认值：** `false` |
| `--port`<br>`-p`     | 连接的服务端口。<br><br>**环境变量：** `COCKROACH_PORT`<br>**默认值：** `26257` |
| `--pretty`           | 使用 ASCII 编码格式打印到标准输出的表格行，并禁用特殊字符的转义。<br><br>当使用 `--pretty=false` 来禁用这个功能或当标准输出不是一个终端时，表格将以制表符分隔的方式输出，特殊字符将被忽略。这样有利于输出内容被其他程序解析。<br><br>当输出设备是终端时**默认值：** `true` 否则为`false`。 |
| `--url`              | 连接的 URL。 如果使用了这个标识，就不要再设置其他的连接标识符。<br><br>对于不安全的连接，URL 的格式是这样的：<br>`--url=postgresql://<user>@<host>:<port>/<database>?sslmode=disable`<br><br>对于安全安全模式下的连接，URL 的格式是这样的：<br>`--url=postgresql://<user>@<host>:<port>/<database>`<br>在查询语句中带上如下的参数：<br>`sslcert=<path-to-client-crt>`<br>`sslkey=<path-to-client-key>`<br>`sslmode=verify-full`<br>`sslrootcert=<path-to-ca-crt>` <br><br>**环境变量：** `COCKROACH_URL` |
| `--user`<br>`-u`     | 连接的[用户](https://www.cockroachlabs.com/docs/stable/create-and-manage-users.html)。用户必须有操作的[权限](https://www.cockroachlabs.com/docs/stable/privileges.html)。<br><br>**环境变量：** `COCKROACH_USER`<br>**默认值：** `root` |

### 日志

默认情况下， `sql` 命令记录错误日志到 `stderr`。

If you need to troubleshoot this command's behavior, you can change its [logging behavior](debug-and-error-logs.html).

## SQL 终端命令

下面的命令可以用来与 SQL 终端交互：

| 命令                                   | 用法                                       |
| ------------------------------------ | ---------------------------------------- |
| `\q`<br>**CTRL + D**<br>**CTRL + C** | 退出 shell。                                |
| `\!`                                 | 运行外部的命令并且将结果输出到 `stdout`。查看[例子](#在 SQL 终端运行外部命令)。 |
| `\|`                                 | 将外部命令的输出作为 SQL 语句的参数。查看[例子](#在 SQL 终端运行外部命令)。 |
| `\set <option>`                      | 设置一个客户端参数可用。具体查看下面的参数列表。<br><br>使用不带参数的 `\set` 查看最新的设置。 |
| `\unset <option>`                    | 禁用客户端参数。具体查看下面的参数列表。                     |
| `\?`<br>`help`                       | 查看帮助。                                    |

| 客户端参数               | 描述                                       |
| ------------------- | ---------------------------------------- |
| `CHECK_SYNTAX`      | SQL 语句发送到服务器之前在客户端进行语法验证，这保证了用户输入的错字或错误不在交互时出现。<br><br>这个参数默认是可用的，要禁止它，运行 `\unset CHECK_SYNTAX`。 |
| `NORMALIZE_HISTORY` | 在 shell 历史中存储规范化语法，例如大写关键字、标准化间距、以及将多行语句作为单行语句进行会看。<br><br>这个参数默认是可用的。但是在 `CHECK_SYNTAX` 可用时才可用。需要禁止这个参数，运行 `\unset NORMALIZE_HISTORY`。 |
| `ERREXIT`           | z在遇到错误时退出 SQL 终端。<br><br>这个命令默认是不可用的。运行 `\set ERREXIT`开启。 |

## SQL 终端快捷方式

SQL 终端支持很多快捷方式，比如 **CTRL + R** 搜索历史。更多细节查看[快捷方式](https://github.com/chzyer/readline/blob/master/doc/shortcut.md)。

## 例子

### 开启 SQL 终端

在这个例子中，我们将在安全模式下链接终端。

~~~ shell
# 使用标准连接方式：
$ cockroach sql \
--certs-dir=certs \
--user=maxroach \
--host=12.345.67.89 \
--port=26257 \
--database=critterdb

# 使用 --url 标识连接:
$ cockroach sql \
--url="postgresql://maxroach@12.345.67.89:26257/critterdb?sslcert=certs/client.maxroach.crt&sslkey=certs/client.maxroach.key&sslmode=verify-full&sslroot=certs/ca.crt"
~~~

z在这个例子中，我们将在非安全模式下链接终端：


~~~ shell
# 使用标准链接方式：
$ cockroach sql --insecure \
--user=maxroach \
--host=12.345.67.89 \
--port=26257 \
--database=critterdb

# 使用 --url 标识连接:
$ cockroach sql \
--url="postgresql://maxroach@12.345.67.89:26257/critterdb?sslmode=disable"
~~~

### 用 SQL 终端运行 SQL 语句

这个例子假设我们已经打开了一个 SQL 终端：

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

### 在命令行里运行 SQL 语句：

在这个例子中，我们在命令行里使用 `--execute` 标识来运行参数。

~~~ shell
# 只有一个 --execute 标识:
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
# 带有多个 --execute 标识:
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

在这个例子中，我们在命令行里使用 `echo` 来运行命令。

~~~ shell
# 带有 echo 命令的参数:
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

### 优美的和不优美的输出

在这个例子中，我们演示优美的和不优美的列表和特殊字符的输出。当优美的输出方式被打开，列表将以 ASCII 编码的形式输出并且特殊字符不会被忽略且以可阅读的方式输出。当优美的输出方式被禁用，列表将以制表符分隔的方式被输出，且特殊字符将被忽略；但是这种方式容易被其他程序解析。

当标准输出是终端时，优美的输出方式将被默认启用，但是你也可以禁用它 `--pretty=false`：

~~~ shell
# 使用默认的优美输出：
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
# 禁用优美输出：
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

当输出到其他的程序或者文件时，默认值是相反的。优美输出默认不可用， 但是你也可以启用它 `--pretty`:

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
# 启用优美输出：
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

如果 `--pretty` 没有用 `--execute ` 标识，它将把每个表交互的结果输出应用到这个格式上。

### 从文件中运行 SQL 语句

在这个例子中，我们演示如何运行一个包含了 SQL 语句的文件：

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

### 在 SQL 终端运行外部命令

在这个例子中，我们使用  `\!` 去查看在创建表之前 CSV 文件的行和使用 `\|` 向表中插入行。

> **注意：<br>这个例子只在当 CSV 文件中的值都是数字时才有效。其他格式的值时，使用在线 CSV-to-SQL   转换器创建你自己的输入程序。**

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

在这个例子中我们创建了一个表并且使用 `\|` 以编程的形式插入值：

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

## 参考

- [其他的 Cockroach 命令](https://www.cockroachlabs.com/docs/stable/cockroach-commands.html)
- [SQL 参数](https://www.cockroachlabs.com/docs/stable/sql-statements.html)
- [学习 CockroachDB SQL](https://www.cockroachlabs.com/docs/stable/learn-cockroachdb-sql.html)
- [导入数据](https://www.cockroachlabs.com/docs/stable/import-data.html)
