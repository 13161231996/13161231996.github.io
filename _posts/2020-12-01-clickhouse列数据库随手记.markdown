---

layout:     post
title:      "clickhouse随手记录AggregatingMergeTree"
subtitle:   " \"clickhouse随手记录AggregatingMergeTree\""
date:       2020-12-01 19:23:00
author:     "BaiDong"
header-img: "img/home-bg-art.jpg"
catalog: true
tags:
    - 列数据库
---
AggregatingMergeTree
AggregatingMergeTree可以理解为SummingMergeTree的升级版，他们的许多设计思路上都是一致的，例如同时定义ORDER BY和PRIMARY KEY的原因和目的，但是在使用方法上存在差异。
    AggregatingMergeTree引擎能够在合并分区时，按照预先定义的条件聚合数据，同时，根据预先定义的聚合函数计算数据并通过二进制的格式存入表内。

AggregatingMergeTree表引擎声明方式如下
ENGINE = AggregationMergeTree()
AggregatingMergeTree没有任何额外的设置参数，在分区合并时，在每个数据分区内，会按照ORDER BY聚合，而使用何种聚合函数，以及针对哪些列字段计算，则是通过定义AggregateFunction函数类型实现，例如：

    create table test_agg (
    shop_code String,
    product_code String,
    name AggregateFunction(uniq,String),
    out_count AggregateFunction(sum,Int),
    write_date DateTime
    ) ENGINE = AggregatingMergeTree()
    PARTITION BY toYYYYMM(write_date)
    ORDER BY (shop_code,product_code)
    PRIMARY KEY shop_code;

AggregateFunction时ClickHouse提供的一种特殊的数据类型，它能够以二进制的形式存储中间状态结果，AggregateFunction类型的数据再写入和查询时需要分别调用*state、*merge函数，*表示定义字段类型时使用的聚合函数，上边表定义的name、out_count字段分别使用了uniq、sum函数，那么在写入数据时需要调用uniqState、sumState函数，并使用INSERT SELECT语法：

    insert into test_agg select '1','短袖',uniqState('tracy'),sumState(toInt32(100)),'2020-12-17 21:18:00';
    insert into test_agg select '1','短袖',uniqState('tracy'),sumState(toInt32(100)),'2020-12-17 21:18:00';
    insert into test_agg select '1','短袖',uniqState('tracy'),sumState(toInt32(100)),'2020-11-17 21:18:00';
    insert into test_agg select '1','短袖',uniqState('tracy'),sumState(toInt32(100)),'2020-12-17 21:18:00';
    insert into test_agg select '1','短袖',uniqState('tracy'),sumState(toInt32(100)),'2020-11-17 21:18:00';
    insert into test_agg select '1','短袖',uniqState('monica'),sumState(toInt32(100)),'2020-12-17 21:18:00';

在查询数据时也需要调用对应的函数uniqMerge、sumMerge：

    select shop_code,product_code,uniqMerge(name),sumMerge(out_count) from test_agg group by shop_code,product_code;
    1

到这里其实会发现AggregatingMergeTree作为基础表引擎用起来很繁琐，所以，AggregatingMergeTree更常用的方式是结合物化试图使用，物化视图即其它数据表上层的一种查询视图。

AggregatingMergeTree表引擎与物化视图结合使用
物化视图即其它数据表上层的一种查询视图，接下来，我们测试AggregatingMergeTree与物化视图结合使用的场景，首先，建立明细数据表，也就是底表：

    create table test_table (
    shop_code String,
    product_code String,
    name String,
    out_count Int,
    write_date DateTime
    ) ENGINE = AggregatingMergeTree()
    PARTITION BY toYYYYMM(write_date)
    ORDER BY (shop_code,product_code);

通常这里会使用MergeTree作为基础表，用于存储全量的明细数据，并以此对外提供实时查询，接着，新建一张物化视图：

    create materialized view test_agg_view 
    ENGINE = AggregatingMergeTree()
    PARTITION BY shop_code
    ORDER BY (shop_code,product_code)
    AS SELECT
    shop_code,
    product_code,
    uniqState(name) as name,
    sumState(out_count) as out_count
    FROM test_table
    GROUP BY shop_code,product_code;

新增或插入数据时，我们还是直接把明细数据写进基础表：

    insert into test_table values 
    ('1','短袖','tracy',100,'2020-12-17 21:18:00'),
    ('1','短袖','tracy',100,'2020-12-18 21:18:00'),
    ('1','外套','tracy',100,'2020-12-19 21:18:00');
在查询数据时，直接查询物化视图test_agg_view ：

    select shop_code,uniqMerge(name),sumMerge(out_count) from test_agg_view group by shop_code,product_code;

总结

    使用ORDER BY排序键作为聚合数据的依据
    使用AggregateFunction字段类型定义聚合函数的类型以及聚合字段
    只有在合并分区的时候才会触发聚合计算的逻辑
    聚合只会发生在同分区内，不同分区的数据不会发生聚合
    在进行数据计算时，因为同分区的数据已经基于ORDER BY排序，所以能够找到相邻且具有相同聚合key的数据
    在聚合数据时，同一分区内，相同聚合key的多行数据会合并成一行，对于那些非主键、非AggregateFunction类型字段，则会取第一行数据
    AggregateFunction类型字段使用二进制存储，在写入数据时，需要调用state函数；在读数据时，需要调用merge函数，*表示定义时使用的聚合函数
    AggregateMergeTree通常作为物化视图的引擎，与普通的MergeTree搭配使用
