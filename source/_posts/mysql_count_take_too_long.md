---
title: MySQL count(*) take too long
date: 2017-11-02 15:06
tags:
  - MySQL
  - Performance Tunning
---

## 前言

- 最近在做一个使用 PHP + MySQL 的项目，随着数据量的加大，页面的访问速度突然变的很慢很慢。。。
- 目前线上环境使用的 MySQL 版本是 5.7.15

## 监控服务集成

这里我使用了 [听云](http://www.tingyun.com/) 的服务做为我的性能分析工具，具体的集成步骤大家看文档就好了，这里不再赘述。

## 发现问题

集成好 [听云](http://www.tingyun.com/) 后，点了几下响应特别慢的页面，看了下听云控制台上的数据。发现一个 `SELECT COUNT(*) FROM user` 的查询就用了13秒之多，况且数据库中只有14w条数据就要花费这么长的时间。

![improve with nothing](/images/mysql-count/mysql_count_result.png)

<!-- more -->

## 解决问题

查看 MySQL 的文档发现，当使用 MySQL 的 InnoDB 存储引擎时，在执行 `COUNT(*)` 操作时，MySQL 会遍历所有行去计算总数，这样就会导致很耗时。

![mysql innodb restrictions](/images/mysql-count/mysql_innodb_restrictions.png)

文档中已经提到在 MySQL 中使用 `COUNT(*)` 和 `COUNT(1)` 是一样的效果。所以这条路已经可以不用考虑了。

通过阅读文档，可以看到有2个优化的思路:

- Create a counter table and let your application update it according to the inserts and deletes it does
- If an approximate row count is sufficient, SHOW TABLE STATUS can be used

### Counter Table

其实对于 Counter Table 的维护，可以通过使用我们的代码控制，也可以通过 MySQL 本身的一些特性来做。比如使用 MySQL 的 [Trigger](https://dev.mysql.com/doc/refman/5.7/en/trigger-syntax.html) 和 [Event Scheduler](https://dev.mysql.com/doc/refman/5.7/en/event-scheduler.html)

- Trigger

```
CREATE TRIGGER `count_up` AFTER INSERT ON `user` FOR EACH ROW UPDATE `stats`
SET
  `stats`.`value` = `stats`.`value` + 1
WHERE
  `stats`.`key` = "user_count";

CREATE TRIGGER `count_down` AFTER DELETE ON `user` FOR EACH ROW UPDATE `stats`
SET
  `stats`.`value` = `stats`.`value` - 1
WHERE
  `stats`.`key` = "user_count";
```

- Event Scheduler

```
CREATE EVENT update_stats
ON SCHEDULE
  EVERY 5 MINUTE
DO
  INSERT INTO stats (`key`, `value`)
  VALUES ('user_count', (select count(*) from user))
  ON DUPLICATE KEY UPDATE value=VALUES(value);
```

### SHOW TABLE STATUS

通过实验这个 *SHOW TABLE STATUS* 的方法看起来 **不太靠谱**。原因请看下图

![mysql count table status](/images/mysql-count/mysql_count_table_status.png)

![mysql count normal](/images/mysql-count/mysql_count_normal.png)

这个 "近似值" 相差了 3000 多，这个对我来说有点太多了。所以这条路也先不考虑了。

## Take advantage of Index

### [Clustered index VS Secondary index](https://dev.mysql.com/doc/refman/5.7/en/innodb-index-types.html)

> Prior to MySQL 5.7.18, InnoDB processes SELECT COUNT(*) statements by scanning the clustered index. As of MySQL 5.7.18, InnoDB processes SELECT COUNT(*) statements by traversing a smaller secondary index, if present.

以上内容摘自 [MySQL 官方文档](https://dev.mysql.com/doc/refman/5.7/en/innodb-restrictions.html#innodb-table-restrictions)。然而在实际测试中，发现了一些疑问:

首先我手头上有两个 MySQL 服务，一个是 5.7.15 版本，一个是 5.7.19 版本。两个数据库存储了相同数据

在使用 5.7.19 版本的 MySQL 执行 `SELECT COUNT(*) FROM user` 语句时，结果如下，同时给出了 explain 的结果

![mysql count higher version](/images/mysql-count/mysql_count_higher_version.png)

![mysql count explain higher version](/images/mysql-count/mysql_count_explain_higher_version.png)

可以看到在使用 5.7.18 及以上版本的 MySQL 执行 `COUNT(*)` 的时候，14万的数据响应时间还不到 40ms。从 Explain 的结果可以看出，MySQL 在执行 `COUNT(*)` 的时候正如文档所说的是 *traversing a smaller secondary index* 。

同样的测试在 5.7.15 版本上执行，结果如下

![mysql count lower version](/images/mysql-count/mysql_count_lower_version.png)

![mysql count lower version](/images/mysql-count/mysql_count_explain_lower_version.png)

这次执行 `COUNT(*)` 的响应时间达到了 14s，Explain 的结果显示没有 **using index**。而按照文档的描述，对于 5.7.18 版本以前的 MySQL 在执行 `COUNT(*)` 语句的时候应该会使用 clustered index，然而从 Explain 的结果和查询的响应时间上来看似乎都不是这样。

### Make MySQL count data by using index explicitly

根据索引的工作原理，猜想通过在 `SELECT COUNT(*) FROM user` 语句中加入 `WHERE` 查询条件，且查询的字段已经添加过索引，MySQL 便会先根据索引过滤数据，然后再 count 数据。
基于以上假设，在查询语句后又加上了 `WHERE id IS NOT NULL`。以下是相应的测试结果

![mysql count higher version](/images/mysql-count/mysql_count_with_where.png)

![mysql count explain higher version](/images/mysql-count/mysql_count_explain_with_where.png)

通过测试结果可以看出，在执行 `COUNT(*)` 的时候使用了索引，响应时间也减少到了几百毫秒。

## 思考

其实，上面的通过加 `WHERE` 查询条件的优化方式，随着数据的不断增长，依然会出现性能问题，毕竟时间复杂度是O(n)嘛。而使用 counter table 的方式就显得更稳定一些了。

## 参考

- https://dev.mysql.com/doc/refman/5.7/en/innodb-restrictions.html#innodb-table-restrictions
- https://stackoverflow.com/questions/19267507/how-to-optimize-count-performance-on-innodb-by-using-index
- https://dev.mysql.com/doc/refman/5.7/en/innodb-index-types.html
