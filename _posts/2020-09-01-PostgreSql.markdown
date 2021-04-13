---

layout:     post
title:      "PostgreSql"
subtitle:   " \"PostgreSql频繁对表进行Update,Delete导致锁表\""
date:       2020-09-01 11:23:00
author:     "BaiDong"
header-img: "img/home-bg-art.jpg"
catalog: true
tags:
    - Postgres
---

1.autovacuum的任务

    autovacuum作用

    1、清理UPDATE或DELETE操作后留下的“死元组”

    2、更新可用空间映射(free space map)，以跟踪表块中的可用空间

    3、更新仅索引扫描所需的可见性图(visibility map)

    4、“冻结”(freeze)表行，以便事务ID计数器可以安全地环绕

2.autovacuum触发机制

    1、autovacuum_freeze_max_age达到限定值 200000000 两亿的事务执行的上限的时候,强制冻结事务在增加了,进行强行的VACUUM，这个时候就会锁表，自动清除历史版本

3.autovacuum 解决办法

    1、修改sql,不做大量数据频繁修改
    2、调整 autovacuum_freeze_max_age 限定值