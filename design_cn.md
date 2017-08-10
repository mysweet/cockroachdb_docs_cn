# 关于

本文档是 Spencer Kimball 在2014年早期写的最初的设计文档的一个更新版。

# 概述

CockroachDB 是一个分布式 SQL 数据库，其首要设计目标是**可扩展性**，**强一致性**和**生存性**。CockroachDB 的目标是，以最小的延迟扰乱且不需手动干预，而能够容忍磁盘、计算机，机架，甚至**数据中心**的失效。CockroachDB 节点是对称的；一个设计目标是以最小的配置且不需要额外的依赖**同质部署** （一个二进制文件）。

数据库客户端的入口点是 SQL 界面。一个CockroachDB 集群中的每个节点都可以作为一个客户端 SQL 网关。客户端 SQL 网关将客户端 SQL 语句转换为键-值（KV）操作并执行，如果需要，网关会发布到整个集群，并返回结果给客户端。CockroachDB 实现了一个**单一的、巨大的、排序的映射**，映射从键到值，其中键值都是字节串。

键值映射在逻辑上由称为域（域）的键空间的小片段构成。每个域由存储在本地的 KV 存储引擎（我们使用[RocksDB](http://rocksdb.org)，[LevelDB](https://github.com/google/leveldb) 的一个变种）中的数据支持。域数据被复制到数量可配置的其它 CockroachDB 节点上。域被合并和分裂，以维持一个目标尺寸，缺省为`64M`。相对小的尺寸有利于修复和再平衡，以处理节点失效、新的容量，甚至读/写负载。然而，尺寸必须针对系统管理更多的域所带来的压力进行平衡。

CockroachDB 实现水平可扩展性的途径：
- 添加更多的节点，通过每个节点上的存储数量来提供集群的容量（除以一个可配置的复制因子），理论上最高可达 4艾 (4E，4*2^16) 字节的逻辑数据；
- 客户端查询可被发送到集群中的任何节点，查询可独立操作（没有冲突），意即，总体吞吐量随集群中节点数量线性增长。
- 查询是分布式的（参考：分布式 SQL），因此，单个查询的总体吞吐量可以通过添加更多节点而提高。

CockroachDB 实现强一致性的途径：
- 使用一个分布式的一致协议，用于同步复制每个键值域中的数据。我们选择使用[Raft 一致算法](https://raftconsensus.github.io)；所有一致状态存储在 RocksDB 中。
- 单个域的单个或批量转换通过域的域的 Raft 实例调解。Raft 保证 ACID 语义。
- 影响多个域的逻辑转换使用分布式事务，用于 ACID 语义。CockroachDB 使用一个高效的**非锁定分布式提交**协议。

CockroachDB 实现生存性的途径：
- 域副本可以共存于一个数据中心内，已实现低延迟复制并容许磁盘或计算机故障。它们可以跨机架分布，以容许一些网络交换机故障。
- 域副本可存在于跨增长的不同的地理区域的数据中心中，以容许从数据中心或网络失效到区域电力故障的更大的故障场景，
  （如 `{ US-East-1a, US-East-1b, US-East-1c }`, `{ US-East, US-West,
  Japan }`, `{ Ireland, US-East, US-West}`, `{ Ireland, US-East,
  US-West, Japan, Australia }`）。

CockroachDB 提供 [快照隔离](http://en.wikipedia.org/wiki/Snapshot_isolation) (SI) 和可序列化快照隔离 (SSI) 语义，允许**外部一致的，无锁读和写** -- 两者都是从一个历史的快照时间戳以及当前的系统时间。SI 提供无锁的读和写，但仍然允许写偏。SSI 消除了写偏，但是在有争议的系统的情况下引入对性能的影响。SSI 是缺省的隔离；客户端必须有意识地决定为性能牺牲正确性。CockroachDB 实现了[一个有限形式的线性化](#strict-serializability-linearizability)，为任意的观察者或观察者链提供顺序。

类似于
[Spanner](http://static.googleusercontent.com/media/research.google.com/en/us/archive/spanner-osdi2012.pdf)
目录，CockroachDB 运行任意数据区的配置。这允许选择复制因子、存储设备类型，和/或数据中心位置，以优化性能和/或可用性。不同于 Spanner 的是，区是巨大的并且不允许在实体组级移动细粒度的数据。

# 架构

CockroachDB 实现了一个分层的架构。抽象的最高级是 SQL 层（当前在本文档中没有说明）。它直接依赖于 [*SQL 层*](#sql)，后者提供了熟悉的关系概念，如模式、表、列和索引。SQL 层顺次依赖于[分布式键值存储](#key-value-api)，后者处理域寻址的细节，以提供一个单一的、巨大的键值存储的抽象。分布式键值存储与任意数目的物理 CockroachDB 节点通信。每个节点有一个或多个存储，每个物理设备一个。

![架构](media/architecture.png)

每个存储潜在包含很多域，即最低层的键-值数据单元。域使用 Raft 一致协议复制。下图是上图中五个节点中四个节点的放大版。每个域以三种方式使用 Raft 复制。颜色编码表示出相关的域复制。

![域](media/域s.png)

每个物理节点提供两个基于 RPC 的键值 API：一个给外部的客户端，一个给内部的客户端（暴露敏感的运行功能）。两项服务接受批量的请求，并返回批量的回应。节点在能力和提供的界面上是对称的；每个有同样的二进制文件并且能够担任任何角色。

节点和它们提供访问的域可以布置在各种物理网络拓扑上，以在可靠性和性能之间取得平衡。；例如，一个三重（三路复制）的域可以将每个副本放在：

-   一台服务器的不同磁盘上，以容许磁盘故障。
-   一个机架的不同服务器上，以容许服务器故障。
-   一个数据中心的不同机架的不同服务器上，以容许机架电源/网络故障。
-   不同数据中心的不同服务器上，以容许大规模的网络或电力停运。

最多可以容许 `F` 个故障，其中，副本的总数为 `N = 2F + 1` （即，以 3 倍的复制，可以容许一个故障；以 5 倍的复制，可以容许两个故障，以此类推）。

# 键

CockroachDB 键是任意的字节串。键分两种：系统键和表数据键。系统键被 CockroachDB 用于内部数据结构和元数据。表数据键包含 SQL 表数据（以及索引数据）。系统和表数据键的前缀方式使得所有系统键排序在表数据键的前面。

系统键分为几个子类：

- **全局**键保存集群范围的数据，如 "meta1" 和 "meta2" 键，以及其他的系统范围的键，如节点和存储的 ID 分配器。
- **存储本地**键用于非复制的存储元数据（即，`StoreIdent` 结构）。"非复制的"意味着这些值没有被跨多个存储复制，因为它们存的数据与其存在的存储的生命周期绑定。
- **域本地**键保存域与全局键关联的元数据。域本地键有一个特别的前缀，后面是全局键和一个特别后缀。例如，事务记录是域本地键，看起来是这样的：
    `\x01k<global-key>txn-<txnID>`.
- **复制的域 ID 本地**键保存一个域的所有副本中都有的域元数据。这些键通过 Raft 操作更新。例子包括域租约状态和退出缓存条目。
- **非复制的域 ID 本地**键保存副本本地的域元数据。这样的键的主要的例子是 Raft 状态和 Raft 日志。

表数据键用于保存所有的 SQL 数据。表数据键包含在[在 SQL 模型和 KV 间映射数据](#data-mapping-between-the-sql-model-and-kv)一节中描述的内部结构。

# 带版本的值

CockroachDB 通过保存相关的提交时间戳，维护值的历史版本。读和扫描可以指定一个快照时间，以返回快照时间戳之前的最近的写。就的数据版本在压紧期间按照用户定义的到期间隔被系统垃圾回收。为了支持长时间运行的扫描（即用于 MapReduce），所有版本有一个最小的到期间隔。

对带版本的值的支持是通过修改 RocksDB 以记录提交时间戳和每天的 GC 到期间隔完成的。

# 无锁的分布式事务

CockroachDB 提供不带锁的分布式事务。CockroachDB 事务支持两个级别的隔离：

- 快照隔离（SI）和
- *可序列化*快照隔离（SSI）。

*SI* 实现简单，高效，对于除了不多的几个异常情况（即，写偏）都是正确的。*SSI* 要求一点更多的复杂性，仍旧高效（对于竞争差一点），而且没有异常情况。CockroachDB 的 SSI 实现基于来自文献和一些可能新颖的想法。

SSI 是缺省的级别，而 SI 提供给应用开发者们，他们需要足够肯定他们对于效率和不存在写偏的情况而有意识地选择使用它。在一个轻微竞争的系统中，我们的 SSI 实现就像 SI 一样高效，不需要所或者条件写。在竞争的情况下，我们的 SII 实现仍旧不需要所，但是会导致退出更多的事务。CockroachDB 的 SI 和 SSI 实现避免了饥饿的场景，甚至对于任意长的事务也是如此。

一个可能的 SSI 实现，见 [Cahill 的论文](https://drive.google.com/file/d/0B9GCVTp_FHJIcEVyZVdDWEpYYXVVbFVDWElrYUV0NHFhU2Fv/edit?usp=sharing)。这是另一篇[伟大的论文](http://cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf)。
关于通过避免读-写冲突（相对于检测它们，称之为写-快照隔离）实现 SSI 的讨论，见
[Yabandeh 的论文](https://drive.google.com/file/d/0B9GCVTp_FHJIMjJ2U2t6aGpHLTFUVHFnMTRUbnBwc2pLa1RN/edit?usp=sharing)，这是 CockroachDB 的SSI 的更多灵感来源。

SI 和 SSI 都要求读的结果必须被保存，即，一个在比前一个读低的时间戳的键的写必须不能成功。因此，每个域维护一个有界的*内存内*缓存，从键域到被读的最后的时间戳。

大多数对于这个*时间戳缓存*的更新对应被读的键，尽管时间戳缓存也保护一些写的结果（注意是域删除），它们随后也必须填充缓存。缓存的条目首先赶出最旧的时间戳，相应更新缓存的低水位。

每个 CockroachDB 事务在开始时被分配一个随机的优先级和一个”候选时间戳“。候选时间戳是临时的时间，在那个点事务会提交，并被选为协调事务的节点的当前时钟时间。这意味着，没有冲突的事务将通常带一个以绝对时间表示的，在事务的实际工作完成之前的时间戳提交。

在一个或多个分布式节点间协调一个事务的过程中，候选时间戳可能会增长，但从不会减小。在两个隔离级别 SI 和 SSI 之间的核心差别是，前者允许事务的候选时间戳增长，而后者不会。

**混合逻辑时钟**

每个 cockroachDB 节点维护一个混合逻辑时钟（HLC），讨论见[混合逻辑时钟论文](http://www.cse.buffalo.edu/tech-reports/2014-04.pdf)。
HLC 时间使用由一个物理元素（被认为并且总是 接近本地的系统时间）和一个逻辑元素（用于区分带有同样物理元素的事件）构成的时间戳。它允许我们追踪类似于向量时间的相关事件的因果关系，但开销更小。在实践中，它的工作方式非常像其他的逻辑时钟：当事件被一个节点接收，以及当事件被发送由本地附加的 HLC 生成的时间戳，它通知本地 HLC 由发送者提供给事件的时间戳。

are received by a node, it informs the local HLC about the timestamp supplied
with the event by the sender, and when events are sent a timestamp generated by
the local HLC is attached.

关于 HLC 的更深入的描述，请阅读论文。我们的实现在
[这里](https://github.com/cockroachdb/cockroach/blob/master/pkg/util/hlc/hlc.go)。

CockroachDB 使用 HLC 时间，为一个事务选了一个时间戳。在本文档中，*时间戳*总是指 HLC 时间，它在每个节点上是个单例。
HLC 被每个节点上的每个读/写事件更新，并且 HLC 时间 >= 系统时间。在来自另一个节点的 CockroachDB 请求中收到的读/写时间戳不仅用于对操作生成版本，而且也更新节点上的 HLC。这在保证一个节点上的所有数据读/写是在一个小于下一个 HLC 时间的时间戳方面是有用的。

**事务执行流**

事务按两个阶段执行：

1. 通过选择一个可能深度涉及事务的域并写一个状态为"PENDING"的新的事务记录到一个域的保留区，从而启动该事务。与此同时，作为事务的一部分，对被写的每个数据写入一个"意向"值。这些事正常的 MVCC 值，增加了一个特殊标记（即，"intent"），表示这个值可能在事务本身提交之后被提交。另外，事务 id （唯一，而且在 txn 启动时被客户端选择）和意向值保存在一起。txn id 用于在冲突发生时指向事务记录，并用于在同样的时间戳之间的排序做出决胜局决策。每个节点返回用于写的时间戳（在没有读/写冲突的情况下，是当初的候选时间戳）；客户端从所有写时间戳中选择最大的作为最终的提交时间戳。

2. 通过更新事务记录提交事务。提交条目的值包含候选时间戳（如果需要则增长，以容纳最近的读时间戳）。注意，事务在这一点上被认为是完全提交的，而且控制被返回给客户端。

   在 SI 事务的情况下，一个被增大以容纳同时发生的写的时间戳是完全可以接受的，而且提交可以继续。然而，对于 SSI 事务，在候选和提交时间戳之间的间隔有必要重启事务。（注意：重启不同于退出 -- 键下文）。

   事务被提交后，所有的写意向通过移除"intent"标记被并行升级。在这一步之前，事务被认为是完全提交的，并且不等待返回控制给事务协调器。

如果没有冲突，这就是结束了。不需要任何事情来保证系统的正确性。

**冲突解决**

当一个读者或者写者在一个位置遇到一个它需要读或者写的意向记录或者新提交的值时，事情变得更有趣了。这是一个冲突，通常造成事务退出或者重启，取决于冲突的类型。

***事务重启:***

这是通常（也是更高效）的行为类型，而且被用于除了事务被（例如，被另一个事务）退出的情况。

实际上，这被缩减为两种情况；第一种是上面列出的：SSI 事务找到试图提交，而提交时间戳被推。第二种情况涉及一个事务频繁遇到冲突，即，它的一个读者或者写者需要需要解决冲突的数据（见下面的事务交互）。
above: An SSI transaction that finds upon attempting to commit that
its commit timestamp has been pushed. 

当一个事务重启，它改变了优先级和/或将其时间戳向前移动，这取决于绑定到冲突的数据，并开始重新使用同一个 txn id。事务的前一次运行可能写了一些写意向，这些需要在事务提交之前删除，从而不必包含为事务的一部分。这些陈旧的写意向删除在事务的再次执行期间完成，或者是隐式地写作为重新执行事务的一部分的新的意向到同样的键，或者是显式地通过清除不是重新执行事务的一部分的陈旧意向。由于大部分事务最终写到同样的键，显式清除恰好在提交事务之前运行通常是一个空操作。

***事务退出:***

在这种情况下，一个事务在读了它的事务记录后，发现它被退出了。此时，事务不能重用它的意向；它在清理意向（其他的读者和写者在遇到无主的意向时会清理）之前将控制返回给客户端，但会随后努力清理它们。下一个尝试（如果可用）用一个**新的 txn id**运行一个新的事务。

***事务交互:***

事务交互的几种情形：

- **读者遇到带有远在将来的时间戳的写意向或值**：
  这不是冲突。读者可以自由继续；毕竟，它将读到一个旧的值因而不冲突。记得写意向可能以一个比它的候选晚的时间戳提交；它从不会一个早的时间戳提交。**边注**：如果一个 SI 事务读者发现一个该读者自己的事务写的带有较新的时间戳的意向，该读者总是返回那个意向的值。

- **读者遇到带有不远的将来的较新的时间戳的写意向或值**：
  在这种情况下，我们得小心处理。较新的意向，绝对发生在我们的读的过去，如果写者的时钟在提供值的节点之前。在这种情况下，我们需要将这个值考虑在内，但是我们恰恰不知道。因此，事务重启，使用一个将来的时间戳（但记住用于限制最大时钟偏移的不确定性窗口的最大时间戳）。实际上，这个被进一步优化了；见下面"选择一个时间戳"条目下的细节。

- **读者遇到带有旧的时间戳的写意向**：
  读者必须按照意向的事务 id 找到事务的记录。如果事务已经被提交，读者可以只读取值。如果写事务还没有被提交，读者有两个选择。如果写冲突来自一个 SI 事务，读者可以*将事务的提交时间戳推到将来*（并因此不必读它）。容易去做：读者仅更新事务的提交时间戳以表明当/如果事务确实提交，它应该使用一个*至少*同样高的时间戳。而然，如果写冲突来自一个 SSI 事务，读者必须比较优先级。如果读者有更高的优先级，它将推后事务的提交时间戳（事务将注意到它的时间戳被推后了，并重启）。如果有较低或同样的优先级，它使用一个新的优先级 `max
(新的优先级，冲突的 txn 的优先级 - 1)` 重试自己。

- **读者遇到未提交的写意向**：
  如果其他写意向被一个带有较低优先级的事务写入，写者退出冲突的事务。如果写意向有较高或同样的优先级，事务以一个新的优先级 *max(新的随机优先级，冲突 txn 的优先级 - 1)* 重试；重试发生在一个短的、随机的退避间隔。

- **写者遇到较新的已提交的值**：
  已提交的值可能也有一个由已提交的事务造成的未解决的写意向。事务重启。重启时，使用同样的优先级，但是候选时间戳被移动为遇到的值的时间戳。

- **写者遇到更近的被读的键**：
  *读时间戳缓存* 咨询节点上的每个写。如果写的候选时间戳早于缓存自己的低水位（即，它的最后被驱逐的时间戳），或者，被写的键有一个晚于写的候选时间戳的读时间戳，这个晚些的时间戳值被写返回。只有当它是可序列化时，一个新的时间戳迫使一个事务重启。

**事务管理**

事务由客户端代理（SQL Azure 的叫法是网关）管理。不同于Spanner，写不被缓存，而是直接发给所有的隐含的域。
这允许事务遇到写冲突时快速退出。客户端代理追踪所有的被写键以在事务完成时异步解决所有的写意向。如果一个
事务成功提交，所有意向被升级为已提交。在事务被退出的情况下，所有被写的意向被删除。客户端代理不保证它会
解决意向。

在客户端在等待的事务被提交之前重启，无主的事务将继续"存在"，知道被另一个事务退出。事务定期心跳其事务记录，
以维持活跃度。

被带有在要求的间隔未被心跳的无主意向的读者和写者遇到的事务被退出。在事务提交之后当在异步解决完成之前代理
重启的情况下，当被未来的读者和写者遇到，而且系统不依赖于及时解决保证正确性，无主的意向被升级。

[这里](https://docs.google.com/document/d/1kBCu4sdGAnvLqpT-_2vaTbomNmX3_saayWEGYu1j7mQ/edit?usp=sharing)
是一个对带竞争的重试和对放弃的事务的退出次数的探索。

**事务记录**

最新的结构，请见 [pkg/roachpb/data.proto](https://github.com/cockroachdb/cockroach/blob/master/pkg/roachpb/data.proto)，
最佳的入口是`消息事务`。

**优势**

- 不需要可靠的节点执行以避免停滞的两阶段提交（2PC）协议。
- 用 SI 语义，读者从不会阻塞；用 SSI 语义，它们可能会退出。
- 比传统的两阶段提交协议延迟低（没有竞争），因为第二阶段只需要对事务记录的当个写，
  而不是一个对所有事务参与者的同步回合。
- 优先级避免了对任意长的事务的饥饿，并总是在竞争的事务（没有相互退出）中选取一个
  胜利者。
- 写在客户端不被缓存；写快速失败。
- 对可序列化的 SI （相对于其他的 SSI 实现）不需要读锁定开销。
- 良好选择的（即，较少随机）优先级能够灵活地给予任意事务在延迟上的概率保证（例如，
  相比于低优先级的事务，如异步调度的任务，使得 OLTP 事务退出的可能性是十分之一。

**劣势**

- 从非租约持有者复本的读仍然要求提醒租约持有者更新*读时间戳缓存*。
  Reads from non-lease holder replicas still require a ping to the lease holder
  to update the *read timestamp cache*.
- 被放弃的事务可能阻塞竞争的写者最大到心跳间隔，尽管平均等待可能被认为更短（见
  [链接里的图](https://docs.google.com/document/d/1kBCu4sdGAnvLqpT-_2vaTbomNmX3_saayWEGYu1j7mQ/edit?usp=sharing)）。
  相比于检测并重启 2PC 以释放读和写锁，这可能是相当地更长。
- 不同于其他 SI 实现的行为：没有第一个写者赢，而且短的事务并不总是快速结束。
  对 OLTP 系统的惊异元素可能是一个问题因素。
  Element of surprise for OLTP systems may be a problematic factor.
- 与两阶段锁定相比，退出会降低一个竞争系统的吞吐量。退出和重试增加了读和写的流量，
  提高了延迟并降低了吞吐量。

**选择一个时间戳**

从带有时钟偏移的分布式系统读取数据的一个关键挑战是选择一个时间戳，确保大于最后任何
被提交事务的时间戳（按绝对时间）。没有系统能够声称一致性并无法读取已经提交的数据。

达到访问一个节点的事务（或仅仅单个操作）的一致性是容易的。时间戳有节点本身分配，因此，
确保了是一个比该节点上所有现存时间戳数据都大的时间戳。

对于多个节点，使用协调事务的节点的时间戳 `t`。另外，提供一个最大的时间戳 `t+ε `
作为已经提交的数据的时间戳上限 （`ε`是最大的时钟偏移）。随着事务的进展 ，任何时间戳大于
`t` 但小于 `t+ε` 的数据读造成事务退出并以冲突的时间戳 t<sub>c</sub>，其中 t<sub>c</sub> \> t。
最大时间戳 `t+ε` 不变。这意味着，事务由于时钟的不确定性重启只能发生在一个长度为 `ε`的时间间隔。

我们使用另外一个优化以减少由于不确定性造成的重启。在重启的时候，事务不仅考虑
 t<sub>c</sub>，而且也要考虑在不确定读时的节点时间戳 t<sub>node</sub>。两个时间戳
t<sub>c</sub> 和 t<sub>node</sub> 中较大的（可能等于后者）被用于增大读时间戳。
另外，冲突的节点被标记为 “certain。随后，为了对事务中那个节点的未来的读，我们设置
 `MaxTimestamp = 读时间戳`，避免进一步的不确定性重启。

正确性来自于以下事实，我们知道，在读的时候，节点上没有任何键的版本的时间戳大于
t<sub>node</sub>。在由节点造成的重启时，如果事务遇到一个有更大时间戳的键，它知道，
按绝对时间，值在 t<sub>node</sub> 得到之后，即在非确定读之后，被写入。因此，事务
能够将读数据的一个较旧的版本向前移动（在事务的时间戳）。这样将对一个节点的时间非确定性
重启限制为最多一个。权衡是，我们可能选取一个大于最理想的时间戳（大于最大的冲突时间戳），
结果是新的冲突比较少。

我们期望重试会比较少，但是如果重试成了问题，这个假设可能需要重新考虑。注意，这个
问题不适用于历史读。一个代替的方法不要求重试，提前对所有的节点参与者做一轮，并
选择被报告的最高节点系统时间作为时间戳。然而，知道那个节点将被提前访问是困难的，
而且，可能是有限的。CockroachDB 也可能使用一个全局的时钟（谷歌使用
 [Percolator](https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Peng.pdf))，
这对于较小的、地理位置上接近的集群是可行的。

# 严格序列化（线性化）

大致来说，在 <i>strict serializability</i> （我们和 <i>linearizability</i> 互换使用) 
和 CockroachDB 的缺省隔离级别 (<i>serializable</i>) 之间的差距是，对于可线性化的事务，
因果关系被保留。即，如果一个事务（比如，为一个用户创建一个发帖）等待它前面的事务
（首先创建用户）完成，用户会希望分配给前者的逻辑时间戳大于后者的。

实际上，对于分布式数据库可能不是这样的，典型的原因是，在一个分布式系统中的时钟不是被
完美同步的，而且，”后来的“事务触碰了部分非连接，而第一个事务在上面运行，结果是，
带有非连接信息的时钟决定提交的时间戳。
and the "later" transaction touches a part disjoint from that on which the
first transaction ran, resulting in clocks with disjoint information to decide
on the commit timestamps.

实际上，在 CockroachDB 中，很多事务性的负载实际上是线性化的，尽管确切的情况过于复杂，
难以在这里列出。

因果关系通常对于很多事务是不需要的，因此，只有在它*确实*需要的时间才用是有利的。
CockroachDB 通过<i>因果令牌</i>实现了这个：当提交一个事务时，一个因果关系令牌被获取
并传递给下一个事务，保证这两个事务被分配递增的逻辑时间戳。

另外，随着更好同步的时钟成为云服务商提供的标准商品，CockroachDB 通过做与
[谷歌的 Spanner](http://research.google.com/archive/spanner.html) 基本相同的：
在提交后但在返回给客户端之前等待最大的时钟抵消完成，能够提供全局的线性化。

更深入的信息，见下面的博客帖子。

https://www.cockroachlabs.com/blog/living-without-atomic-clocks/

# 逻辑映射内容

逻辑上，映射在实际用户数据之前（这些由 SQL 子系统管理）包含一系列保留的系统键/值对。

- `\x02<key1>`: 域结尾 `\x03<key1>` 的域元数据。 这是一个 "meta1" 键。
- ...
- `\x02<keyN>`: 域结尾 `\x03<keyN>` 的域元数据。 这是一个 "meta1" 键。
- `\x03<key1>`: 域结尾 `<key1>` 的域元数据。 这是一个 "meta2" 键。
- ...
- `\x03<keyN>`: 域结尾 `<keyN>` 的域元数据。 这是一个 "meta2" 键。
- `\x04{desc,node,域,store}-idegen`: 对于各种元素类型的 ID 生成神谕。
- `\x04status-node-<varint encoded Store ID>`: 存储运行时元数据。
- `\x04tsd<key>`: 时间序列数据键。
- `<key>`: 一个用户键。实际上，这些键由 SQL 子系统管理，它使用自己的键剖析。

# 存储和存储空间

节点包含一个或多个存储。每个存储应该放在一个唯一的磁盘上。
在内部，每个存储包含一个单独的 RocksDB 实例，带有一个在节点上所有存储间共享的块缓存。
这些存储顺次有域副本的一个集合。一个域的多余一个的副本从不会被放在同样的存储甚至同一个节点上。

早期，当一个首先初始化时，不多的缺省初始域将只有一个单一的副本，但是，一旦其他的节点可用，
它们将复制直到它们达到它们期望的复制因子，缺省为 3。

区配置可以被用作控制一个域的复制因子并添加域的副本可以放在哪里是限制。当域的区配置发生变化时，
域将根据区的配置限制，增加或减少副本的数目到合适的值，并移动副本到合适的存储。

# 自修复

如果一个存储在一段时间内，缺省是 5 分钟，没有被听到（闲聊它们的描述符），集群将认为这个存储死了。 
当这种情况发生时，所有在那个存储上有副本的域被定为不可用并移除。这些域将随后向上复制它们自己到其他
可用的存储，直到达到它们期望的复制因子。如果 50% 或更多的副本同时不可用，没有法定人数而且整个域将
被认为不可用直到最少多于 50% 的副本再次可用。

# 再平衡

随着更多的数据被加入到系统中，一些存储的增长可能快于其他的存储。为了克服这一点并将总体负载发散到
整个集群，副本将在存储间移动，以维护期望的复制因子。用于执行整个再平衡的启发式包括：

- 每个存储的副本数量
- 每个存储使用的数据整体大小
- 每个存储可用的自由空间

将来，可能被考虑的其他因素包括：

- 每个存储的 cpu/网络负载
- 在查询中一起使用的域
- 每个存储的活跃域的数量
- 每个存储保有的域租约数量

# 域元数据

域的缺省近似大小是 is 64M (2\^26 B)。为了支持 1P (2\^50 B) 逻辑数据，需要大约 
2\^(50 - 26) = 2\^24 个域的元数据。域元数据大小的一个合理上限大约是 256 字节 
(3\*12 字节用于三倍的节点位置，220 字节用于域键本身)。2\^24 个域 \* 2\^8 B 将
需要大约 4G (2\^32 B) 用于存放 -- 在机器之间复制太大了。结论是，对于大的安装，域
元数据必须被分散。

为了让键检索在分散的元数据的情况下保持相对快速，我们将所有的顶级元数据存放在一个单一的
域(第一个域)中。这些顶级元数据键被称为 *meta1* 键，并加上前缀，使得它们在键空间中排在
开始的位置。由于上面给出的 256 字节的元数据大小，单一的 64M 域将支持 64M/256B = 2\^18 
个域，这总共是 64M \* 2\^18 = 16T 的存储空间。为了支持上面提到的 1P，我们需要两级间接
寻址，其中，第一级寻址第二级，第二级寻址用户数据。使用两级间接寻址，我们可以寻址 
2\^(18 + 18) = 2\^36 个域；每个域寻址 2\^26 B，总共我们可以寻址 
2\^(36+26) B = 2\^62 B = 4E 用户数据。

对于一个给定的用户可寻址 `key1`，相关的 *meta1* 记录在 *meta1* 空间中的 `key1` 的后继键找到。因为，*meta1* 空间是稀疏的，后继键被定义为存在的下一个键。*meta1* 记录确定包含 *meta2* 
记录的域，它使用同样的过程找到。*meta2* 记录确定包含 `key1` 的域，它又以同样的方式找到（见
下面的例子中）。

具体来说，元数据键被加上前缀 `\x02` (meta1) 和 `\x03` (meta2)；前缀 `\x02` 和 `\x03` 提供了期望的排序行为。因此， `key1`'s *meta1* 记录将位于 `\x02<key1>` 的后继键。

注：我们将每个域的结尾键附加给 meta{1,2} 记录，因为 RocksDB 迭代器只支持一个 Seek() 界面，
它是作为一个 Ceil()。使用域的开始键将造成 Seek() 寻找在我们正在寻找的索引记录的元数据*之后*的
键，这将导致不得不备份迭代器，这是一个既低效也不是在所有情况下可用的选择。

下面的例子展示了有三个域数据的映射的目录结构。
directory structure for a map with three ranges worth of data. 

省略号表示填充整个数据域的另外的键/值对。为了清晰，例子使用 `meta1` 和 `meta2` 指前缀 `\x02` 和 `\x03`。除了分裂域需要知道元数据布局的域元数据更新的事实外，域元数据本身不需要特别的处理或引导。

**域 0** (位于服务器 `dcrama1:8000`, `dcrama2:8000`, `dcrama3:8000`)

- `meta1\xff`: `dcrama1:8000`, `dcrama2:8000`, `dcrama3:8000`
- `meta2<lastkey0>`: `dcrama1:8000`, `dcrama2:8000`, `dcrama3:8000`
- `meta2<lastkey1>`: `dcrama4:8000`, `dcrama5:8000`, `dcrama6:8000`
- `meta2\xff`: `dcrama7:8000`, `dcrama8:8000`, `dcrama9:8000`
- ...
- `<lastkey0>`: `<lastvalue0>`

**域 1** (位于服务器 `dcrama4:8000`, `dcrama5:8000`, `dcrama6:8000`)

- ...
- `<lastkey1>`: `<lastvalue1>`

**域 2** (位于服务器 `dcrama7:8000`, `dcrama8:8000`, `dcrama9:8000`)

- ...
- `<lastkey2>`: `<lastvalue2>`

看一个简单些的例子，一个包含少于一个单个数据域的映射。在这个例子中，所有的域元数据和所有的数据
位于同一个域中：

**域 0** (位于服务器  `dcrama1:8000`, `dcrama2:8000`, `dcrama3:8000`)*

- `meta1\xff`: `dcrama1:8000`, `dcrama2:8000`, `dcrama3:8000`
- `meta2\xff`: `dcrama1:8000`, `dcrama2:8000`, `dcrama3:8000`
- `<key0>`: `<value0>`
- `...`

最后，一个足够大需要两级间接寻址的映射会是这样的 (注意这个例子简化了只显示域索引，而没有显示域副本)：

**域 0**

- `meta1<lastkeyN-1>`: 域 0
- `meta1\xff`: 域 1
- `meta2<lastkey1>`:  域 1
- `meta2<lastkey2>`:  域 2
- `meta2<lastkey3>`:  域 3
- ...
- `meta2<lastkeyN-1>`: 域 262143

**域 1**

- `meta2<lastkeyN>`: 域 262144
- `meta2<lastkeyN+1>`: 域 262145
- ...
- `meta2\xff`: 域 500,000
- ...
- `<lastkey1>`: `<lastvalue1>`

**域 2**

- ...
- `<lastkey2>`: `<lastvalue2>`

**域 3**

- ...
- `<lastkey3>`: `<lastvalue3>`

**域 262144**

- ...
- `<lastkeyN>`: `<lastvalueN>`

**域 262145**

- ...
- `<lastkeyN+1>`: `<lastvalueN+1>`

注意，选择 域 `262144` 仅仅是一个近似值。通过一个单个元数据域可寻址的域的实际数量取决于键的大小。如果试图保持小的键尺寸，可寻址的域总数会增加，反之亦然。

从上面的例子中可以清楚看到，键位置检索需要最多三次读以得到 `<key>` 的值：

1. `meta1<key>` 的下限
2. `meta2<key>` 的下限，
3. `<key>`。

对于小的映射，整个查询在一次对域 0 的 RPC 完成。包含少于 16T 数据的映射将需要两次查询。客户端缓存两级的域元数据，我们期望对于各个客户端的数据本地性将会是高的。客户端可能会留有陈旧的缓存内容。如果在一次查询中，被咨询的域没有满足客户端的期望，客户端驱逐了陈旧的缓存内容并可能进行一次新的查询。

# Raft - 域副本的一致性

Each 域 is configured to consist of three or more replicas, as specified by
their ZoneConfig. The replicas in a 域 maintain their own instance of a
distributed consensus algorithm. We use the [*Raft consensus algorithm*](https://raftconsensus.github.io)
as it is simpler to reason about and includes a reference implementation
covering important details.
[ePaxos](https://www.cs.cmu.edu/~dga/papers/epaxos-sosp2013.pdf) has
promising performance characteristics for WAN-distributed replicas, but
it does not guarantee a consistent ordering between replicas.

Raft elects a relatively long-lived leader which must be involved to
propose commands. It heartbeats followers periodically and keeps their logs
replicated. In the absence of heartbeats, followers become candidates
after randomized election timeouts and proceed to hold new leader
elections. Cockroach weights random timeouts such that the replicas with
shorter round trip times to peers are more likely to hold elections
first (not implemented yet). Only the Raft leader may propose commands;
followers will simply relay commands to the last known leader.

Our Raft implementation was developed together with CoreOS, but adds an extra
layer of optimization to account for the fact that a single Node may have
millions of consensus groups (one for each 域). Areas of optimization
are chiefly coalesced heartbeats (so that the number of nodes dictates the
number of heartbeats as opposed to the much larger number of 域s) and
batch processing of requests.
Future optimizations may include two-phase elections and quiescent 域s
(i.e. stopping traffic completely for inactive 域s).

# 域租约

As outlined in the Raft section, the replicas of a 域 are organized as a
Raft group and execute commands from their shared commit log. Going through
Raft is an expensive operation though, and there are tasks which should only be
carried out by a single replica at a time (as opposed to all of them).
In particular, it is desirable to serve authoritative reads from a single
Replica (ideally from more than one, but that is far more difficult).

For these reasons, Cockroach introduces the concept of **域租约**:
This is a lease held for a slice of (database, i.e. hybrid logical) time.
A replica establishes itself as owning the lease on a 域 by committing
a special lease acquisition log entry through raft. The log entry contains
the replica node's epoch from the node liveness table--a system
table containing an epoch and an expiration time for each node. A node is
responsible for continuously updating the expiration time for its entry
in the liveness table. Once the lease has been committed through raft
the replica becomes the lease holder as soon as it applies the lease
acquisition command, guaranteeing that when it uses the lease it has
already applied all prior writes on the replica and can see them locally.

To prevent two nodes from acquiring the lease, the requestor includes a copy
of the lease that it believes to be valid at the time it requests the lease.
If that lease is still valid when the new lease is applied, it is granted,
or another lease is granted in the interim and the requested lease is
ignored. A lease can move from node A to node B only after node A's
liveness record has expired and its epoch has been incremented.

Note: 域 leases for 域s within the node liveness table keyspace and
all 域s that precede it, including meta1 and meta2, are not managed using
the above mechanism to prevent circular dependencies.

A replica holding a lease at a specific epoch can use the lease as long as
the node epoch hasn't changed and the expiration time hasn't passed.
The replica holding the lease may satisfy reads locally, without incurring the
overhead of going through Raft, and is in charge or involved in handling
域-specific maintenance tasks such as splitting, merging and rebalancing

All Reads and writes are generally addressed to the replica holding
the lease; if none does, any replica may be addressed, causing it to try
to obtain the lease synchronously. Requests received by a non-lease holder
(for the HLC timestamp specified in the request's header) fail with an
error pointing at the replica's last known lease holder. These requests
are retried transparently with the updated lease by the gateway node and
never reach the client.

Since reads bypass Raft, a new lease holder will, among other things, ascertain
that its timestamp cache does not report timestamps smaller than the previous
lease holder's (so that it's compatible with reads which may have occurred on
the former lease holder). This is accomplished by letting leases enter
a <i>stasis period</i> (which is just the expiration minus the maximum clock
offset) before the actual expiration of the lease, so that all the next lease
holder has to do is set the low water mark of the timestamp cache to its
new lease's start time.

As a lease enters its stasis period, no more reads or writes are served, which
is undesirable. However, this would only happen in practice if a node became
unavailable. In almost all practical situations, no unavailability results
since leases are usually long-lived (and/or eagerly extended, which can avoid
the stasis period) or proactively transferred away from the lease holder, which
can also avoid the stasis period by promising not to serve any further reads
until the next lease goes into effect.

## 与 Raft 领导地位共存

The 域租约 is completely separate from Raft leadership, and so without
further efforts, Raft leadership and the 域 lease might not be held by the
same Replica. Since it's expensive to not have these two roles colocated (the
lease holder has to forward each proposal to the leader, adding costly RPC
round-trips), each lease renewal or transfer also attempts to colocate them.
In practice, that means that the mismatch is rare and self-corrects quickly.

## 命令执行流

This subsection describes how a lease holder replica processes a
read/write command in more details. Each command specifies (1) a key
(or a 域 of keys) that the command accesses and (2) the ID of a
域 which the key(s) belongs to. When receiving a command, a node
looks up a 域 by the specified 域 ID and checks if the 域 is
still responsible for the supplied keys. If any of the keys do not
belong to the 域, the node returns an error so that the client will
retry and send a request to a correct 域.

When all the keys belong to the 域, the node attempts to
process the command. If the command is an inconsistent read-only
command, it is processed immediately. If the command is a consistent
read or a write, the command is executed when both of the following
conditions hold:

- The 域 replica has a 域 lease.
- There are no other running commands whose keys overlap with
the submitted command and cause read/write conflict.

When the first condition is not met, the replica attempts to acquire
a lease or returns an error so that the client will redirect the
command to the current lease holder. The second condition guarantees that
consistent read/write commands for a given key are sequentially
executed.

When the above two conditions are met, the lease holder replica processes the
command. Consistent reads are processed on the lease holder immediately.
Write commands are committed into the Raft log so that every replica
will execute the same commands. All commands produce deterministic
results so that the 域 replicas keep consistent states among them.

When a write command completes, all the replica updates their response
cache to ensure idempotency. When a read command completes, the lease holder
replica updates its timestamp cache to keep track of the latest read
for a given key.

There is a chance that a 域 lease gets expired while a command is
executed. Before executing a command, each replica checks if a replica
proposing the command has a still lease. When the lease has been
expired, the command will be rejected by the replica.


# 分裂/合并域

Nodes split or merge 域s based on whether they exceed maximum or
minimum thresholds for capacity or load. 域s exceeding maximums for
either capacity or load are split; 域s below minimums for *both*
capacity and load are merged.

域s maintain the same accounting statistics as accounting key
prefixes. These boil down to a time series of data points with minute
granularity. Everything from number of bytes to read/write queue sizes.
Arbitrary distillations of the accounting stats can be determined as the
basis for splitting / merging. Two sensible metrics for use with
split/merge are 域 size in bytes and IOps. A good metric for
rebalancing a replica from one node to another would be total read/write
queue wait times. These metrics are gossipped, with each 域 / node
passing along relevant metrics if they’re in the bottom or top of the
域 it’s aware of.

A 域 finding itself exceeding either capacity or load threshold
splits. To this end, the 域 lease holder computes an appropriate split key
candidate and issues the split through Raft. In contrast to splitting,
merging requires a 域 to be below the minimum threshold for both
capacity *and* load. A 域 being merged chooses the smaller of the
域s immediately preceding and succeeding it.

Splitting, merging, rebalancing and recovering all follow the same basic
algorithm for moving data between roach nodes. New target replicas are
created and added to the replica set of source 域. Then each new
replica is brought up to date by either replaying the log in full or
copying a snapshot of the source replica data and then replaying the log
from the timestamp of the snapshot to catch up fully. Once the new
replicas are fully up to date, the 域 metadata is updated and old,
source replica(s) deleted if applicable.

**Coordinator** (lease holder replica)

```
if splitting
  Split域(split_key): splits happen locally on 域 replicas and
  only after being completed locally, are moved to new target replicas.
else if merging
  Choose new replicas on same servers as target 域 replicas;
  add to replica set.
else if rebalancing || recovering
  Choose new replica(s) on least loaded servers; add to replica set.
```

**New Replica**

*Bring replica up to date:*

```
if all info can be read from replicated log
  copy replicated log
else
  snapshot source replica
  send successive Read域 requests to source replica
  referencing snapshot

if merging
  combine 域s on all replicas
else if rebalancing || recovering
  remove old 域 replica(s)
```

Nodes split 域s when the total data in a 域 exceeds a
configurable maximum threshold. Similarly, 域s are merged when the
total data falls below a configurable minimum threshold.

**TBD: flesh this out**: Especially for merges (but also rebalancing) we have a
域 disappearing from the local node; that 域 needs to disappear
gracefully, with a smooth handoff of operation to the new owner of its data.

域s are rebalanced if a node determines its load or capacity is one
of the worst in the cluster based on gossipped load stats. A node with
spare capacity is chosen in the same datacenter and a special-case split
is done which simply duplicates the data 1:1 and resets the 域
configuration metadata.

# 节点分配（通过 Gossip）

New nodes must be allocated when a 域 is split. Instead of requiring
every node to know about the status of all or even a large number
of peer nodes --or-- alternatively requiring a specialized curator or
master with sufficiently global knowledge, we use a gossip protocol to
efficiently communicate only interesting information between all of the
nodes in the cluster. What’s interesting information? One example would
be whether a particular node has a lot of spare capacity. Each node,
when gossiping, compares each topic of gossip to its own state. If its
own state is somehow “more interesting” than the least interesting item
in the topic it’s seen recently, it includes its own state as part of
the next gossip session with a peer node. In this way, a node with
capacity sufficiently in excess of the mean quickly becomes discovered
by the entire cluster. To avoid piling onto outliers, nodes from the
high capacity set are selected at random for allocation.

The gossip protocol itself contains two primary components:

- **Peer Selection**: each node maintains up to N peers with which it
  regularly communicates. It selects peers with an eye towards
  maximizing fanout. A peer node which itself communicates with an
  array of otherwise unknown nodes will be selected over one which
  communicates with a set containing significant overlap. Each time
  gossip is initiated, each nodes’ set of peers is exchanged. Each
  node is then free to incorporate the other’s peers as it sees fit.
  To avoid any node suffering from excess incoming requests, a node
  may refuse to answer a gossip exchange. Each node is biased
  towards answering requests from nodes without significant overlap
  and refusing requests otherwise.

  Peers are efficiently selected using a heuristic as described in
  [Agarwal & Trachtenberg (2006)](https://drive.google.com/file/d/0B9GCVTp_FHJISmFRTThkOEZSM1U/edit?usp=sharing).

  **TBD**: how to avoid partitions? Need to work out a simulation of
  the protocol to tune the behavior and see empirically how well it
  works.

- **Gossip Selection**: what to communicate. Gossip is divided into
  topics. Load characteristics (capacity per disk, cpu load, and
  state [e.g. draining, ok, failure]) are used to drive node
  allocation. 域 statistics (域 read/write load, missing
  replicas, unavailable 域s) and network topology (inter-rack
  bandwidth/latency, inter-datacenter bandwidth/latency, subnet
  outages) are used for determining when to split 域s, when to
  recover replicas vs. wait for network connectivity, and for
  debugging / sysops. In all cases, a set of minimums and a set of
  maximums is propagated; each node applies its own view of the
  world to augment the values. Each minimum and maximum value is
  tagged with the reporting node and other accompanying contextual
  information. Each topic of gossip has its own protobuf to hold the
  structured data. The number of items of gossip in each topic is
  limited by a configurable bound.

  For efficiency, nodes assign each new item of gossip a sequence
  number and keep track of the highest sequence number each peer
  node has seen. Each round of gossip communicates only the delta
  containing new items.

# 节点和集群度量

Every component of the system is responsible for exporting interesting
metrics about itself. These could be histograms, throughput counters, or
gauges.

These metrics are exported for external monitoring systems (such as Prometheus)
via a HTTP endpoint, but CockroachDB also implements an internal timeseries
database which is stored in the replicated key-value map.

Time series are stored at Store granularity and allow the admin dashboard
to efficiently gain visibility into a universe of information at the Cluster,
Node or Store level. A [periodic background process](RFCS/time_series_culling.md)
culls older timeseries data, downsampling and eventually discarding it.

# 键前缀会计和区

Arbitrarily fine-grained accounting is specified via
key prefixes. Key prefixes can overlap, as is necessary for capturing
hierarchical relationships. For illustrative purposes, let’s say keys
specifying rows in a set of databases have the following format:

`<db>:<table>:<primary-key>[:<secondary-key>]`

In this case, we might collect accounting with
key prefixes:

`db1`, `db1:user`, `db1:order`,

Accounting is kept for the entire map by default.

## 会计
to keep accounting for a 域 defined by a key prefix, an entry is created in
the accounting system table. The format of accounting table keys is:

`\0acct<key-prefix>`

In practice, we assume each node is capable of caching the
entire accounting table as it is likely to be relatively small.

Accounting is kept for key prefix 域s with eventual consistency for
efficiency. There are two types of values which comprise accounting:
counts and occurrences, for lack of better terms. Counts describe
system state, such as the total number of bytes, rows,
etc. Occurrences include transient performance and load metrics. Both
types of accounting are captured as time series with minute
granularity. The length of time accounting metrics are kept is
configurable. Below are examples of each type of accounting value.

**System State Counters/Performance**

- Count of items (e.g. rows)
- Total bytes
- Total key bytes
- Total value length
- Queued message count
- Queued message total bytes
- Count of values \< 16B
- Count of values \< 64B
- Count of values \< 256B
- Count of values \< 1K
- Count of values \< 4K
- Count of values \< 16K
- Count of values \< 64K
- Count of values \< 256K
- Count of values \< 1M
- Count of values \> 1M
- Total bytes of accounting


**Load Occurrences**

- Get op count
- Get total MB
- Put op count
- Put total MB
- Delete op count
- Delete total MB
- Delete 域 op count
- Delete 域 total MB
- Scan op count
- Scan op MB
- Split count
- Merge count

Because accounting information is kept as time series and over many
possible metrics of interest, the data can become numerous. Accounting
data are stored in the map near the key prefix described, in order to
distribute load (for both aggregation and storage).

Accounting keys for system state have the form:
`<key-prefix>|acctd<metric-name>*`. Notice the leading ‘pipe’
character. It’s meant to sort the root level account AFTER any other
system tables. They must increment the same underlying values as they
are permanent counts, and not transient activity. Logic at the
node takes care of snapshotting the value into an appropriately
suffixed (e.g. with timestamp hour) multi-value time series entry.

Keys for perf/load metrics:
`<key-prefix>acctd<metric-name><hourly-timestamp>`.

`<hourly-timestamp>`-suffixed accounting entries are multi-valued,
containing a varint64 entry for each minute with activity during the
specified hour.

To efficiently keep accounting over large key 域s, the task of
aggregation must be distributed. If activity occurs within the same
域 as the key prefix for accounting, the updates are made as part
of the consensus write. If the 域s differ, then a message is sent
to the parent 域 to increment the accounting. If upon receiving the
message, the parent 域 also does not include the key prefix, it in
turn forwards it to its parent or left child in the balanced binary
tree which is maintained to describe the 域 hierarchy. This limits
the number of messages before an update is visible at the root to `2*log N`,
where `N` is the number of 域s in the key prefix.

## 区
zones are stored in the map with keys prefixed by
`\0zone` followed by the key prefix to which the zone
configuration applies. Zone values specify a protobuf containing
the datacenters from which replicas for 域s which fall under
the zone must be chosen.

Please see [pkg/config/config.proto](https://github.com/cockroachdb/cockroach/blob/master/pkg/config/config.proto) for up-to-date data structures used, the best entry point being `message ZoneConfig`.

If zones are modified in situ, each node verifies the
existing zones for its 域s against the zone configuration. If
it discovers differences, it reconfigures 域s in the same way
that it rebalances away from busy nodes, via special-case 1:1
split to a duplicate 域 comprising the new configuration.

# SQL

Each node in a cluster can accept SQL client connections. CockroachDB
supports the PostgreSQL wire protocol, to enable reuse of native
PostgreSQL client drivers. Connections using SSL and authenticated
using client certificates are supported and even encouraged over
unencrypted (insecure) and password-based connections.

Each connection is associated with a SQL session which holds the
server-side state of the connection. Over the lifespan of a session
the client can send SQL to open/close transactions, issue statements
or queries or configure session parameters, much like with any other
SQL database.

## 语言支持

CockroachDB also attempts to emulate the flavor of SQL supported by
PostgreSQL, although it also diverges in significant ways:

- CockroachDB exclusively implements MVCC-based consistency for
  transactions, and thus only supports SQL's isolation levels SNAPSHOT
  and SERIALIZABLE.  The other traditional SQL isolation levels are
  internally mapped to either SNAPSHOT or SERIALIZABLE.

- CockroachDB implements its own [SQL type system](RFCS/typing.md)
  which only supports a limited form of implicit coercions between
  types compared to PostgreSQL. The rationale is to keep the
  implementation simple and efficient, capitalizing on the observation
  that 1) most SQL code in clients is automatically generated with
  coherent typing already and 2) existing SQL code for other databases
  will need to be massaged for CockroachDB anyways.

## SQL 架构

Client connections over the network are handled in each node by a
pgwire server process (goroutine). This handles the stream of incoming
commands and sends back responses including query/statement results.
The pgwire server also handles pgwire-level prepared statements,
binding prepared statements to arguments and looking up prepared
statements for execution.

Meanwhile the state of a SQL connection is maintained by a Session
object and a monolithic `planner` object (one per connection) which
coordinates execution between the session, the current SQL transaction
state and the underlying KV store.

Upon receiving a query/statement (either directly or via an execute
command for a previously prepared statement) the pgwire server forwards
the SQL text to the `planner` associated with the connection. The SQL
code is then transformed into a SQL query plan.
The query plan is implemented as a tree of objects which describe the
high-level data operations needed to resolve the query, for example
"join", "index join", "scan", "group", etc.

The query plan objects currently also embed the run-time state needed
for the execution of the query plan. Once the SQL query plan is ready,
methods on these objects then carry the execution out in the fashion
of "generators" in other programming languages: each node *starts* its
children nodes and from that point forward each child node serves as a
*generator* for a stream of result rows, which the parent node can
consume and transform incrementally and present to its own parent node
also as a generator.

The top-level planner consumes the data produced by the top node of
the query plan and returns it to the client via pgwire.

## SQL 模型和KV间的数据映射

Every SQL table has a primary key in CockroachDB. (If a table is created
without one, an implicit primary key is provided automatically.)
The table identifier, followed by the value of the primary key for
each row, are encoded as the *prefix* of a key in the underlying KV
store.

Each remaining column or *column family* in the table is then encoded
as a value in the underlying KV store, and the column/family identifier
is appended as *suffix* to the KV key.

For example:

- after table `customers` is created in a database `mydb` with a
primary key column `name` and normal columns `address` and `URL`, the KV pairs
to store the schema would be:

| Key                          | Values |
| ---------------------------- | ------ |
| `/system/databases/mydb/id`  | 51     |
| `/system/tables/customer/id` | 42     |
| `/system/desc/51/42/address` | 69     |
| `/system/desc/51/42/url`     | 66     |

(The numeric values on the right are chosen arbitrarily for the
example; the structure of the schema keys on the left is simplified
for the example and subject to change.)  Each database/table/column
name is mapped to a spontaneously generated identifier, so as to
simplify renames.

Then for a single row in this table:

| Key               | Values                           |
| ----------------- | -------------------------------- |
| `/51/42/Apple/69` | `1 Infinite Loop, Cupertino, CA` |
| `/51/42/Apple/66` | `http://apple.com/`              |

Each key has the table prefix `/51/42` followed by the primary key
prefix `/Apple` followed by the column/family suffix (`/66`,
`/69`). The KV value is directly encoded from the SQL value.

Efficient storage for the keys is guaranteed by the underlying RocksDB engine
by means of prefix compression.

Finally, for SQL indexes, the KV key is formed using the SQL value of the
indexed columns, and the KV value is the KV key prefix of the rest of
the indexed row.

## 分布式 SQL

Dist-SQL is a new execution framework being developed as of Q3 2016 with the
goal of distributing the processing of SQL queries.
See the [Distributed SQL
RFC](RFCS/distributed_sql.md)
for a detailed design of the subsystem; this section will serve as a summary.

Distributing the processing is desirable for multiple reasons:
- Remote-side filtering: when querying for a set of rows that match a filtering
  expression, instead of querying all the keys in certain 域s and processing
  the filters after receiving the data on the gateway node over the network,
  we'd like the filtering expression to be processed by the lease holder or
  remote node, saving on network traffic and related processing.
- For statements like `UPDATE .. WHERE` and `DELETE .. WHERE` we want to
  perform the query and the updates on the node which has the data (as opposed
  to receiving results at the gateway over the network, and then performing the
  update or deletion there, which involves additional round-trips).
- Parallelize SQL computation: when significant computation is required, we
  want to distribute it to multiple node, so that it scales with the amount of
  data involved. This applies to `JOIN`s, aggregation, sorting.

The approach we took  was originally inspired by
[Sawzall](https://cloud.google.com/dataflow/model/programming-model) - a
project by Rob Pike et al. at Google that proposes a "shell" (high-level
language interpreter) to ease the exploitation of MapReduce. It provides a
clear separation between "local" processes which process a limited amount of
data and distributed computations, which are abstracted away behind a
restricted set of conceptual constructs.

To run SQL statements in a distributed fashion, we introduce a couple of concepts:
- _logical plan_ - similar on the surface to the `planNode` tree described in
  the [SQL](#sql) section, it represents the abstract (non-distributed) data flow
  through computation stages.
- _physical plan_ - a physical plan is conceptually a mapping of the _logical
  plan_ nodes to CockroachDB nodes. Logical plan nodes are replicated and
  specialized depending on the cluster topology. The components of the physical
  plan are scheduled and run on the cluster.

## 逻辑规划

The logical plan is made up of _aggregators_. Each _aggregator_ consumes an
_input stream_ of rows (or multiple streams for joins) and produces an _output
stream_ of rows. Both the input and the output streams have a set schema. The
streams are a logical concept and might not map to a single data stream in the
actual computation. Aggregators will be potentially distributed when converting
the *logical plan* to a *physical plan*; to express what distribution and
parallelization is allowed, an aggregator defines a _grouping_ on the data that
flows through it, expressing which rows need to be processed on the same node
(this mechanism constraints rows matching in a subset of columns to be
processed on the same node). This concept is useful for aggregators that need
to see some set of rows for producing output - e.g. the SQL aggregation
functions. An aggregator with no grouping is a special but important case in
which we are not aggregating multiple pieces of data, but we may be filtering,
transforming, or reordering individual pieces of data.

Special **table reader** aggregators with no inputs are used as data sources; a
table reader can be configured to output only certain columns, as needed.
A special **final** aggregator with no outputs is used for the results of the
query/statement.

To reflect the result ordering that a query has to produce, some aggregators
(`final`, `limit`) are configured with an **ordering requirement** on the input
stream (a list of columns with corresponding ascending/descending
requirements). Some aggregators (like `table readers`) can guarantee a certain
ordering on their output stream, called an **ordering guarantee**. All
aggregators have an associated **ordering characterization** function
`ord(input_order) -> output_order` that maps `input_order` (an ordering
guarantee on the input stream) into `output_order` (an ordering guarantee for
the output stream) - meaning that if the rows in the input stream are ordered
according to `input_order`, then the rows in the output stream will be ordered
according to `output_order`.

The ordering guarantee of the table readers along with the characterization
functions can be used to propagate ordering information across the logical plan.
When there is a mismatch (an aggregator has an ordering requirement that is not
matched by a guarantee), we insert a **sorting aggregator**.

### 聚合子类型

- `TABLE READER` is a special aggregator, with no input stream. It's configured
  with spans of a table or index and the schema that it needs to read.
  Like every other aggregator, it can be configured with a programmable output
  filter.
- `JOIN` performs a join on two streams, with equality constraints between
  certain columns. The aggregator is grouped on the columns that are
  constrained to be equal.
- `JOIN READER` performs point-lookups for rows with the keys indicated by the
  input stream. It can do so by performing (potentially remote) KV reads, or by
  setting up remote flows.
- `SET OPERATION` takes several inputs and performs set arithmetic on them
  (union, difference).
- `AGGREGATOR` is the one that does "aggregation" in the SQL sense. It groups
  rows and computes an aggregate for each group. The group is configured using
  the group key. `AGGREGATOR` can be configured with one or more aggregation
  functions:
  - `SUM`
  - `COUNT`
  - `COUNT DISTINCT`
  - `DISTINCT`

  An optional output filter has access to the group key and all the
  aggregated values (i.e. it can use even values that are not ultimately
  outputted).
- `SORT` sorts the input according to a configurable set of columns.
  This is a no-grouping aggregator, hence it can be distributed arbitrarily to
  the data producers. This means that it doesn't produce a global ordering,
  instead it just guarantees an intra-stream ordering on each physical output
  streams). The global ordering, when needed, is achieved by an input
  synchronizer of a grouped processor (such as `LIMIT` or `FINAL`).
- `LIMIT` is a single-group aggregator that stops after reading so many input
  rows.
- `FINAL` is a single-group aggregator, scheduled on the gateway, that collects
  the results of the query. This aggregator will be hooked up to the pgwire
  connection to the client.

## 物理规划

Logical plans are transformed into physical plans in a *physical planning
phase*. See the [corresponding
section](RFCS/distributed_sql.md#from-logical-to-physical) of the Distributed SQL RFC
for details.  To summarize, each aggregator is planned as one or more
*processors*, which we distribute starting from the data layout - `TABLE
READER`s have multiple instances, split according to the 域s - each instance
is planned on the lease holder of the relevant 域. From that point on,
subsequent processors are generally either colocated with their inputs, or
planned as singletons, usually on the final destination node.

### 处理器

When turning a _logical plan_ into a _physical plan_, its nodes are turned into
_processors_. Processors are generally made up of three components:

![Processor](RFCS/images/distributed_sql_processor.png?raw=true "Processor")

1. The *input synchronizer* merges the input streams into a single stream of
   data. Types:
   * single-input (pass-through)
   * unsynchronized: passes rows from all input streams, arbitrarily
     interleaved.
   * ordered: the input physical streams have an ordering guarantee (namely the
     guarantee of the corresponding logical stream); the synchronizer is careful
     to interleave the streams so that the merged stream has the same guarantee.

2. The *data processor* core implements the data transformation or aggregation
   logic (and in some cases performs KV operations).

3. The *output router* splits the data processor's output to multiple streams;
   types:
   * single-output (pass-through)
   * mirror: every row is sent to all output streams
   * hashing: each row goes to a single output stream, chosen according
     to a hash function applied on certain elements of the data tuples.
   * by 域: the router is configured with 域 information (relating to a
     certain table) and is able to send rows to the nodes that are lease holders for
     the respective 域s (useful for `JoinReader` nodes (taking index values
     to the node responsible for the PK) and `INSERT` (taking new rows to their
     lease holder-to-be)).

To illustrate with an example from the Distributed SQL RFC, the query:
```
TABLE Orders (OId INT PRIMARY KEY, CId INT, Value DECIMAL, Date DATE)

SELECT CID, SUM(VALUE) FROM Orders
  WHERE DATE > 2015
  GROUP BY CID
  ORDER BY 1 - SUM(Value)
```

produces the following logical plan:

![Logical plan](RFCS/images/distributed_sql_logical_plan.png?raw=true "Logical Plan")

This logical plan above could be transformed into either one of the following
physical plans:

![Physical plan](RFCS/images/distributed_sql_physical_plan.png?raw=true "Physical Plan")

or

![Alternate physical plan](RFCS/images/distributed_sql_physical_plan_2.png?raw=true "Alternate physical Plan")


## 执行基础设施

Once a physical plan has been generated, the system needs to divvy it up
between the nodes and send it around for execution. Each node is responsible
for locally scheduling data processors and input synchronizers. Nodes also
communicate with each other for connecting output routers to input
synchronizers through a streaming interface.

### 创建一个本地计划：`ScheduleFlows` RPC

Distributed execution starts with the gateway making a request to every node
that's supposed to execute part of the plan asking the node to schedule the
sub-plan(s) it's responsible for (except for "on-the-fly" flows, see design
doc). A node might be responsible for multiple disparate pieces of the overall
DAG - let's call each of them a *flow*. A flow is described by the sequence of
physical plan nodes in it, the connections between them (input synchronizers,
output routers) plus identifiers for the input streams of the top node in the
plan and the output streams of the (possibly multiple) bottom nodes. A node
might be responsible for multiple heterogeneous flows. More commonly, when a
node is the lease holder for multiple 域s from the same table involved in
the query, it will run a `TableReader` configured with all the spans to be
read across all the 域s local to the node.

A node therefore implements a `ScheduleFlows` RPC which takes a set of flows,
sets up the input and output [mailboxes](#mailboxes), creates the local
processors and starts their execution.

### 流的本地调度

The simplest way to schedule the different processors locally on a node is
concurrently: each data processor, synchronizer and router runs as a goroutine,
with channels between them. The channels are buffered to synchronize producers
and consumers to a controllable degree.

### 信箱

Flows on different nodes communicate with each other over gRPC streams. To
allow the producer and the consumer to start at different times,
`ScheduleFlows` creates named mailboxes for all the input and output streams.
These message boxes will hold some number of tuples in an internal queue until
a gRPC stream is established for transporting them. From that moment on, gRPC
flow control is used to synchronize the producer and consumer. A gRPC stream is
established by the consumer using the `StreamMailbox` RPC, taking a mailbox id
(the same one that's been already used in the flows passed to `ScheduleFlows`).

A diagram of a simple query using mailboxes for its execution:
![Mailboxes](RFCS/images/distributed_sql_mailboxes.png?raw=true)

## 一个复杂的例子：每日促销

To give a visual intuition of all the concepts presented, we draw the physical plan of a relatively involved query. The
point of the query is to help with a promotion that goes out daily, targeting
customers that have spent over $1000 in the last year. We'll insert into the
`DailyPromotion` table rows representing each such customer and the sum of her
recent orders.

```SQL
TABLE DailyPromotion (
  Email TEXT,
  Name TEXT,
  OrderCount INT
)

TABLE Customers (
  CustomerID INT PRIMARY KEY,
  Email TEXT,
  Name TEXT
)

TABLE Orders (
  CustomerID INT,
  Date DATETIME,
  Value INT,

  PRIMARY KEY (CustomerID, Date),
  INDEX date (Date)
)

INSERT INTO DailyPromotion
(SELECT c.Email, c.Name, os.OrderCount FROM
      Customers AS c
    INNER JOIN
      (SELECT CustomerID, COUNT(*) as OrderCount FROM Orders
        WHERE Date >= '2015-01-01'
        GROUP BY CustomerID HAVING SUM(Value) >= 1000) AS os
    ON c.CustomerID = os.CustomerID)
```

一个可能的物理计划：
![Physical plan](RFCS/images/distributed_sql_daily_promotion_physical_plan.png?raw=true)
