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

- **原子性:** Transactions in CockroachDB are “all or nothing.” If any part of a transaction fails, the entire transaction is aborted, and the database is left unchanged. If a transaction succeeds, all mutations are applied together with virtual simultaneity. For a detailed discussion of atomicity in CockroachDB transactions, see [How CockroachDB Distributes Atomic Transactions](https://www.cockroachlabs.com/blog/how-cockroachdb-distributes-atomic-transactions/).
- **一致性:** SQL operations never see any intermediate states and move the database from one valid state to another, keeping indexes up to date. Operations always see the results of previously completed statements on overlapping data and maintain specified constraints such as unique columns. For a detailed look at how we've tested CockroachDB for correctness and consistency, see [DIY Jepsen Testing of CockroachDB](https://www.cockroachlabs.com/blog/diy-jepsen-testing-cockroachdb/).
- **隔离:** By default, transactions in CockroachDB use serializable snapshot isolation (SSI). This means that even concurrent read-write transactions will never result in anomalies. We also provide snapshot isolation (SI), which is more performant with high-contention workloads, although it exhibits anomalies not present in SSI (write skew). For a detailed discussion of isolation in CockroachDB transactions, see [Serializable, Lockless, Distributed: Isolation in CockroachDB](https://www.cockroachlabs.com/blog/serializable-lockless-distributed-isolation-cockroachdb/).
- **持久性:** In CockroachDB, every acknowledged write has been persisted consistently on a majority of replicas (by default, at least 2) via the [Raft consensus algorithm](https://raft.github.io/). Power or disk failures that affect only a minority of replicas (typically 1) do not prevent the cluster from operating and do not lose any data.

## 由于 CockroachDB 是受 Spanner 启发的，它需要原子时钟同步时间吗？

No. CockroachDB was designed to work without atomic clocks or GPS clocks. It’s an open source database intended to be run on arbitrary collections of nodes, from physical servers in a corp development cluster to public cloud infrastructure using the flavor-of-the-month virtualization layer. It’d be a showstopper to require an external dependency on specialized hardware for clock synchronization. However, CockroachDB does require moderate levels of clock synchronization for correctness. If clocks drift past a maximum threshold, nodes will be taken offline. It's therefore highly recommended to run [NTP](http://www.ntp.org/) or other clock synchronization software on each node.

For more details on how CockroachDB handles unsychronized clocks, see [Clock Synchronization](recommended-production-settings.md#clock-synchronization). And for a broader discussion of clocks, and the differences between clocks in Spanner and CockroachDB, see [Living Without Atomic Clocks](https://www.cockroachlabs.com/blog/living-without-atomic-clocks/).

## 哪些语言可以和 CockroachDB 一起使用?

CockroachDB supports the PostgreSQL wire protocol, so you can use any available PostgreSQL client drivers. We've tested it from the following languages:

- Go
- Python
- Ruby
- Java
- JavaScript (node.js)
- C++/C
- Clojure
- PHP
- Rust

See [Install Client Drivers](install-client-drivers.md) for more details.

## 什么是 CockroachDB 的安全模型？

You can run a secure or insecure CockroachDB cluster. When secure, client/node and inter-node communication is encrypted, and SSL certificates authenticate the identity of both clients and nodes. When insecure, there's no encryption or authentication.

Also, CockroachDB supports common SQL privileges on databases and tables. The `root` user has privileges for all databases, while unique users can be granted privileges for specific statements at the database and table-levels.

For more details, see our documentation on [privileges](privileges.md) and the [`GRANT`](grant.md) statement.

## CockroachDB 与 MySQL 或者 PostgreSQL 如何比较？

While all of these databases support SQL syntax, CockroachDB is the only one that scales easily (without the manual complexity of sharding), rebalances and repairs itself automatically, and distributes transactions seamlessly across your cluster.

For more insight, see [CockroachDB in Comparison](cockroachdb-in-comparison.md).

## CockroachDB 与 Cassandra, HBase, MongoDB 或 Riak 如何比较？

While all of these are distributed databases, only CockroachDB supports distributed transactions and provides strong consistency. Also, these other databases provide custom APIs, whereas CockroachDB offers standard SQL with extensions.

For more insight, see [CockroachDB in Comparison](cockroachdb-in-comparison.md).

## MySQL 或者 PostgreSQL 应用能够迁移到 CockroachDB 吗？

The current version of CockroachDB is intended for use with new applications. The initial subset of SQL we support is small relative to the extensive standard, and every popular database implements its own set of extensions and exhibits a unique set of idiosyncrasies. This makes porting an existing application non-trivial unless it is only a very lightweight consumer of SQL functionality.

## Cockroach Labs 提供云数据库作为服务吗？

Not yet, but this is on our long-term roadmap.

## 我能使用 CockroachDB 作为键-值存储吗？

{% include faq/simulate-key-value-store.md %}

## 有没被回答的问题吗？

- [CockroachDB Community Forum](https://forum.cockroachlabs.com): Ask questions, find answers, and help other users.
- [Join us on Gitter](https://gitter.im/cockroachdb/cockroach): This is the most immediate way to connect with CockroachDB engineers. To open Gitter without leaving these docs, click **Chat with Developers** in the lower-right corner of any page.
- [SQL FAQs](sql-faqs.md): Get answers to frequently asked questions about CockroachDB SQL.
- [Operational FAQS](operational-faqs.md): Get answers to frequently asked questions about operating CockroachDB.

