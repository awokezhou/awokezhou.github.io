---
title: csv导入cassandra大量row丢失
date: 2019-05-17 13:43:58
tags: [Cassandra,csv,问题解决]
categories:
- 数据库
- cassandra
comments: true
---

在测试cassandra数据库导出/导入csv文件功能时，发现从csv文件导入所有数据到cassandra的表中后，大量行丢失。经查资料和排查发现，是因为创建表时的主键设置问题

## 问题描述
csv文件名为record_2019-01-01.csv，列结构为`[time,lineID,stationID,deviceID,status,userID,payType]`，一共7列，行数为2539593
```
[root@eca977993d07 import]# cat record_2019-01-01.csv | head -n 1
time,lineID,stationID,deviceID,status,userID,payType
[root@eca977993d07 import]#   
[root@eca977993d07 import]# wc -l record_2019-01-01.csv
2539593 record_2019-01-01.csv
```

执行cqlsh连接到本地的cassandra数据库，在名为`open_metro`的keysapce中，创建了名为`metro_train`的表
```
[root@eca977993d07 import]# cqlsh
Connected to Test Cluster at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.11.4 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
cqlsh> use open_metro ;
cqlsh:open_metro> CREATE TABLE metro_train ( time text PRIMARY KEY, lineID varchar, stationID varchar, deviceID text, status varint,  userID text, payType varint );                  
cqlsh:open_metro> SELECT * FROM metro_train ;

 time | deviceid | lineid | paytype | stationid | status | userid
------+----------+--------+---------+-----------+--------+--------

(0 rows)
cqlsh:open_metro>
```
从csv文件中导入数据到`metro_train`表中
```
cqlsh:open_metro> COPY metro_train (time, lineID, stationID, deviceID, status, userID, payType) FROM 'record_2019-01-01.csv' WITH HEADER = true;        
Using 1 child processes

Starting copy of open_metro.metro_train with columns [time, lineid, stationid, deviceid, status, userid, paytype].
Processed: 2539592 rows; Rate:    6755 rows/s; Avg. rate:   12327 rows/s
2539592 rows imported from 1 files in 3 minutes and 26.019 seconds (0 skipped).
cqlsh:open_metro>   
cqlsh:open_metro> SELECT COUNT(*) FROM metro_train ;

 count
-------
 65790

(1 rows)

Warnings :
Aggregation query used without partition key

cqlsh:open_metro>
```
由打印可以看出，的确处理了2539592行数据，和原文件行数一致，但是查看表的行数，只有65790行

## 问题排查
在StackOverflow里找到了类似的问题，[Cassandra is missing data when loading a csv with cassandra-loader
](https://stackoverflow.com/questions/51775404/cassandra-is-missing-data-when-loading-a-csv-with-cassandra-loader)
原因在于表中的主键不唯一，cassandra的COPY操作做的是插入更新操作，因为主键是time，而后续的数据中有重复的时间数据，后一个就会覆盖前一个，因此导致最终导入表中的数据行数少了很多

## 问题解决
删除metro_train表，重新创建该表，并设置为多个主键
```sql
cqlsh:open_metro> CREATE TABLE metro_train (
    time text,
    lineID varchar,
    stationID varchar,
    deviceID text,
    status varint,
    userID text,
    payType varint,
    PRIMARY KEY(time, lineID, stationID, deviceID, status, userID, payType)
);
cqlsh:open_metro> COPY metro_train (time, lineID, stationID, deviceID, status, userID, payType) FROM '/usr/local/apache-cassandra-3.11.4/import/record_2019-01-01.csv'
...
```

等待所有行都从文件加载好，查看一下表大小
```sql
cqlsh:open_metro> SELECT COUNT(*) FROM metro_train;
OperationTimedOut: errors={'127.0.0.1': 'Client request timeout. See Session.execute[_async](timeout)'}, last_host=127.0.0.1
cqlsh:open_metro>
```

这里出现超时，是因为cassandra设置的超时，关闭cqlsh，重新运行，并携带超时参数
```cmd
cqlsh --request-timeout 60000
```

重新查看表大小，为2539060
```sql
cqlsh> use open_metro ;                 
cqlsh:open_metro> SELECT COUNT(*) FROM metro_train;

 count
---------
 2539060

(1 rows)

Warnings :
Aggregation query used without partition key

cqlsh:open_metro>
```

这次数据量大致与文件行数差不多，但是还差了533行，为什么会差这么多行呢？猜测可能是因为文件本身有一些重复的行

使用sort命令查看文件重复的行
```cmd
[root@eca977993d07 import]# sort record_2019-01-01.csv |uniq -d > repeat.txt
[root@eca977993d07 import]# wc -l repeat.txt
532 repeat.txt
[root@eca977993d07 import]#
```
可以看到，record_2019-01-01.csv文件本身有532行是重复的，再加上第一行行首，刚好是cassandra的metro_train表大小和文件行数的差距
