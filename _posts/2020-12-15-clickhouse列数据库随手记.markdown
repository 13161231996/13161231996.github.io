---

layout:     post
title:      "clickhouse随手记录CollapsingMergeTree"
subtitle:   " \"clickhouse随手记录CollapsingMergeTree\""
date:       2020-12-15 19:23:00
author:     "BaiDong"
header-img: "img/home-bg-art.jpg"
catalog: true
tags:
    - 列数据库
---
CollapsingMergeTree表引擎
CollapsingMergeTree就是一种通过以增代删的思路，支持行级数据修改和删除的表引擎。它通过定义一个sign标记位字段，记录数据行的状态。如果sign标记为1，则表示这是一行有效的数据；如果sign标记为-1，则表示这行数据需要被删除。当CollapsingMergeTree分区合并时，同一数据分区内，sign标记为1和-1的一组数据会被抵消删除。

每次需要新增数据时，写入一行sign标记为1的数据；需要删除数据时，则写入一行sign标记为-1的数据。

建表语法

    CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
    (
        name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
        name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
        ...
    ) ENGINE = CollapsingMergeTree(sign)
    [PARTITION BY expr]
    [ORDER BY expr]
    [SAMPLE BY expr]
    [SETTINGS name=value, ...]

建表示例
上面的建表语句使用CollapsingMergeTree(sign)，其中字段sign是一个Int8类型的字段

    CREATE TABLE emp_collapsingmergetree
    (
        emp_id     UInt16 COMMENT '员工id',
        name       String COMMENT '员工姓名',
        work_place String COMMENT '工作地点',
        age        UInt8 COMMENT '员工年龄',
        depart     String COMMENT '部门',
        salary     Decimal32(2) COMMENT '工资',
        sign       Int8
    ) ENGINE = CollapsingMergeTree(sign) ORDER BY (emp_id, name) PARTITION BY work_place;
使用方式

    CollapsingMergeTree同样是以ORDER BY排序键作为判断数据唯一性的依据。

-- 插入新增数据,sign=1表示正常数据

    INSERT INTO emp_collapsingmergetree VALUES (1,'tom','上海',25,'技术部',20000,1);

-- 更新上述的数据

    -- 首先插入一条与原来相同的数据(ORDER BY字段一致),并将sign置为-1
    INSERT INTO emp_collapsingmergetree VALUES (1,'tom','上海',25,'技术部',20000,-1);

    -- 再插入更新之后的数据
    INSERT INTO emp_collapsingmergetree VALUES (1,'tom','上海',25,'技术部',30000,1);

-- 查看一下结果

    select * from emp_collapsingmergetree ;
    ┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┬─sign─┐
    │      1 │ tom  │ 上海       │  25 │ 技术部 │ 20000.00 │   -1 │
    └────────┴──────┴────────────┴─────┴────────┴──────────┴──────┘
    ┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┬─sign─┐
    │      1 │ tom  │ 上海       │  25 │ 技术部 │ 20000.00 │    1 │
    └────────┴──────┴────────────┴─────┴────────┴──────────┴──────┘
    ┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┬─sign─┐
    │      1 │ tom  │ 上海       │  25 │ 技术部 │ 30000.00 │    1 │
    └────────┴──────┴────────────┴─────┴────────┴──────────┴──────┘

-- 执行分区合并操作

    optimize table emp_collapsingmergetree;
    -- 再次查询，sign=1与sign=-1的数据相互抵消了，即被删除
    select * from emp_collapsingmergetree ;

    ┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┬─sign─┐
    │      1 │ tom  │ 上海       │  25 │ 技术部 │ 30000.00 │    1 │
    └────────┴──────┴────────────┴─────┴────────┴──────────┴──────┘
注意点
分区合并
分数数据折叠不是实时的，需要后台进行Compaction操作，用户也可以使用手动合并命令，但是效率会很低，一般不推荐在生产环境中使用。

当进行汇总数据操作时，可以通过改变查询方式，来过滤掉被删除的数据

    SELECT emp_id,name,sum(salary * sign)FROM emp_collapsingmergetree GROUP BY emp_id, name HAVING sum(sign) > 0;
    ┌─emp_id─┬─name─┬─sum(multiply(salary, sign))─┐
    │      1 │ tom  │                    30000.00 │
    └────────┴──────┴─────────────────────────────┘

只有相同分区内的数据才有可能被折叠。其实，当我们修改或删除数据时，这些被修改的数据通常是在一个分区内的，所以不会产生影响。

数据写入顺序
值得注意的是：CollapsingMergeTree对于写入数据的顺序有着严格要求，否则导致无法正常折叠。

-- 建表

    CREATE TABLE emp_collapsingmergetree_order
    (
        emp_id     UInt16 COMMENT '员工id',
        name       String COMMENT '员工姓名',
        work_place String COMMENT '工作地点',
        age        UInt8 COMMENT '员工年龄',
        depart     String COMMENT '部门',
        salary     Decimal32(2) COMMENT '工资',
        sign       Int8
    ) ENGINE = CollapsingMergeTree(sign) ORDER BY (emp_id, name) PARTITION BY work_place;
  
-- 先插入需要被删除的数据，即sign=-1的数据

    INSERT INTO emp_collapsingmergetree_order VALUES (1,'tom','上海',25,'技术部',20000,-1);
    -- 再插入sign=1的数据

    INSERT INTO emp_collapsingmergetree_order VALUES (1,'tom','上海',25,'技术部',20000,1);
