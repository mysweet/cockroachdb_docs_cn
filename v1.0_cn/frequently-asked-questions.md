# FAQ

## 什么是 CockroachDB？

CockroachDB 是一个建于事务性和强一致的键-值存储之上的分布式 SQL 数据库。它横向**扩展**；以最小的延迟终端并无需手动干预而**克服**磁盘、机器、机架，甚至数据中心的故障；支持**强一致** ACID 事务；并一组 **SQL** API 用于结构化、操作和查询数据。

CockroachDB 受谷歌的 [Spanner](http://research.google.com/archive/spanner.html) 和 [F1](http://research.google.com/pubs/pub38125.html) 技术启发，而且它是完全[开源的](https://github.com/cockroachdb/cockroach)。

## 何时选择 CockroachDB 好？

CockroachDB 很适合要求可靠、可用、正确的数据而不管规模的应用。它被构建为以最小的配置和运行开销自动复制、再平衡和恢复。特定的用例包括：

- 分布式或复制的 OLTP
- 多数据中心部署
- 云原生的基础设施行动

## 何时选择 CockroachDB 不好？

CockroachDB 在很低的读和写延迟是关键时不是一个好的选择；这种情况应该使用内存数据库。

而且，CockroachDB 还不适用于：

- 复杂的 SQL 连接 ([这个特性仍需要优化](https://www.cockroachlabs.com/blog/cockroachdbs-first-join/))
- 重度分析 / OLAP

## 安装 CockroachDB 有多么容易？

在 OS X 和 Linux 上就是下载一个二进制文件，在 Windows 上就是运行一个官方 Docker 镜像。也有其他的简单安装方法，如在 OS X 上运行我们的 Homebrew recipe，或者在 OS X 和 Linux 上从源文件构建。

更多细节，见[安装 CockroachDB](install-cockroachdb.md)。

## CockroachDB 如何扩展？

CockroachDB 以最小的运行开销横向扩展。你可以在你本地的计算机、单一服务器、公司的开发集群，或者私有或公有云上运行它。[增加容量](start-a-node.md)容易得就是指向一个运行集群中的新节点。

在键-值层，CockroachDB 以一个单一的、空的域启动。当你放入数据，这个单一域最终达到了一个阈值（默认 64MB）。当达到时，数据被分入两个域，每个包含整个键-值空间的一个连续段。这个过程无限制地继续；当新的数据进来，现有的域继续分裂为新的域，目标是保持相对小和一致的域大小。

当你的集群跨多个节点（物理机、虚拟机、或者容器），新分裂的域被自动再平衡到有更大容量的节点。CockroachDB 使用一个端到端的[闲话协议](https://en.wikipedia.org/wiki/Gossip_protocol)沟通再平衡的机会，通过这个协议，节点交换网络地址、存储容量和其他信息。

## CockroachDB 如何在故障中运行？

CockroachDB 被设计为在从服务器重启到数据中心停电的软件和硬件故障中运行。这个目标的达到使用了强一致的复制以及故障之后的自动修复，没有其他分布式系统中典型的令人迷惑的东西（即，过期的读）。

**复制**

CockroachDB 为了可用性复制你的数据并使用 [Raft 共识算法](https://raft.github.io/)保证副本之间的一致性，这个算法是 [Paxos](http://research.microsoft.com/en-us/um/people/lamport/pubs/paxos-simple.pdf)的一个流行的替代品。取决于你要预防的故障和你的网络拓扑，可以用不同的方式[定义副本的位置](configure-replication-zones.md)。你可以在下面的位置放置副本：

- 一个机架上的不同服务器，以容许服务器故障
- 一个数据中心内的不同机架上的不同服务器，以容许机架电力/网络故障
- 不同数据中心的不同服务器，以容许大规模的网络故障故障或停电

在跨数据中心复制时，我们推荐使用单一洲上的数据中心，以保证更好的性能。跨洲和其他高延迟的情况将在未来被更好地支持。

**自动修复**

对于短期故障，例如服务器重启，只要大部分副本可用，CockroachDB 使用 Raft 持续无缝运行。如果前面的领导者时序，Raft 确保每个副本组的一个新“领导者”被选出，因而事务可以继续，而且被影响的副本一旦回到线上，能够重新加入它们的组。对于长期故障，例如服务器/机架长时间宕机或者一个数据中心停电，CockroachDB 使用未被影响的副本作为源，自动再平衡消失的节点。使用来自闲聊网络的容量信息，确认在集群中的新位置，而且使用集群的所有可用节点和合并的磁盘以及网络带宽，以分布式的形式再复制消失的副本。

## CockroachDB 是如何做到强一致的？

CockroachDB 保证 SQL 隔离层 "可序列化"，是 SQL 标准定义的最高级别。它结合了 Raft 共识算法用于写和自定义的基于时间的同步算法用于读。更多细节，见我们关于[强一致性](strong-consistency.md)的描述。

## CockroachDB 是如何既高可用又强一致的？

[CAP 定理](https://en.wikipedia.org/wiki/CAP_theorem) 指出，对于一个分布式系统只能同时保证下列三个中的最多两个：

- 一致性
- 可用性
- 分区容许

CockroachDB 是一个 CP (一致性和分区容许的)系统。这意味着，在分区的情况下，系统会变得不可用而不是做任何造成不一致后果的事情。例如，写要求来自大部分副本的确认，读要求一个租约，当写是可能的时候才被转换到一个不同的节点。

另外，CockroachDB 也是高可用的，尽管，这里的“可用”和 CAP 定理中的含义不同。在 CAP 定理中，可用性是一个二元属性，但对于高可用性，我们认为它是一个范围（对于在 99.999% 的时间可用的系统使用类似“5 个 9”的术语）。

同时是 CP 和 HA 意味着，当大部分的副本能够相互通信，它们就应该能够取得进展。例如，如果你将 CockroachDB 部署到三个数据中心，而连接到它们中的一个的网络发生故障，另外两个数据中心一个能够正常运行而只有几秒钟的中断。我们以这种方式快速有效地检测分区和故障，将领导权转移到能够与大多数通信的节点，将内部流量从被分区的节点转移走。

## CockroachDB 为什么是 SQL?

在最底层，CockroachDB 是一个分布式、强一致、事务性的键-值存储，但是外部 API 是一个带扩展的标准 SQL。这为开发者提供了熟悉的关系概念，例如模式、表、列和索引，以及能够使用成熟的、经时间证明的工具和过程结构化、操作并查询数据。而且，由于 CockroachDB 支持 PostgreSQL 连线协议，让你的应用与 Cockroach 交互很简单；只需找到 [PostgreSQL 特定于语言的驱动程序](install-client-drivers.md)并开始构建。

更多细节，学习[基本的 CockroachDB SQL 语句](learn-cockroachdb-sql.md)，探索[完整的 SQL 语法](sql-grammar.md)，并通过我们的[内建 SQL 客户端](use-the-built-in-sql-client.md)尝试。而且，要理解 CockroachDB 如何将 SQL 表数据映射为键-值存储以及 CockroachDB 如何选择最佳索引用以运行一个查询，见 [SQL in CockroachDB](https://www.cockroachlabs.com/blog/sql-in-cockroachdb-mapping-table-data-to-key-value-storage/) 和 [Index Selection in CockroachDB](https://www.cockroachlabs.com/blog/index-selection-cockroachdb-2/)。

## CockroachDB 支持分布式事务吗？

是的。CockroachDB 跨集群分布事务，无论是某个位置的几台服务器还是跨多个数据中心的很多服务器。不同于分片的设置，你不需要知道数据的准确位置；只需与集群中的任意节点通信，CockroachDB 将事务无缝地放到正确的位置。分布式事务进行，在有再平衡的时候，没有停机或者额外的延迟。在集群负载足够的情况下，你甚至能够在数据中心或者云基础设施提供商之间移动表或者整个数据库。

## CockroachDB 中的事务保证 ACID 语义吗?

是的。CockroachDB 中的每个[事务](transactions.md)保证 [ACID 语义](https://en.wikipedia.org/wiki/ACID) 跨任意表和行，甚至在数据是分布的时候。

- **原子性:**  CockroachDB 中的事务是“全部或没有。” 如果一个事务的任何部分失败，整个事务被退出，数据库没有变化。如果一个事务成功，所有的变动几乎是同时被应用。关于 CockroachDB  事务原子性的详细讨论，见[CockroachDB 如何分布原子事务](https://www.cockroachlabs.com/blog/how-cockroachdb-distributes-atomic-transactions/)。
- **一致性:** SQL 操作永远不会见到任何中间状态，并将数据库从一个有效状态转到另一个有效状态，保持索引更新。操作总是见到前面完成的重叠数据的语句的结果，并维护特定的限制，如唯一的列。关于我们如何测试 CockroachDB 的正确性和一致性的细致观察，见 [CockroachDB DIY Jepsen 测试](https://www.cockroachlabs.com/blog/diy-jepsen-testing-cockroachdb/)。
- **隔离:** CockroachDB 中的事务默认使用可序列化快照隔离 (SSI)。这意味着，即使是并发的读-写事务也从不会导致异常。我们也提供快照隔离 (SI)，这对于高竞争的负载性能更好，尽管它会出现 SSI 中没有的异常（写偏）。关于 CockroachDB 事务隔离的详细讨论，见[可序列化、无锁的、分布式：CockroachDB 中的隔离](https://www.cockroachlabs.com/blog/serializable-lockless-distributed-isolation-cockroachdb/)。
- **持久性:** 在 CockroachDB 中，每一个被确认的写在大多数（默认至少是 2 个）副本上通过 [Raft 共识算法](https://raft.github.io/)保持一致性。仅影响少数（典型是 1 个）副本的供电或者磁盘故障不会阻碍集群运行，而且不会丢失任何数据。

## 由于 CockroachDB 是受 Spanner 启发的，它需要原子时钟同步时间吗？

不需要。CockroachDB 被设计为不使用原子时钟或 GPS 时钟工作。它是一个意于运行在任何节点集合上的开源数据库，从一个公司开发集群的物理服务器到使用最新流行的虚拟化层的公有云基础设施。要求一个外部依赖于特殊的硬件用于时钟同步，将是一个搅局者。然而，CockroachDB 确实要求中等水平的时钟同步，用于保证正确性。如果时钟漂移超出了一个最大阈值，节点会被下线。因此，高度推荐在每个节点上运行 [NTP](http://www.ntp.org/) 或者时钟同步软件。

关于 CockroachDB 如何处理非同步时钟的更多细节，见[时钟同步](recommended-production-settings.md#clock-synchronization)。而关于时钟，以及 Spanner 和 CockroachDB 中时钟差异的更广泛讨论，见[没有原子时钟的办法](https://www.cockroachlabs.com/blog/living-without-atomic-clocks/)。

## 哪些语言可以和 CockroachDB 一起使用?

CockroachDB 支持 PostgreSQL 连线协议，所以你能够使用任何可用的 PostgreSQL 客户端驱动程序。我们用下列语言进行了测试：

- Go
- Python
- Ruby
- Java
- JavaScript (node.js)
- C++/C
- Clojure
- PHP
- Rust

更多细节见[安装驱动程序](install-client-drivers.md)。

## 为什么 CockroachDB 使用 PostgreSQL 连线协议而不是 MySQL 协议？

CockroachDB 使用 PostgreSQL 连线协议，因为它的文档比 MySQL 协议的文档要好，还因为 PostgreSQL 有一个更自由的开源许可，类似于 BSD 或 MIT 许可，而 MySQL 有一个限制更多的 GNU GPL 许可。

然而，注意，使用的协议并没有严重影响移植应用程序有多容易。在几乎每种语言中更换 SQL 网络驱动程序都是相当直接明了。令从一个数据库换为另一个数据库困难的是所使用的 SQL 方言。 CockroachDB 的方言也是基于 PostgreSQL 的。

## 什么是 CockroachDB 的安全模型？

你可以运行一个安全或者不安全的 CockroachDB 集群。安全的，客户端/节点和节点间通信是加密的，而且 SSL 证书认证客户端和节点的身份。不安全的，没有加密和认证。

而且，CockroachDB 支持通用的 SQL 数据库和表权限。`root` 用户拥有所有数据库的权限，而单个用户可以被授予特定语句在数据库和表级别的权限。

更多细节，见我们关于[权限](privileges.md)和 [`GRANT`](grant.md) 语句的文档。

## CockroachDB 与 MySQL 或者 PostgreSQL 如何比较？

尽管所有这些数据库都支持 SQL 语法，CockroachDB 是唯一一个容易地扩展（不需要手动分片的复杂操作），自动再平衡和修复，而且跨集群无缝分布事务。

更多深刻见解，见[比较 CockroachDB](cockroachdb-in-comparison.md)。

## CockroachDB 与 Cassandra, HBase, MongoDB 或 Riak 如何比较？

尽管所有这些都是分布式数据库，只有 CockroachDB 支持分布式事务并提供强一致。而且，这些其他数据库提供定制的 API，而 CockroachDB 提供标准的 SQL 并扩展。

更多深刻见解，见[比较 CockroachDB](cockroachdb-in-comparison.md)。

## MySQL 或者 PostgreSQL 应用能够迁移到 CockroachDB 吗？

CockroachDB 的当前版本是用于新的应用的。我们支持的 SQL 的最初子集相对于广泛的标准是小的，每一个流行的数据库实现了它自己的扩展并展现了唯一的特质集合。这使得移植一个现有的应用不是轻而易举的，除非它是一个非常轻量级的 SQL 功能。

## Cockroach Labs 提供云数据库作为服务吗？

还没有，但是这在我们的长期计划中。

## 我能使用 CockroachDB 作为键-值存储吗？

CockroachDB 是一个构建于一个事务性和强一致的键-值存储之上的分布式 SQL 数据库。尽管不可能直接访问键-值存储，你可以只用一个两列的“简单”表映射直接访问，其中一列被设置为主键：

~~~ sql
> CREATE TABLE kv (k INT PRIMARY KEY, v BYTES);
~~~

当这样一张“简单的”表没有所以或外键时，[`INSERT`](insert.md)/[`UPSERT`](upsert.md)/[`UPDATE`](update.md)/[`DELETE`](delete.md) 语句以极小的开销（个位数百分比的性能下降）翻译为键-值操作。例如，下面的 `UPSERT` 加入或替换表中的一行会翻译为一条键-值 Put 操作：

~~~ sql
> UPSERT INTO kv VALUES (1, b'hello')
~~~

这个 SQL 表方法也提供了一个定义良好的查询语言，一个已知的事务模型，以及在需要时加入更多列到表中的灵活性。

## 有没被回答的问题吗？

- [CockroachDB 社区论坛](https://forum.cockroachlabs.com)：问问题，寻找答案，并帮助其他用户。
- [在 Gitter 上加入我们](https://gitter.im/cockroachdb/cockroach)：这是与 CockroachDB 工程师们连接的最直接方式。打开 Gitter 而不离开这些文档，在任何页面的右下角点击 **Chat with Developers**。
- [SQL FAQs](sql-faqs.md)：获得关于 CockroachDB SQL 的常问问题的答案。
- [运行 FAQs](operational-faqs.md)：获得关于操作 CockroachDB 的常问问题的答案。

