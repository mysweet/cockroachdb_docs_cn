# SQL FAQs

##如何增大插入 CockroachDB 的数据？

一般情况下，你可以批量使用 [`INSERT`](insert.html) 语句增大插入的数据，每次插入的数据不超过几MB。行的大小决定你可以使用多少，不过通常情况下1000~10000行是最好的选择。详情请查看 [导入数据](https://www.cockroachlabs.com/docs/import-data.html)。

## 在 CockroachDB 中，如何自动得生成唯一的行 ID？

请看 [自动生成唯一ID](faq/auto-generate-unique-ids.html).

## 如何获取插入表中的最后的项的 ID 或者 序列号？

CockroachDB 中没有函数能返回最后插入的数据，但是可以使用 `INSERT` 语句的 [`RETURNING` 子句](insert.html#insert-and-return-values)。

比如，使用 `RETURNING` 返回自动生成的 [`序列号`](serial.html)：

```sql
> CREATE TABLE users (id SERIAL, name STRING);

> INSERT INTO users (name) VALUES ('mike') RETURNING id;
```

## CockroachDB 是否支持 `JOIN` ？

CockroachDB 对 `JOIN` 有基本的、未经优化的支持，我们正在努力改善他的性能。

详情请查看我们的关于 CockroachDB 的 `JOIN` 的博客：

-   [简单朴素: CockroachDB 的 JOIN](https://www.cockroachlabs.com/blog/cockroachdbs-first-join/).
-   [更好的使用 JOIN](https://www.cockroachlabs.com/blog/better-sql-joins-in-cockroachdb/)

## 如何使用交叉表？

[交叉表](interleave-in-parent.html) 通过优化密切相关的表的键值结构来改善查询性能，如果可能要同时读和写，就会试图将数据保存在相同的键值范围内。