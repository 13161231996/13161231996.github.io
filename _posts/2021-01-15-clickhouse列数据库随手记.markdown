---

layout:     post
title:      "clickhouse随手记录GraphiteMergeTree"
subtitle:   " \"clickhouse随手记录GraphiteMergeTree\""
date:       2021-01-15 19:23:00
author:     "BaiDong"
header-img: "img/home-bg-art.jpg"
catalog: true
tags:
    - 列数据库
---
GraphiteMergeTree
该引擎用来对 Graphite数据进行瘦身及汇总。对于想使用CH来存储Graphite数据的开发者来说可能有用。

如果不需要对Graphite数据做汇总，那么可以使用任意的CH表引擎；但若需要，那就采用 GraphiteMergeTree 引擎。它能减少存储空间，同时能提高Graphite数据的查询效率。

该引擎继承自 MergeTree.

    创建表 
    CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
    (
        Path String,
        Time DateTime,
        Value <Numeric_type>,
        Version <Numeric_type>
        ...
    ) ENGINE = GraphiteMergeTree(config_section)
    [PARTITION BY expr]
    [ORDER BY expr]
    [SAMPLE BY expr]
    [SETTINGS name=value, ...]
建表语句的详细说明请参见 创建表

含有Graphite数据集的表应该包含以下的数据列：

    - 指标名称(Graphite sensor)，数据类型：String
    - 指标的时间度量，数据类型： DateTime
    - 指标的值，数据类型：任意数值类型
    - 指标的版本号，数据类型： 任意数值类型

CH以最大的版本号保存行记录，若版本号相同，保留最后写入的数据。
以上列必须设置在汇总参数配置中。

GraphiteMergeTree 参数

    - config_section - 配置文件中标识汇总规则的节点名称

建表语句

在创建 GraphiteMergeTree 表时，需要采用和 clauses 相同的语句，就像创建 MergeTree 一样。

已废弃的建表语句
汇总配置的参数
汇总的配置参数由服务器配置的 graphite_rollup 参数定义。参数名称可以是任意的。允许为多个不同表创建多组配置并使用。

汇总配置的结构如下：
所需的列
模式Patterns

所需的列

    path_column_name — 保存指标名称的列名 (Graphite sensor). 默认值: Path.
    time_column_name — 保存指标时间度量的列名. Default value: Time.
    value_column_name — The name of the column storing the value of the metric at the time set in time_column_name.默认值: Value.
    version_column_name - 保存指标的版本号列. 默认值: Timestamp.
    模式Patterns 
patterns 的结构：

        pattern
            regexp
            function
        pattern
            regexp
            age + precision
            ...
        pattern
            regexp
            function
            age + precision
            ...
        pattern
            ...
        default
            function
            age + precision
            ...
        Attention

模式必须严格按顺序配置：

    不含function or retention的Patterns
    同时含有function and retention的Patterns
    default的Patterns.

CH在处理行记录时，会检查 pattern节点的规则。每个 pattern（含default）节点可以包含 function 用于聚合操作，或retention参数，或者两者都有。如果指标名称和 regexp相匹配，相应 pattern的规则会生效；否则，使用 default 节点的规则。

pattern 和 default 节点的字段设置:

regexp– 指标名的pattern.
age – 数据的最小存活时间(按秒算).
precision– 按秒来衡量数据存活时间时的精确程度. 必须能被86400整除 (一天的秒数).
function – 对于存活时间在 [age, age + precision]之内的数据，需要使用的聚合函数
配置示例

    <graphite_rollup>
        <version_column_name>Version</version_column_name>
        <pattern>
            <regexp>click_cost</regexp>
            <function>any</function>
            <retention>
                <age>0</age>
                <precision>5</precision>
            </retention>
            <retention>
                <age>86400</age>
                <precision>60</precision>
            </retention>
        </pattern>
        <default>
            <function>max</function>
            <retention>
                <age>0</age>
                <precision>60</precision>
            </retention>
            <retention>
                <age>3600</age>
                <precision>300</precision>
            </retention>
            <retention>
                <age>86400</age>
                <precision>3600</precision>
            </retention>
        </default>
    </graphite_rollup>
