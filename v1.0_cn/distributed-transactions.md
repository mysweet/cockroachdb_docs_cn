# 分布式事务

CockroachDB 跨集群分布[事务](transactions.md)，无论是一个单一位置的几台服务器，还是跨多个数据中心的很多服务器。与分片的设置不同，你不需要知道数据的准确位置；仅和集群中的任意节点通信就可以，而 CockroachDB 无缝地将事务放到正确的地方。有再平衡进，分布式事务处理不会有停机和额外的延迟。集群在负载下，你甚至可以在数据中心或者云基础设施提供商之间移动表或整个数据库。

-   容易构建的一致应用
-   带有分布式死锁检测的优化并发
-   可序列化默认隔离级别

<img src='../images/2distributed-transactions.png' alt="Distributed transactions in CockroachDB" style="width: 400px" />

## 另见

- [CockroachDB 如何处理分布式、原子事务](https://www.cockroachlabs.com/blog/how-cockroachdb-distributes-atomic-transactions/)
- [可序列化、无锁、分布式：CockroachDB 中的隔离](https://www.cockroachlabs.com/blog/serializable-lockless-distributed-isolation-cockroachdb/)
