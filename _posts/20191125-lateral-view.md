---
title: lateral view简介
tags:
  - hive
date: 2019-11-25 15:05:43
category: CODING
---

众所周知，hive储存数据单个列可以是结合map或array的复杂形式，如果我们想将一个复杂列的某个数据单独拿出来处理

## List

例如我们有这样一张表

|id|eles|
|---|---|
|1|["a","b","c"]|
|2|["b","c","d"]|
|3|["b","e","f"]|

如果我们使用lateral view语句可以将eles的内容分隔出来

```sql
select *
from table_name
lateral view explode(eles) ex_table as ele
group by 1;
```

之后就能得到以下结果：

|id|ele|
|---|---|
|1|"a"|
|1|"b"|
|1|"c"|
|2|"a"|
|2|"b"|
|2|"c"|
|3|"a"|
|3|"b"|
|3|"c"|

之后就很容易做处理了，比如说想得到每个元素出现的个数：

```sql
select ele, count(ele) as count
from table_name
lateral view explode(eles) ex_table as ele
group by 1;
```

多个数组分离也可以类似的方法：

|id|eles1|eles2|
|---|---|---|
|1|["a","b"]|["alpha","beta"]|
|2|["b","c"]|["alpha","gama"]|
|3|["b","e"]|["beta","omega"]|

```sql
select *
from table_name
lateral view explode(eles1) ex_table1 as ele1
lateral view explode(eles2) ex_table2 as ele2;
```

就能得到结果：

|id|ele1|ele2|
|---|---|---|
|1|"a"|"alpha"|
|1|"a"|"beta"|
|1|"b"|"alpha"|
|1|"b"|"beta"|
|...（下略）|

但是如果我们想让两个数组每个元素一一对应而不是像这样弄成一个笛卡尔积怎么办呢？我们一个用posexplode

```sql
select *
from table_name
lateral view posexplode(eles1) ex_table as ele11, ele12
lateral view posexplode(eles2) ex_table as ele21, ele22
where ele11 = ele21;
```

结果是这样的：

|id|ele1|ele2|
|---|---|---|
|1|"a"|"alpha"|
|1|"b"|"beta"|
|2|"a"|"alpha"|
|2|"b"|"gama"|
|3|"a"|"beta"|
|3|"b"|"omega"|

## Map

map的处理也类似

|id|add|
|---|---|
|1|{"city":"SH","street":"Pudong Ave","code":"200000"}|
|2|{"city":"NY","street":"5th Ave","code":"90000"}|

```sql
select *
from table_name
lateral view explode(add) ex_table as k, v;
```

就能拿到以下结果

|id|k|v|
|---|---|---|
|1|"city"|"SH"|
|1|"street"|"Pudong Ave"|
|1|"code"|"200000"|
|2|"city"|"NY"|
|2|"street"|"5th Ave"|
|2|"code"|"90000"|