-- 查询表

    SELECT * FROM emp_collapsingmergetree_order;
    ┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┬─sign─┐
    │      1 │ tom  │ 上海       │  25 │ 技术部 │ 20000.00 │   -1 │
    └────────┴──────┴────────────┴─────┴────────┴──────────┴──────┘
    ┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┬─sign─┐
    │      1 │ tom  │ 上海       │  25 │ 技术部 │ 20000.00 │    1 │
    └────────┴──────┴────────────┴─────┴────────┴──────────┴──────┘

-- 执行合并操作

    optimize table emp_collapsingmergetree_order;
-- 再次查询表

    -- 旧数据依然存在
    SELECT * FROM emp_collapsingmergetree_order;
    ┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┬─sign─┐
    │      1 │ tom  │ 上海       │  25 │ 技术部 │ 20000.00 │   -1 │
    │      1 │ tom  │ 上海       │  25 │ 技术部 │ 20000.00 │    1 │
    └────────┴──────┴────────────┴─────┴────────┴──────────┴──────┘

如果数据的写入程序是单线程执行的，则能够较好地控制写入顺序；如果需要处理的数据量很大，数据的写入程序通常是多线程执行的，那么此时就不能保障数据的写入顺序了。在这种情况下，CollapsingMergeTree的工作机制就会出现问题。但是可以通过VersionedCollapsingMergeTree的表引擎得到解决。

VersionedCollapsingMergeTree表引擎
上面提到CollapsingMergeTree表引擎对于数据写入乱序的情况下，不能够实现数据折叠的效果。VersionedCollapsingMergeTree表引擎的作用与CollapsingMergeTree完全相同，它们的不同之处在于，VersionedCollapsingMergeTree对数据的写入顺序没有要求，在同一个分区内，任意顺序的数据都能够完成折叠操作。

VersionedCollapsingMergeTree使用version列来实现乱序情况下的数据折叠。

建表语法

    CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
    (
        name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1],
        name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2],
        ...
    ) ENGINE = VersionedCollapsingMergeTree(sign, version)
    [PARTITION BY expr]
    [ORDER BY expr]
    [SAMPLE BY expr]
    [SETTINGS name=value, ...]

可以看出：该引擎除了需要指定一个sign标识之外，还需要指定一个UInt8类型的version版本号。

建表示例

    CREATE TABLE emp_versioned
    (
        emp_id     UInt16 COMMENT '员工id',
        name       String COMMENT '员工姓名',
        work_place String COMMENT '工作地点',
        age        UInt8 COMMENT '员工年龄',
        depart     String COMMENT '部门',
        salary     Decimal32(2) COMMENT '工资',
        sign       Int8,
        version    Int8
    ) ENGINE = VersionedCollapsingMergeTree(sign, version) ORDER BY (emp_id, name) PARTITION BY work_place;
  
-- 先插入需要被删除的数据，即sign=-1的数据

    INSERT INTO emp_versioned VALUES (1,'tom','上海',25,'技术部',20000,-1,1);
    -- 再插入sign=1的数据
    INSERT INTO emp_versioned VALUES (1,'tom','上海',25,'技术部',20000,1,1);
    -- 在插入一个新版本数据
    INSERT INTO emp_versioned VALUES (1,'tom','上海',25,'技术部',30000,1,2);

-- 先不执行合并，查看表数据

    select * from emp_versioned;
    ┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┬─sign─┬─version─┐
    │      1 │ tom  │ 上海       │  25 │ 技术部 │ 20000.00 │    1 │       1 │
    └────────┴──────┴────────────┴─────┴────────┴──────────┴──────┴─────────┘
    ┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┬─sign─┬─version─┐
    │      1 │ tom  │ 上海       │  25 │ 技术部 │ 20000.00 │   -1 │       1 │
    └────────┴──────┴────────────┴─────┴────────┴──────────┴──────┴─────────┘
    ┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┬─sign─┬─version─┐
    │      1 │ tom  │ 上海       │  25 │ 技术部 │ 30000.00 │    1 │       2 │
    └────────┴──────┴────────────┴─────┴────────┴──────────┴──────┴─────────┘

-- 获取正确查询结果

    SELECT emp_id,name,sum(salary * sign) FROM emp_versioned GROUP BY emp_id,name HAVING sum(sign) > 0;
    ┌─emp_id─┬─name─┬─sum(multiply(salary, sign))─┐
    │      1 │ tom  │                    30000.00 │
    └────────┴──────┴─────────────────────────────┘

    -- 手动合并
    optimize table emp_versioned;

    -- 再次查询
    select * from emp_versioned;
    ┌─emp_id─┬─name─┬─work_place─┬─age─┬─depart─┬───salary─┬─sign─┬─version─┐
    │      1 │ tom  │ 上海       │  25 │ 技术部 │ 30000.00 │    1 │       2 │
    └────────┴──────┴────────────┴─────┴────────┴──────────┴──────┴─────────┘

可见上面虽然在插入数据乱序的情况下，依然能够实现折叠的效果。之所以能够达到这种效果，是因为在定义version字段之后，VersionedCollapsingMergeTree会自动将version作为排序条件并增加到ORDER BY的末端，就上述的例子而言，最终的排序字段为ORDER BY emp_id,name，version desc。
