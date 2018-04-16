---
title: TiDB AUTO_INCREMENT 功能测试
date: 2018-03-27 21:40:52
updated: 2017-04-01 21:40:52
categories:
  - TiDB
tags:
  - AUTO_INCREMENT
---
# TiDB AUTO_INCREMENT 功能测试

- 测试版本：2018-03-27 master
- 测试功能：AUTO_INCREMENT
- 测试原因：使用 AUTO_INCREMENT 时，发现多个 TiDB 查询到的结果中 AUTO_INCREMENT 相差 3 万
- 代码块：[autoid.go](https://github.com/pingcap/tidb/blob/4db4b45e4ab0fba19a3201e169c3d5620996304c/meta/autoid/autoid.go),[autoid_test.go](https://github.com/pingcap/tidb/blob/de9c192cba5e461650dd99f654bf82d659373f15/meta/autoid/autoid_test.go)
- 引用 [RowID](https://github.com/pingcap/blog-cn/blob/master/tidb-internal-2.md) 格式

## 测试单 TiDB

- 假设 TiDB 初始 30001-60000 AUTO_INCREMENT 范围，其中30001 - 45000 部分占用
- 假设出现以下几个场景
  - 指定 AUTO_INCREMENT 数据插入到 30001 之前；正常插入
  - 指定 AUTO_INCREMENT 数据插入到 30001 到 60000 之间；正常插入，会出现 rebase 机制
  - 插入一个已经被占用的 AUTO_INCREMENT

### 测试过程

- 准备基础数据

  ```SQL
  CREATE TABLE t33 (id INT NOT NULL AUTO_INCREMENT, age INT, PRIMARY KEY(id)) ENGINE=InnoDB;

  mysql> select * from t33;
  Empty set (0.01 sec)

  mysql> show create table t33\G;
  *************************** 1. row ***************************
        Table: t33
  Create Table: CREATE TABLE `t33` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `age` int(11) DEFAULT NULL,
    PRIMARY KEY (`id`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin AUTO_INCREMENT=30001
  1 row in set (0.00 sec)
  ```

- 插入数据
  - 不指定 AUTO_INCREMENT 插入数据
  - 指定 AUTO_INCREMENT 插入数据

  ```SQL
  mysql> insert into t33 (age) values(10);
  Query OK, 1 row affected (0.02 sec)

  mysql> select * from t33;
  +----+------+
  | id | age  |
  +----+------+
  |  1 |   10 |
  +----+------+
  1 row in set (0.00 sec)

  mysql> insert into t33 values(10,10);
  Query OK, 1 row affected (0.01 sec)

  mysql> select * from t33;
  +----+------+
  | id | age  |
  +----+------+
  |  1 |   10 |
  | 10 |   10 |
  +----+------+
  2 rows in set (0.00 sec)

  mysql> insert into t33 values(10000,10);
  Query OK, 1 row affected (0.01 sec)

  # 此时 AUTO_INCREMENT 再次不指定 AUTO_INCREMENT 会发生变化

  mysql> select * from t33;
  +-------+------+
  | id    | age  |
  +-------+------+
  |     1 |   10 |
  |    10 |   10 |
  | 10000 |   10 |
  +-------+------+
  3 rows in set (0.00 sec)

  mysql> insert into t33 (age) values(10);
  Query OK, 1 row affected (0.01 sec)

  mysql> select * from t33;
  +-------+------+
  | id    | age  |
  +-------+------+
  |     1 |   10 |
  |    10 |   10 |
  | 10000 |   10 |
  | 10001 |   10 |
  +-------+------+
  4 rows in set (0.00 sec)

  mysql> show create table t33\G;
  *************************** 1. row ***************************
        Table: t33
  Create Table: CREATE TABLE `t33` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `age` int(11) DEFAULT NULL,
    PRIMARY KEY (`id`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin AUTO_INCREMENT=30001
  1 row in set (0.00 sec)
  ```

- 指定 AUTO_INCREMENT 数据插入(大于当前 TiDB AUTO_INCREMENT 范围)
  - 插入数据到 30001 以上

  ```SQL
  mysql> insert into t33 values(30000,10);
  Query OK, 1 row affected (0.01 sec)

  mysql> select * from t33;
  +-------+------+
  | id    | age  |
  +-------+------+
  |     1 |   10 |
  |    10 |   10 |
  | 10000 |   10 |
  | 10001 |   10 |
  | 30000 |   10 |
  +-------+------+
  5 rows in set (0.01 sec)

  mysql> insert into t33 (age) values(10);
  Query OK, 1 row affected (0.02 sec)

  mysql> select * from t33;
  +-------+------+
  | id    | age  |
  +-------+------+
  |     1 |   10 |
  |    10 |   10 |
  | 10000 |   10 |
  | 10001 |   10 |
  | 30000 |   10 |
  | 30001 |   10 |
  +-------+------+
  6 rows in set (0.01 sec)

  # 本条数据插入会超过 30001 的范围

  mysql> insert into t33 (age) values(10);
  Query OK, 1 row affected (0.01 sec)

  mysql> select * from t33;
  +-------+------+
  | id    | age  |
  +-------+------+
  |     1 |   10 |
  |    10 |   10 |
  | 10000 |   10 |
  | 10001 |   10 |
  | 30000 |   10 |
  | 30001 |   10 |
  | 30002 |   10 |
  +-------+------+
  7 rows in set (0.00 sec)

  # 因为超过范围，此时再次查看 show create table 时，AUTO_INCREMENT 已经更新到下个范围，出现 rebase 机制，从当前最大值 + 3W

  mysql> show create table t33\G;
  *************************** 1. row ***************************
        Table: t33
  Create Table: CREATE TABLE `t33` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `age` int(11) DEFAULT NULL,
    PRIMARY KEY (`id`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin AUTO_INCREMENT=60001
  1 row in set (0.00 sec)

  # 继续插入更大的 AUTO_INCREMENT ，TiDB 再次获取下一个 AUTO_INCREMENT 范围

  mysql> insert into t33 values(61000,10);
  Query OK, 1 row affected (0.02 sec)

  mysql> select * from t33;
  +-------+------+
  | id    | age  |
  +-------+------+
  |     1 |   10 |
  |    10 |   10 |
  | 10000 |   10 |
  | 10001 |   10 |
  | 30000 |   10 |
  | 30001 |   10 |
  | 30002 |   10 |
  | 61000 |   10 |
  +-------+------+
  8 rows in set (0.01 sec)

  mysql> show create table t33\G;
  *************************** 1. row ***************************
        Table: t33
  Create Table: CREATE TABLE `t33` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `age` int(11) DEFAULT NULL,
    PRIMARY KEY (`id`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin AUTO_INCREMENT=91001
  1 row in set (0.00 sec)
  ```

- 指定 61000 之前的 AUTO_INCREMENT 数据插入
  - 插入 AUTO_INCREMENT 1000，
  - 重复插入上一条数据，模拟 AUTO_INCREMENT 被占用报错

  ```SQL
  mysql> insert into t33 values(1000,10);
  Query OK, 1 row affected (0.01 sec)

  mysql> select * from t33;
  +-------+------+
  | id    | age  |
  +-------+------+
  |     1 |   10 |
  |    10 |   10 |
  |  1000 |   10 |
  | 10000 |   10 |
  | 10001 |   10 |
  | 30000 |   10 |
  | 30001 |   10 |
  | 30002 |   10 |
  | 61000 |   10 |
  +-------+------+
  9 rows in set (0.00 sec)

  mysql> show create table t33\G;
  *************************** 1. row ***************************
        Table: t33
  Create Table: CREATE TABLE `t33` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `age` int(11) DEFAULT NULL,
    PRIMARY KEY (`id`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8 COLLATE=utf8_bin AUTO_INCREMENT=91001
  1 row in set (0.00 sec)

  # 如果 AUTO_INCREMENT 被占用会出现报错

  mysql> insert into t33 values(1000,10);
  ERROR 1062 (23000): Duplicate entry '1000' for key 'PRIMARY'
  mysql> select * from t33;
  +-------+------+
  | id    | age  |
  +-------+------+
  |     1 |   10 |
  |    10 |   10 |
  |  1000 |   10 |
  | 10000 |   10 |
  | 10001 |   10 |
  | 30000 |   10 |
  | 30001 |   10 |
  | 30002 |   10 |
  | 61000 |   10 |
  +-------+------+
  9 rows in set (0.00 sec)
  ```

## 测试多 TiDB

- ing