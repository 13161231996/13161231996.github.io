---

layout:     post
title:      "clickhouse随手记录"
subtitle:   " \"clickhouse随手记录PostgreSQL\""
date:       2021-04-01 23:23:00
author:     "BaiDong"
header-img: "img/home-bg-art.jpg"
catalog: true
tags:
    - 列数据库
---
允许连接到远程PostgreSQL服务器上的数据库。支持读写操作（SELECT和INSERT查询）以在 ClickHouse 和 PostgreSQL 之间交换数据。

在查询的帮助下SHOW TABLES，从远程 PostgreSQL 实时访问表列表和表结构DESCRIBE TABLE。

支持表结构修改（ALTER TABLE ... ADD|DROP COLUMN）。如果use_table_cache参数（请参阅下面的引擎参数）设置为1，则表结构将被缓存并且不会检查是否被修改，但可以使用DETACH和ATTACH查询进行更新。

创建数据库

    CREATE DATABASE test_database 
    ENGINE = PostgreSQL('host:port', 'database', 'user', 'password'[, `use_table_cache`]);
参数

    DATE	Date
    TIMESTAMP	DateTime
    REAL	Float32
    DOUBLE	Float64
    DECIMAL, NUMERIC	Decimal
    SMALLINT	Int16
    INTEGER	Int32
    BIGINT	Int64
    SERIAL	UInt32
    BIGSERIAL	UInt64
    TEXT, CHAR	String
    INTEGER	Nullable(Int32)
    ARRAY	Array
使用示例 
ClickHouse 中的数据库，与 PostgreSQL 服务器交换数据：

    CREATE DATABASE test_database 
    ENGINE = PostgreSQL('postgres1:5432', 'test_database', 'postgres', 'mysecretpassword', 1);
    SHOW DATABASES;
    ┌─name──────────┐
    │ default       │
    │ test_database │
    │ system        │
    └───────────────┘
    SHOW TABLES FROM test_database;
    ┌─name───────┐
    │ test_table │
    └────────────┘
从 PostgreSQL 表中读取数据：

    SELECT * FROM test_database.test_table;
    ┌─id─┬─value─┐
    │  1 │     2 │
    └────┴───────┘
将数据写入 PostgreSQL 表：

    INSERT INTO test_database.test_table VALUES (3,4);
    SELECT * FROM test_database.test_table;
    ┌─int_id─┬─value─┐
    │      1 │     2 │
    │      3 │     4 │
    └────────┴───────┘
考虑在 PostgreSQL 中修改了表结构：

postgre> ALTER TABLE test_table ADD COLUMN data Text
由于该use_table_cache参数在1创建数据库时设置为，ClickHouse 中的表结构被缓存，因此不会被修改：

    DESCRIBE TABLE test_database.test_table;
    ┌─name───┬─type──────────────┐
    │ id     │ Nullable(Integer) │
    │ value  │ Nullable(Integer) │
    └────────┴───────────────────┘
分离表并再次附加后，结构已更新：

    DETACH TABLE test_database.test_table;
    ATTACH TABLE test_database.test_table;
    DESCRIBE TABLE test_database.test_table;
    ┌─name───┬─type──────────────┐
    │ id     │ Nullable(Integer) │
    │ value  │ Nullable(Integer) │
    │ data   │ Nullable(String)  │
    └────────┴───────────────────┘