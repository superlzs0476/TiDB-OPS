---
title: TiDB Select count 性能测试
date: 2018-04-06 21:40:52
updated: 2017-04-06 21:40:52
categories: 
  - deploy
tags:
  - domain
---
# TiDB Select count 性能测试

## 机器配置

## 准备数据

mysql> show tables;
+------------------+
| Tables_in_sbtest |
+------------------+
| sbtest1          |
+------------------+
1 row in set (0.00 sec)

## 测试

mysql> select count(1) from sbtest1;
ERROR 9002 (HY000): TiKV server timeout[try again later]
mysql> select count(1) from sbtest1;
+-----------+
| count(1)  |
+-----------+
| 486358785 |
+-----------+
1 row in set (4 min 10.16 sec)

mysql> set @@session.tidb_distsql_scan_concurrency = 30;
Query OK, 0 rows affected (0.06 sec)

mysql> 
mysql> 
mysql> set @@session.tidb_build_stats_concurrency = 30;
Query OK, 0 rows affected (0.00 sec)

mysql> set @@session.tidb_index_serial_scan_concurrency=30;
Query OK, 0 rows affected (0.00 sec)

mysql> select count(1) from sbtest1;
+-----------+
| count(1)  |
+-----------+
| 486358785 |
+-----------+
1 row in set (3 min 25.92 sec)

mysql> set @@session.tidb_index_serial_scan_concurrency = 30;
Query OK, 0 rows affected (0.04 sec)

mysql> select count(1) from sbtest1;
+-----------+
| count(1)  |
+-----------+
| 486358785 |
+-----------+
1 row in set (2 min 11.55 sec)

mysql> set @@session.tidb_index_serial_scan_concurrency = 300;
Query OK, 0 rows affected (0.00 sec)

mysql> select count(1) from sbtest1;
+-----------+
| count(1)  |
+-----------+
| 486358785 |
+-----------+
1 row in set (2 min 19.31 sec)

mysql> set @@session.tidb_distsql_scan_concurrency = 300;
Query OK, 0 rows affected (0.00 sec)

mysql> select count(1) from sbtest1;
+-----------+
| count(1)  |
+-----------+
| 486358785 |
+-----------+
1 row in set (1 min 59.72 sec)

mysql> select count(1) from sbtest1;
+-----------+
| count(1)  |
+-----------+
| 486358785 |
+-----------+
1 row in set (1 min 59.33 sec)


mysql> set @@session.tidb_distsql_scan_concurrency = 3;
Query OK, 0 rows affected (0.10 sec)

mysql> select count(1) from sbtest1;
+-----------+
| count(1)  |
+-----------+
| 486358785 |
+-----------+
1 row in set (4 min 48.57 sec)

mysql> set @@session.tidb_index_serial_scan_concurrency = 3;
Query OK, 0 rows affected (0.04 sec)

mysql> select count(1) from sbtest1;
+-----------+
| count(1)  |
+-----------+
| 486358785 |
+-----------+
1 row in set (4 min 16.00 sec)

mysql> set @@session.tidb_index_serial_scan_concurrency = 300;
Query OK, 0 rows affected (0.05 sec)

mysql> set @@session.tidb_distsql_scan_concurrency = 300;
Query OK, 0 rows affected (0.01 sec)

mysql> select count(1) from sbtest1;
+-----------+
| count(1)  |
+-----------+
| 486358785 |
+-----------+
1 row in set (1 min 38.20 sec)

mysql> select count(1) from sbtest1;
+-----------+
| count(1)  |
+-----------+
| 486358785 |
+-----------+
1 row in set (1 min 32.28 sec)

mysql> select count(1) from sbtest1;
+-----------+
| count(1)  |
+-----------+
| 486358785 |
+-----------+
1 row in set (1 min 31.39 sec)


