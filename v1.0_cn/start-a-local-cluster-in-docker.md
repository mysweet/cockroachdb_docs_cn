# 在 Docker 中启动一个集群

## 简介

在单台主机上通过多个 Docker 容器来运行一个不安全的多节点 CockroachDB 集群。

只要你已经[安装了官方的 CockroachDB Docker 镜像]((install-cockroachdb.html))，在单台主机上通过多个 Docker 容器来运行一个不安全的多节点集群是非常简单地，使用 Docker 卷对节点的数据进行持久化。

## 警告

在 Docker 中运行一个像 CockroachDB 这样的有状态的应用比大多数使用 Docker 的应用更复杂和容易出错，不推荐在生产环境部署。可以使用像 Kubernetes 或 Docker Swarm 这样的编排工具，在容器中运行物理分布式集群。

## 开始前准备

确认你已经[安装了官方的 CockroachDB Docker 镜像](install-cockroachdb.html)。

## 1. 创建 一个 bridge 网络

由于你将要在单台主机上运行多个 Docker 容器，每个容器中有一个 CockroachDB 节点，所以需要创建 Docker 所引用的  [bridge 网络](https://docs.docker.com/engine/userguide/networking/#/a-bridge-network)。bridge 网络将使容器能够作为单个集群通信，同时保持与外部网络的隔离。

```shell
$ docker network create -d bridge roachnet
```

在这里及后面的步骤中，我们用 `roachnet` 作为这个网络的名称，但你可以使用任何你喜欢的名称。

## 2. 启动第一个节点

```shell
$ docker run -d \
--name=roach1 \
--hostname=roach1 \
--net=roachnet \
-p 26257:26257 -p 8080:8080  \
-v "${PWD}/cockroach-data/roach1:/cockroach/cockroach-data"  \
cockroachdb/cockroach:v1.0.4 start --insecure
```

这条命令创建一个容器，并且在里面启动了第一个 CockroachDB 节点。下面看一下每部分命令的含义：

- `docker run`: 启动一个新容器的 Docker 命令。

- `-d`: 这个标志表示在后台运行容器，以便可以在同一个 shell 窗口中继续执行下面的步骤。

- `--name`: 容器的名称。这是可选的，但自定义名称使得在其它命令中引用容器更容易，比如，当在容器中打开一个 Bash 会话或停止容器时。

- `--hostname`: 容器的主机名称。你将使用它来将其它容器/节点加入集群。

- `--net`: 这个容器要加入的 bridge 网络。更多详细信息请查看步骤 1。

- `-p 26257:26257 -p 8080:8080`: 这些标志映射节点间和客户端节点通信的默认端口 (`26257`)，以及从容器到主机的管理UI (`8080`) 的HTTP请求的默认端口。这使容器间通信成为可能，并且可以从浏览器调用管理 UI。

- `-v "${PWD}/cockroach-data/roach1:/cockroach/cockroach-data"`: 此标志将主机目录作为数据卷挂载。这意味着此节点的数据和日志将存储在主机的 `${PWD}/cockroach-data/roach1` 目录，并且在容器停止或删除后将继续存在。更多细节查看 Docker 的<a href="https://docs.docker.com/engine/tutorials/dockervolumes/#/mount-a-host-directory-as-a-data-volume">将主机目录挂载为数据卷</a>主题。

- `cockroachdb/cockroach:v1.0.4 start --insecure`: 在非安全模式下在容器中[启动一个节点](start-a-node.html)的 CockroachDB 命令。

  ### Tip：

  默认情况下，每个节点的缓存仅限于可用内存的 25%。当每台主机运行一个容器/节点时这个默认值是合理的。然而当在单台主机上运行多个容器/节点时，它可能导致内存错误，尤其是严重的对集群进行测试时。为避免这样的错误，可以通过设置 `start` 命令中的 `—cache` 标志来手动限制每个节点的缓存大小。

## 3. 向集群中添加节点

此时，你的集群处于活动状态。只使用一个节点，已经可以连接 SQL 客户端并开始构建数据库。然而在实际部署中，你将想用 3 个或更多节点来利用 CockroachDB 的 [自动复制](demo-data-replication.html)，[再平衡](demo-automatic-rebalancing.html) 和 [容错](demo-fault-tolerance-and-recovery.html) 能力的优势。

要模拟真实部署，通过添加两个节点来扩展集群：

```shell
# Start the second container/node:
$ docker run -d \
--name=roach2 \
--hostname=roach2 \
--net=roachnet \
-v "${PWD}/cockroach-data/roach2:/cockroach/cockroach-data" \
cockroachdb/cockroach:v1.0.4 start --insecure --join=roach1

# Start the third container/node:
$ docker run -d \
--name=roach3 \
--hostname=roach3 \
--net=roachnet \
-v "${PWD}/cockroach-data/roach3:/cockroach/cockroach-data" \
cockroachdb/cockroach:v1.0.4 start --insecure --join=roach1
```

这些命令添加了两个容器并在里面启动 CockroachDB 节点，将它们连接到第一个节点。这里与步骤 2 中有一些不同需要注意：

- `-v`: 此标志将主机目录作为数据卷挂载。这些节点的数据和日志将存储在主机的 `${PWD}/cockroach-data/roach2` 和 `${PWD}/cockroach-data/roach3` 目录，并且在容器停止或删除后将继续存在。
- `--join`: 此标志使用第一个容器的 `hostname` 将新节点加入到集群。需要注意的是，因为每个节点都是在独立的容器中，使用相同的默认端口不会引起冲突。

## 4. 测试集群

现在已经扩展到了 3 个节点，可以使用任何一个节点作为集群的 SQL 网关。为了证明这点，使用 `docker exec` 命令在第一个容器中来启动内置的 [SQL shell](use-the-built-in-sql-client.html)：

```shell
$ docker exec -it roach1 ./cockroach sql --insecure
# Welcome to the cockroach SQL interface.
# All statements must be terminated by a semicolon.
# To exit: CTRL + D.
```

运行一些基本的 [CockroachDB SQL 语句](learn-cockroachdb-sql.html)：

~~~ sql
> CREATE DATABASE bank;

> CREATE TABLE bank.accounts (id INT PRIMARY KEY, balance DECIMAL);

> INSERT INTO bank.accounts VALUES (1, 1000.50);

> SELECT * FROM bank.accounts;
~~~

~~~
+----+---------+
| id | balance |
+----+---------+
|  1 |  1000.5 |
+----+---------+
(1 row)
~~~

退出节点 1 上的 SQL shell：

~~~ sql
> \q
~~~

然后在第二个容器中启动 SQL shell：

```shell
$ docker exec -it roach2 ./cockroach sql --insecure
# Welcome to the cockroach SQL interface.
# All statements must be terminated by a semicolon.
# To exit: CTRL + D.
```

现在运行同样地 `SELECT` 查询：

~~~ sql
> SELECT * FROM bank.accounts;
~~~

~~~
+----+---------+
| id | balance |
+----+---------+
|  1 |  1000.5 |
+----+---------+
(1 row)
~~~

如你所见，节点 1 和节点 2 的行为和 SQL 网关相同。

完成后，退出节点 2 上的 SQL shell：

~~~ sql
> \q
~~~

## 5. 监控集群

启动第一个容器/节点时，你将节点的默认 HTTP 端口 `8080` 映射到主机的 `8080` 端口。要检查集群的[管理 UI](explore-the-admin-ui.html)，用浏览器连接到 `localhost` 上的该端口，即：`http://localhost:8080`。

![images/admin_ui.png](/Users/LLLeon/Downloads/LearningNotes/Cockroachdb/images/admin_ui.png)

如前所述，CockroachDB 自动在后台复制数据。为了验证在前面步骤中写入的数据复制成功了，向下滚动来查看：

![images/admin_ui_replicas.png](./images/admin_ui_replicas.png)

每个节点上的副本数量是相同的，表明集群中的所有数据被复制了 3 次（默认值）。

### Tip：

for more insight into how CockroachDB automatically replicates and rebalances data, and tolerates and recovers from failures, see our <a href="demo-data-replication.html">replication</a>, <a href="demo-automatic-rebalancing.html">rebalancing</a>, <a href="demo-fault-tolerance-and-recovery.html">fault tolerance</a> demos.

为了更深入的了解 CockroachDB 是怎样自动复制、再平衡数据、容错和故障恢复的，查看 <a href="demo-data-replication.html">replication</a>, <a href="demo-automatic-rebalancing.html">rebalancing</a>, <a href="demo-fault-tolerance-and-recovery.html">fault tolerance</a> 的示例。

## 6.  停止集群

Use the `docker stop` and `docker rm` commands to stop and remove the containers (and therefore the cluster):

使用 `docker stop` 和 `docker rm` 命令来停止和删除容器（集群也因此被删除）：

```shell
# Stop the containers:
$ docker stop roach1 roach2 roach3

# Remove the containers:
$ docker rm roach1 roach2 roach3
```

## 接下来做什么?

- 学习更过关于 [CockroachDB SQL](learn-cockroachdb-sql.html) 和 [built-in SQL client](use-the-built-in-sql-client.html)
- [Install the client driver](install-client-drivers.html) for your preferred language
- 安装你喜欢的语言的 [客户端驱动程序](install-client-drivers.html) 
- [用 CockroachDB 构建一个应用](build-an-app-with-cockroachdb.html)
- [探索 CockroachDB 的核心特性](demo-data-replication.html)，比如自动复制，再平衡，还有容错。