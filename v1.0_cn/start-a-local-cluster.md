# 搭建一个本地集群

搭建并运行一个具有多个节点的本地 CockroachDB 集群，且每一个节点都监听不同的端口。

## 搭建一个非可靠的本地集群

当安装完 CockroachDB 后，搭建一个非可靠的多节点本地集群将非常容易。

> **注意：**<br>在一台主机上运行多个节点常被用来测试 CockroachDB，但实际部署时不推荐这样做。如何在生产环境中运行一个分布式物理机集群，详见[手动部署](https://www.cockroachlabs.com/docs/stable/manual-deployment.html)，[云端部署](https://www.cockroachlabs.com/docs/stable/cloud-deployment.html)或[业务流程](https://www.cockroachlabs.com/docs/stable/orchestration.html)。

### 目录
* [开始之前](#开始之前)
* [第一步：部署第一个节点](#第一步：部署第一个节点)
* [第二步：向集群中添加节点](#第二步：向集群中添加节点)
* [第三步：集群测试](#第三步：集群测试)
* [第四步：集群监控](第四步：集群监控)
* [第五步：停止集群](第五步：停止集群)
* [第六步：重启集群](第六步：重启集群)

## 开始之前

确定你已经安装了 [CockroachDB](https://www.cockroachlabs.com/docs/stable/install-cockroachdb.html)。

## 第一步：部署第一个节点

```sh
$ cockroach start --insecure \
--host=localhost
```

```sh
CockroachDB node starting at {{page.release_info.start_time}}
build:      CCL {{page.release_info.version}} @ {{page.release_info.build_time}}
admin:      http://localhost:8080
sql:        postgresql://root@localhost:26257?sslmode=disable
logs:       cockroach-data/logs
store[0]:   path=cockroach-data
status:     initialized new cluster
clusterID:  {dab8130a-d20b-4753-85ba-14d8956a294c}
nodeID:     1
```

以上命令以非可靠模式搭建了一个节点, 且接受大部分 [`cockroach 部署`](https://www.cockroachlabs.com/docs/stable/start-a-node.html) 默认设置。

- `--insecure` 设置不加密消息。
- 因为这只是一个本地集群，`--host=localhost` 告诉该节点只监听 `localhost`, 并且都使用默认端口 (`26257`) 进行对内和客户端通信以及处理来自管理接口(`8080`)的请求。
- 节点的数据保存在 `cockroach-data` 目录。
- [标准输出](https://www.cockroachlabs.com/docs/stable/start-a-node.html#standard-output) 提供了 CockroachDB 的使用帮助，如版本信息和 SQL 客户端的 URL。

> **提示：**<br>每个节点的缓存空间都是系统空闲存储空间的 25%。当每台主机上都运行一个节点时，这样的设定是合理的。但当每台主机都运行多个节点，尤其是在可靠模式下测试时，这种设定将导致内存溢出错误。为了避免这种错误，可以在 `start` 命令中通过 `--cache` 来限制每个节点的缓存区大小。

## 第二步：向集群中添加节点

来到这一节，说明你的集群已经运行成功。在已有的一个节点上，你可以连接一个 SQL 客户端并开始搭建你的数据库系统。然而在实际生产中，你将会希望有三个或更多的节点同时运行来发挥 CockroachDB 的[自动化自我复制](https://www.cockroachlabs.com/docs/stable/demo-data-replication.html), [负载均衡](demo-automatic-rebalancing.html), 和[容错性](demo-fault-tolerance-and-recovery.html)能力。这一节将帮助你模拟一次真正的部署。

打开一个新的终端程序，添加第二个节点：

```sh
$ cockroach start --insecure \
--store=node2 \
--host=localhost \
--port=26258 \
--http-port=8081 \
--join=localhost:26257
```

再打开一个终端，添加第三个节点：

```shell
$ cockroach start --insecure \
--store=node3 \
--host=localhost \
--port=26259 \
--http-port=8082 \
--join=localhost:26257
```

这些命令中主要的不同就是使用了 `--join` 来进行集群和新节点之间的连接, 对于指定地址和端口的第一个节点是 `localhost:26257`。当你要在同一台主机上运行所有节点时，你还需要通过 `--store`，`--port` 和 `--http-port` 设置未被其他节点使用的地址和端口，在实际生产部署中，每个节点在不同的机器上时，这样的默认设置就足够了。

## 第三步：集群测试

现在你已经将集群规模扩展至三个节点，你可以使用任意一个节点作为集群的 SQL 网关。为了证明这一点，打开一个新的终端程序并将[内置 SQL 客户端](https://www.cockroachlabs.com/docs/stable/use-the-built-in-sql-client.html)连接至节点 1 :

> **注意：**<br>SQL 客户端内置于 cockroach 二进制文件中，所以不需要额外的支持。

```sh
$ cockroach sql --insecure
```

运行一些基本的 SQL 语句[CockroachDB SQL 语句](https://www.cockroachlabs.com/docs/stable/learn-cockroachdb-sql.html):

``` sql
> CREATE DATABASE bank;
```

``` sql
> CREATE TABLE bank.accounts (id INT PRIMARY KEY, balance DECIMAL);
```

``` sql
> INSERT INTO bank.accounts VALUES (1, 1000.50);
```

``` sql
> SELECT * FROM bank.accounts;
```

```
+----+---------+
| id | balance |
+----+---------+
|  1 |  1000.5 |
+----+---------+
(1 row)
```

退出节点 1 上的 SQL 终端：

``` sql
> \q
```

接下来连接节点 2 的 SQL 终端，此时应说明节点的非默认端口:

```sh
$ cockroach sql --insecure --port=26258
```

> **注意：**<br>在实际部署中，所有的节点都使用 `26257` 作为默认端口，因此你无需设置 `--port`。

现在可以运行 `SELECT` 来进行搜索了:

``` sql
> SELECT * FROM bank.accounts;
```

```
+----+---------+
| id | balance |
+----+---------+
|  1 |  1000.5 |
+----+---------+
(1 row)
```

可以看到，节点 1 和节点 2 上的输出结果是一样的。就像所看到的一样，节点 1 和节点 2 都可以作为 SQL 网关。

退出节点 2 上的 SQL 终端:

``` sql
> \q
```

## 第四步：集群监控

访问集群的[管理窗口](https://www.cockroachlabs.com/docs/stable/explore-the-admin-ui.html)通过地址`http://localhost:8080`或节点启动时控制台输出 `admin` 域中指定的地址:

![](https://www.cockroachlabs.com/docs/images/admin_ui.png)

前面提到，CockroachDB 能在后台自动复制数据。为了验证前一步骤中是否成功写入数据，将页面滚动至 **Replicas per Node** 图标并且将鼠标悬停在折线上:

![](https://www.cockroachlabs.com/docs/images/admin_ui_replicas.png)

因为每个节点的副本都是相同的，以上图表显示数据在集群上被默认复制了三次。

> **提示：**<br>为了更好地了解 CockroachDB 是怎样处理复制、平衡数据、容错和从故障中恢复的，请查看[复制数据](https://www.cockroachlabs.com/docs/stable/demo-data-replication.html)、[平衡数据](https://www.cockroachlabs.com/docs/stable/demo-automatic-rebalancing.html)、[容错](https://www.cockroachlabs.com/docs/stable/demo-fault-tolerance-and-recovery.html)案例。

## 第五步：停止集群

当你完成对集群的测试，打开终端进入第一个节点，按 **CTRL + C** 来停止集群的运行。

此时，仍然有 2 个节点在运行，因为大多数副本还是可用的，因此集群也仍然在运转中。为了验证集群是否有这种错误，连接节点 2 和 3 内置的 SQL 终端来进行测试。以下命令你可以在同一个终端中运行也可以在新的终端中运行：

```sh
$ cockroach sql --insecure --port=26258
```

``` sql
> SELECT * FROM bank.accounts;
```

```
+----+---------+
| id | balance |
+----+---------+
|  1 |  1000.5 |
+----+---------+
(1 row)
```

退出 SQL 终端：

``` sql
> \q
```

现在可以进入节点 2 和 3 各自的终端并按 **CTRL + C** 来停止节点。

> **提示：**<br>节点 3 的关闭过程将耗费较长时间（约 1 分钟）并最终被强制关闭。这是因为当 3 个节点中只有一个留存时，大多数的副本数据都已不可用，因此集群也不再运行。若要加速这个过程，直接按 **CTRL + C**。

如果你以后不需要再重启集群，你可以移除节点存储的数据：

``` shell
$ rm -rf cockroach-data node2 node3
```

## 第六步：重启集群

如果你决定使用集群进行进一步的测试，则需要从包含节点数据存储的目录中重新启动三个节点中的至少两个节点。

从 `cockroach-data/` 父目录中重启第一个节点：

``` shell
$ cockroach start --insecure \
--host=localhost
```

> **注意：**<br>随着唯一的一个节点重新回到线上，集群还不能运行，所以在重新启动第二个节点之前，你不会看到对上述命令的响应。

打开一个新的终端，从 `node2/` 父节点中重启第二个节点：

``` shell
$ cockroach start --insecure \
--store=node2 \
--host=localhost \
--port=26258 \
--http-port=8081 \
--join=localhost:26257
```

再打开一个新的终端，从 `node3/` 父节点中重启第三个节点：

``` shell
$ cockroach start --insecure \
--store=node3 \
--host=localhost \
--port=26259 \
--http-port=8082 \
--join=localhost:26257
```

## 下一步？

- 学习更多关于 [CockroachDB SQL](https://www.cockroachlabs.com/docs/stable/learn-cockroachdb-sql.html) 和 [内置 SQL 终端](https://www.cockroachlabs.com/docs/stable/use-the-built-in-sql-client.html) 的内容。
- 选择你喜欢的语言并对应[安装客户端驱动](https://www.cockroachlabs.com/docs/stable/install-client-drivers.html)
- [构建 CockroachDB 的应用](https://www.cockroachlabs.com/docs/stable/build-an-app-with-cockroachdb.html)
- 查看 [CockroachDB 核心优势](https://www.cockroachlabs.com/docs/stable/demo-data-replication.html)，如自动备份，数据均衡，容错机制和云迁移。
