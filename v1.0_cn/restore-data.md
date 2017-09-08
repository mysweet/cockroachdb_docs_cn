# 恢复数据

如何恢复集群的数据取决于当初的[备份](back-up-data.md)类型：

备份类型 | 恢复使用
------------|-----------------
[`cockroach dump`](sql-dump.md) | [导入数据](import-data.md)
[`BACKUP`](backup.md)<br/>(*仅适用于[企业授权](https://www.cockroachlabs.com/pricing/)*) | [`RESTORE`](restore.md)

## 另见

- [备份数据](back-up-data.md)
- [使用内建的 SQL 客户端](use-the-built-in-sql-client.md)
- [其他 Cockroach 命令](cockroach-commands.md)
