---

layout:     post
title:      "clickhouse随手记录SummingMergeTree"
subtitle:   " \"clickhouse随手记录SummingMergeTree\""
date:       2020-11-15 19:23:00
author:     "BaiDong"
header-img: "img/home-bg-art.jpg"
catalog: true
tags:
    - 列数据库
---
SummingMergeTree

    该引擎继承了MergeTree引擎，当合并 SummingMergeTree 表的数据片段时，ClickHouse 会把所有具有相同主键的行合并为一行，该行包含了被合并的行中具有数值数据类型的列的汇总值，即如果存在重复的数据，会对对这些重复的数据进行合并成一条数据，类似于group by的效果。

    推荐将该引擎和 MergeTree 一起使用。例如，将完整的数据存储在 MergeTree 表中，并且使用 SummingMergeTree 来存储聚合数据。这种方法可以避免因为使用不正确的主键组合方式而丢失数据。

    如果用户只需要查询数据的汇总结果，不关心明细数据，并且数据的汇总条件是预先明确的，即GROUP BY的分组字段是确定的，可以使用该表引擎。

    建表语法
    CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
    (
        name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
        name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
        ...
    ) ENGINE = SummingMergeTree([columns]) -- 指定合并汇总字段
    [PARTITION BY expr]
    [ORDER BY expr]
    [SAMPLE BY expr]
    [SETTINGS name=value, ...]

建表示例

    CREATE TABLE emp_summingmergetree
    (
        emp_id     UInt16 COMMENT '员工id',
        name       String COMMENT '员工姓名',
        work_place String COMMENT '工作地点',
        age        UInt8 COMMENT '员工年龄',
        depart     String COMMENT '部门',
        salary     Decimal32(2) COMMENT '工资'
    ) ENGINE = SummingMergeTree(salary) ORDER BY (emp_id, name) PRIMARY KEY emp_id PARTITION BY work_place;
  
    ORDER BY (emp_id,name) -- 注意排序key是两个字段
    PRIMARY KEY emp_id     -- 主键是一个字段
插入数据 

    INSERT INTO emp_summingmergetree VALUES (1,'tom','上海',25,'技术部',20000),(2,'jack','上海',26,'人事部',10000);
    INSERT INTO emp_summingmergetree VALUES (3,'bob','北京',33,'财务部',50000),(4,'tony','杭州',28,'销售事部',50000); 

    当我们再次插入具有相同emp_id,name的数据时，观察结果

    INSERT INTO emp_summingmergetree VALUES (1,'tom','上海',25,'信息部',10000),(1,'tom','北京',26,'人事部',10000);

    select * from emp_summingmergetree;

    ┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┐
    │      1 │ tom  │ 上海       │  25 │ 信息部 │ 10000.00 │
    └────────┴──────┴────────────┴─────┴────────┴──────────┘
    ┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart───┬───salary─┐
    │      4 │ tony │ 杭州       │  28 │ 销售事部 │ 50000.00 │
    └────────┴──────┴────────────┴─────┴──────────┴──────────┘
    ┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┐
    │      3 │ bob  │ 北京       │  33 │ 财务部 │ 50000.00 │
    └────────┴──────┴────────────┴─────┴────────┴──────────┘
    ┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┐
    │      1 │ tom  │ 上海       │  25 │ 技术部 │ 20000.00 │
    │      2 │ jack │ 上海       │  26 │ 人事部 │ 10000.00 │
    └────────┴──────┴────────────┴─────┴────────┴──────────┘
    ┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┐
    │      1 │ tom  │ 北京       │  26 │ 人事部 │ 10000.00 │
    └────────┴──────┴────────────┴─────┴────────┴──────────┘

-- 执行合并操作

    optimize table emp_summingmergetree final;
    select * from emp_summingmergetree; 

    -- 再次查询，新插入的数据 1 │ tom  │ 上海       │  25 │ 信息部 │ 10000.00 
    -- 原来的数据 ：        1 │ tom  │ 上海       │  25 │ 技术部 │ 20000.00
    -- 这两行数据合并成：    1 │ tom  │ 上海       │  25 │ 技术部 │ 30000.00
    ┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┐
    │      1 │ tom  │ 上海       │  25 │ 技术部 │ 30000.00 │
    │      2 │ jack │ 上海       │  26 │ 人事部 │ 10000.00 │
    └────────┴──────┴────────────┴─────┴────────┴──────────┘
    ┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart───┬───salary─┐
    │      4 │ tony │ 杭州       │  28 │ 销售事部 │ 50000.00 │
    └────────┴──────┴────────────┴─────┴──────────┴──────────┘
    ┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┐
    │      1 │ tom  │ 北京       │  26 │ 人事部 │ 10000.00 │
    │      3 │ bob  │ 北京       │  33 │ 财务部 │ 50000.00 │
    └────────┴──────┴────────────┴─────┴────────┴──────────┘