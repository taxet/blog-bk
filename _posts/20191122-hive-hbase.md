---
title: HBase与Hive数据互相导入
tags:
  - HBase
  - Hive
date: 2019-11-22 15:14:23
category: CODING
---

Hbase 指的是 Hadoop Database, 而Hive 指的是基于Hadoop可以使用SQL查询的数据仓库。既然两者都是基于Hadoop的，那么就一定可以互相导入数据。

## Hbase -> Hive

这个相当简单。只要在Hive中建一个外表就可以了。

```SQL
set hbase.zookeeper.quorum=your.hbase.zkq.path
create external table (...)
STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
TBLPROPERTIES ("hbase.table.name" = "name_space:hbase_table_name");
```

虽然说这里可以建普通的表，但是还是建议建一个外表，起码这个表之后有什么改动不会影响Hbase的数据。

不过很多时候我们只希望拿到Hbase里的一些数据，并不是全部。这个时候可以用WITH SERDEPROPERTIES来选选定你想提取的数据。

```SQL
set hbase.zookeeper.quorum=your.hbase.zkq.path
create external table (
  key string,
  column1 string,
  column2 int,
  column3 array<String>
) STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler'
WITH SERDEPROPERTIES ("hbase.columns.mapping" = ":key,f.c1,f.c2,g.c3")
TBLPROPERTIES ("hbase.table.name" = "name_space:hbase_table_name");
```

值得一提的是，如果要从这个外表拿数据的话必须保证和HBASE处于相同的hbase zookeeper quorum下，也就是前面要加一句 set hbase.zookeeper.quorum=your.hbase.zkq.path

## Hive -> HBase

上面的步骤转化一下思路就能实现了，我们看看要做什么：

1. Hbase里面建一个表
2. Hive里建一个映射表
3. Hive数据导入映射表
