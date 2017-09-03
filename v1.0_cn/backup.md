# 备份（企业级) 

将你的 CockroachDB 数据库集群备份至一个云存储服务，如 AWS S3，Google 云存储或其他NFS(网络文件系统)。

{{site.data.alerts.callout_danger}}The <code>BACKUP</code> feature is only available to <a href="https://www.cockroachlabs.com/pricing/">enterprise license</a> users. For non-enterprise backups, see <a href="sql-dump.html"><code>cockroach dump</code></a>.{{site.data.alerts.end}}

> 警告：`BACKUP` 功能只向企业版本用户开放，对于非企业级用户的需求，请查看 [`cockroach dump`](https://www.cockroachlabs.com/docs/stable/sql-dump.html) 。

CockroachDB 的 `BACKUP` [语句](https://www.cockroachlabs.com/docs/stable/sql-statements.html) 可创建与给定时间戳一致的集群模式和数据的完整或增量备份。这种备份方式可以在你现在正在使用的平台上使用，包括 AWS S3，Google 云存储，NFS 或 HTTP 存储。

因为 CockroachDB 设计有高容错的特性，这种备份方式也主要为错误恢复而设计(也就是说当集群丢失了它的大多数节点) 。孤立的问题（如小规模的节点故障）不需要任何干预。

## 功能详解

### 备份目标

你可以备份整个表（这些表自动包含索引）或视图。备份数据库只需备份它的所有表和视图。

> **注意：** `BACKUP` 备份只提供表级别的粒度；不支持备份一个表的子集。

### 对象的依赖关系

依赖对象应该与它们所依赖的对象同时备份，否则无法还原依赖对象。

| 对象                                       | 依赖                                       |
| ---------------------------------------- | ---------------------------------------- |
| 具有[外键](https://www.cockroachlabs.com/docs/stable/foreign-key.html)约束的表 | 该表所引用的表(但是，这种依赖将在[恢复期间解除](https://www.cockroachlabs.com/docs/stable/restore.html#skip_missing_foreign_keys)) |
| [视图](https://www.cockroachlabs.com/docs/stable/views.html) | 该视图上使用 `SELECT` 语句时使用的表                  |
| [交叉表](https://www.cockroachlabs.com/docs/stable/interleave-in-parent.html) | 在[交错层次](https://www.cockroachlabs.com/docs/stable/interleave-in-parent.html#interleaved-hierarchy)中使用的父表 |

### 用户和权限

每个备份都包括 `system.users`，其中保存了用户和密码。要恢复用户，必须通过这个[流程](https://www.cockroachlabs.com/docs/stable/restore.html#restoring-users-from-system-users-backup)。

恢复的表继承了目标数据库的权限而不保留备份表的权限，因为恢复的集群可能有不同的用户。

在恢复完成后，表级别权限必须被授予表级权限。

### 备份类型

CockroachDB 提供了两种类型的备份：全备份和增量备份。

#### 全备份

全备份包含一个无重复的数据拷贝并随时用于集群的恢复。这些文件大致相当于数据的大小，且需要比增量备份生成更多的资源。

#### 增量备份

增量备份比全备份更小且更快，因为他们只包含已指定的基本备份集（必须包括一个完整备份并可以包括许多增量备份）的数据。

只能在基本备份最新时间戳的垃圾收集期间内创建增量备份。这是因为增量备份是由发现数据已经被修改或创建以来最新的时间戳的基础上备份––也就是时间戳数据，哪怕是由垃圾收集过程删除的。

可以使用 `ttlseconds` 复制区域设置在每个表基础上配置垃圾收集周期。

## 性能

备份过程通过向所有节点分配工作来最小化对集群性能的影响。每个节点只备份它存储的数据的一个特定子集（它提供的写服务；敬请关注随后发布关于这个体系结构概念的更多细节），两个节点不会备份相同的数据。

为了获得最佳性能，我们还建议始终以至少10秒的时间开始特定[时间戳](https://www.cockroachlabs.com/docs/stable/timestamp.html)的备份。例如:

~~~ sql
> BACKUP...AS OF SYSTEM TIME '2017-06-09 16:13:55.571516+00:00';
~~~

因为受其他的声明/处理事务的竞争，所以可通过减少重复备份来提升。

## 自动化备份

我们建议自动化集群的每日备份。
要实现自动化备份，必须让客户机将备份语句发送到集群。
备份完成后，客户机将收到备份响应。

## 简介

![](https://github.com/TechCatsLab/cockroachdb_docs_cn/blob/master/images/backup-synopsis.png)

## 请求权限

只有 `root` 用户可以执行`备份`操作。

## 参数

| 参数                                      | 描述                                       |
| --------------------------------------- | ---------------------------------------- |
| `table_pattern`                         | 你想备份的表或视图                                |
| `name`                                  | 你想备份的数据库的名字(也就是说要备份这个库中的所有表和视图)。         |
| `destination`                           | 你想备份的 URL.<br/><br/>查看更多 URL 的细节，请看 [备份 URL 文件](https://www.cockroachlabs.com/docs/stable/backup.html#backup-file-urls)。 |
| `AS OF SYSTEM TIME timestamp`           | 备份以 [`时间戳`](timestamp.html)形式存在的数据。However，但是 `时间戳`必须比集群的最后一个垃圾回收时间（默认24小时发生一次），但这对每个表来说是可配置的。 |
| `INCREMENTAL FROM full_backup_location` | 创建一个增量备份，其中包括所提供的 URL 上列出的所有备份。<br/><br/>查看更多关于URL 的内容，请看[备份 URL 文件](https://www.cockroachlabs.com/docs/stable/backup.html#backup-file-urls)。 |
| `incremental_backup_location`           | 创建一个增量备份，其中包括在所提供的URL上列出的所有备份。<br/><br/>Lists of incremental backups must be sorted from oldest to newest. The newest incremental backup's timestamp must be within the table's garbage collection period. <br/><br/>For information about this URL structure, see [Backup File URLs](#backup-file-urls). <br/><br/>For more information about garbage collection, see [Configure Replication Zones](configure-replication-zones.html#replication-zone-format). |

### Backup File URLs

The URL for your backup's destination/locations must use the following format:

~~~
[scheme]://[host]/[path to backup]?[parameters]
~~~

`[path to backup]` must be unique for each backup, but the other values depend on where you want to store the backup.

| Backup Location      | scheme      | host                 | parameters                               |
| -------------------- | ----------- | -------------------- | ---------------------------------------- |
| Amazon S3            | `s3`        | Bucket name          | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` |
| Azure                | `azure`     | Container name       | `AZURE_ACCOUNT_KEY`, `AZURE_ACCOUNT_NAME` |
| Google Cloud Storage | `gs`        | Bucket name          | None––currently only supports instance auth, but we can build non-instance auth at a customer's request |
| HTTP                 | `http`      | Remote host          | N/A                                      |
| NFS                  | `nodelocal` | File system location | N/A                                      |

{{site.data.alerts.callout_info}}Because CockroachDB is a distributed system, you cannot meaningfully store backups "locally" on nodes. The entire backup file must be stored in a single location, so attempts to store backups locally must point to an NFS drive to be useful.{{site.data.alerts.end}}

## Examples

Per our guidance in the [Performance](#performance) section, we recommend starting backups from a time at least 10 seconds in the past using `AS OF SYSTEM TIME`.

### Backup a Single Table or View

~~~ sql
> BACKUP bank.customers TO 'azure://acme-co-backup/table-customer-2017-03-27-full?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co'
AS OF SYSTEM TIME '2017-06-09 16:13:55.571516+00:00';
~~~

### Backup Multiple Tables

~~~ sql
> BACKUP bank.customers, bank.accounts TO 'azure://acme-co-backup/tables-accounts-customers-2017-03-27-full?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co'
AS OF SYSTEM TIME '2017-06-09 16:13:55.571516+00:00');
~~~

### Backup an Entire Database

~~~ sql
> BACKUP DATABASE bank TO 'azure://acme-co-backup/database-bank-2017-03-27-full?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co'
AS OF SYSTEM TIME '2017-06-09 16:13:55.571516+00:00';
~~~

### Create Incremental Backups

Incremental backups must be based off of full backups you've already created.

~~~ sql
> BACKUP DATABASE bank TO 'azure://acme-co-backup/database-bank-2017-03-29-incremental?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co'
INCREMENTAL FROM 'azure://acme-co-backup/database-bank-2017-03-27-full?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co'
, 'azure://acme-co-backup/database-bank-2017-03-28-incremental?AZURE_ACCOUNT_KEY=hash&AZURE_ACCOUNT_NAME=acme-co'
AS OF SYSTEM TIME '2017-06-09 16:13:55.571516+00:00'
~~~

## See Also

- [`RESTORE`](restore.html)
- [Configure Replication Zones](configure-replication-zones.html)
