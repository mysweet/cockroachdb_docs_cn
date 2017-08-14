# 在 Docker 中启动一个集群（不安全的）

只要你已经[安装了官方的 CockroachDB Docker 镜像](install-cockroachdb.md)，在单台主机上通过多个 Docker 容器来运行一个不安全的多节点集群是简单的，使用 Docker 卷持久化节点数据。

> ***警告***

> 在 Docker 中运行一个像 CockroachDB 这样的有状态的应用比大多数使用 Docker 的应用更复杂和容易出错，不推荐用于生产环境部署。使用像 Kubernetes 或 Docker Swarm 这样的编排工具，在容器中运行物理上分布的集群。更多细节见[编排](orchestration.md)。

## 准备

确认你已经[安装了官方的 CockroachDB Docker 镜像](install-cockroachdb.md)。

## 第一步：创建桥接网络

由于你将要在单台主机上运行多个 Docker 容器，每个容器运行一个 CockroachDB 节点，需要创建 Docker 称为的[桥接网络](https://docs.docker.com/engine/userguide/networking/#/a-bridge-network)。桥接网络将使容器能够作为单一集群通信，同时保持与外部网络的隔离。

```shell
$ docker network create -d bridge roachnet
```

在这里及后续步骤中，我们用 `roachnet` 作为网络的名称，但你可以使用任何你喜欢的名称。

## 第二步：启动第一个节点

```shell
$ docker run -d \
--name=roach1 \
--hostname=roach1 \
--net=roachnet \
-p 26257:26257 -p 8080:8080  \
-v "${PWD}/cockroach-data/roach1:/cockroach/cockroach-data"  \
cockroachdb/cockroach:v1.0.4 start --insecure
```

这条命令创建一个容器，并且在里面启动了第一个 CockroachDB 节点。下面看一下命令的每部分的含义：

- `docker run`: 启动一个新容器的 Docker 命令。

- `-d`: 这个标志表示在后台运行容器，以便可以在同一个 shell 中继续执行下面的步骤。

- `--name`: 容器的名称。这是可选的，但自定义名称在其它命令中引用容器非常容易，比如，在容器中打开一个 Bash 会话或停止容器。

- `--hostname`: 容器的主机名称。你将使用它来将其它容器/节点加入集群。

- `--net`: 这个容器要加入的桥接网络。更多细节请见第一步。

- `-p 26257:26257 -p 8080:8080`: 这些标志映射节点间和客户端-节点间通信的默认端口 (`26257`)，以及从容器到主机的管理 UI (`8080`) 的 HTTP 请求的默认端口。这使容器间通信成为可能，并且可以从浏览器调用管理 UI。

- `-v "${PWD}/cockroach-data/roach1:/cockroach/cockroach-data"`: 此标志将主机目录挂载为数据卷。这意味着此节点的数据和日志将存储在主机的 `${PWD}/cockroach-data/roach1` 目录中，并且在容器停止或删除后将继续存在。更多细节查看 Docker 的<a href="https://docs.docker.com/engine/tutorials/dockervolumes/#/mount-a-host-directory-as-a-data-volume">将主机目录挂载为数据卷</a>主题。

- `cockroachdb/cockroach:v1.0.4 start --insecure`: 在非安全模式下在容器中[启动一个节点](start-a-node.html)的 CockroachDB 命令。

> **提示**：

> 默认情况下，每个节点的缓存仅限于可用内存的 25%。当每台主机运行一个容器/节点时，这个默认值是合理的。然而当在单台主机上运行多个容器/节点时，可能导致内存错误，尤其是对集群进行认真测试时。为避免这样的错误，可以通过设置 `start` 命令中的 `—cache` 标志来手动限制每个节点的缓存大小。

## 第三步：向集群中添加节点

此时，集群处于活动状态。只用一个节点，你已经可以连接 SQL 客户端并开始构建数据库。然而，在实际部署中，你将想用 3 个或更多节点来利用 CockroachDB 的 [自动复制](demo-data-replication.md)，[再平衡](demo-automatic-rebalancing.md)和[容错](demo-fault-tolerance-and-recovery.md)功能。

为了模拟真实的部署，通过添加两个节点扩展集群：

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

这些命令添加了两个容器并在里面启动 CockroachDB 节点，将它们连接到第一个节点。这里与第二步中有一些需要注意的不同：

- `-v`: 此标志将主机目录挂载为数据卷。这些节点的数据和日志将存储在主机的 `${PWD}/cockroach-data/roach2` 和 `${PWD}/cockroach-data/roach3` 目录，并且在容器停止或删除后将继续存在。
- `--join`: 此标志使用第一个容器的 `hostname` 将新节点加入到集群。否则，[`cockroach start`](start-a-node.md) 的所有默认值被接受。注意，因为每个节点都是在单独的容器中，使用相同的默认端口不会引起冲突。

## 第四步：测试集群

现在已经扩展到了 3 个节点，可以使用任何一个节点作为集群的 SQL 网关。为了显示这一点，使用 `docker exec` 命令在第一个容器中启动内置的 [SQL shell](use-the-built-in-sql-client.md)：

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

然后，在第二个容器中启动 SQL shell：

```shell
$ docker exec -it roach2 ./cockroach sql --insecure
# Welcome to the cockroach SQL interface.
# All statements must be terminated by a semicolon.
# To exit: CTRL + D.
```

现在运行同样的 `SELECT` 查询：

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

## 第五步：监控集群

启动第一个容器/节点时，你将节点的默认 HTTP 端口 `8080` 映射到主机的 `8080` 端口。要检查集群的[管理 UI](explore-the-admin-ui.md)，用浏览器连接到 `localhost` 上的该端口，即：`http://localhost:8080`。

![images/admin_ui.png](images/admin_ui.png)

如前所述，CockroachDB 自动在后台复制数据。为了验证在前面步骤中写入的数据被成功复制，向下滚动到*每个节点的存储*并将光标悬停在线上：

![images/admin_ui_replicas.png](./images/admin_ui_replicas.png)

每个节点上的副本数量是相同的，表明集群中的所有数据被复制了 3 次（默认值）。

> **提示**：

> 为了更深入的了解 CockroachDB 是怎样自动复制、再平衡数据、容错和故障恢复的，查看 [复制](demo-data-replication.md)，[再平衡](demo-automatic-rebalancing.md)，[容错](demo-fault-tolerance-and-recovery.html)演示。

## 第六步：停止集群

使用 `docker stop` 和 `docker rm` 命令停止和删除容器（集群也因此被停止和删除）：

```shell
# Stop the containers:
$ docker stop roach1 roach2 roach3

# Remove the containers:
$ docker rm roach1 roach2 roach3
```

## 下一步

- 学习更过关于 [CockroachDB SQL](learn-cockroachdb-sql.md) 和 [内建 SQL 客户端](use-the-built-in-sql-client.md)
- 安装你喜欢的语言的[客户端驱动程序](install-client-drivers.md) 
- [用 CockroachDB 构建一个应用](build-an-app-with-cockroachdb.md)
- [探索 CockroachDB 的核心功能](demo-data-replication.md)，比如自动复制，再平衡和容错。