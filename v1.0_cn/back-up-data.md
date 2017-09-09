# 备份数据

CockroachDB 提供了下列方法备份集群的数据：

- [`cockroach dump`](sql-dump.md)，这是一条转储/输出数据库的模式和表数据的 CLI 命令。
- [`BACKUP`](backup.md) (*仅适用于[企业授权](https://www.cockroachlabs.com/pricing/)*)，这是一条 SQL 语句，用于备份集群到云或网络文件存储。

### 细节

我们推荐创建数据的每日备份，作为操作最佳实践。

然而，由于 CockroachDB 被设计带有高容错，备份主要被用于灾难恢复（即，如果集群丢失了大部分节点）。隔离问题（比如小规模节点停止运行）不需要任何人为介入。

## 恢复

关于恢复备份数据的信息，见[恢复数据](restore-data.md)。

## 另见

- [恢复数据](restore-data.md)
- [使用内建的 SQL 客户端](use-the-built-in-sql-client.md)
- [其他 Cockroach 命令](cockroach-commands.md)