---

layout:     post
title:      "clickhouse随手记录VersionedCollapsingMergeTree"
subtitle:   " \"clickhouse随手记录VersionedCollapsingMergeTree\""
date:       2021-01-01 19:23:00
author:     "BaiDong"
header-img: "img/home-bg-art.jpg"
catalog: true
tags:
    - 列数据库
---
CVersionedCollapsingMergeTree表引擎声明方式如下

    ENGINE = VersionedCollapsingMergeTree(sign,ver)

ver为版本号字段，VersionedCollapsingMergeTree会将ver字段做为排序条件增加到order by的末端
例如：

    create table test_collapsing (
    shop_code String,
    product_code String,
    name String,
    sign Int8,
    write_date DateTime
    ) ENGINE = VersionedCollapsingMergeTree(sign,write_date)
    PARTITION BY toYYYYMM(write_date)
    ORDER BY (shop_code,product_code);

上边的表会根据shop_code，product_code，write_date对同分区的数据进行排序，最终数据会根据排序键shop_code,product_code进行数据折叠，因此VersionedCollapsingMergeTree也是通过这种方式解决了CollapsingMergeTree数据乱序写入时的折叠问题
插入数据：

    insert into table test_collapsing values ('1','ccc','TRACY',1,'2021-01-11 23:00:00');
    insert into table test_collapsing values ('1','ccc','TRACY',-1,'2021-01-11 23:00:00');

进行合并分区:

    optimize table test_collapsing final;

插入数据：

    insert into table test_collapsing values ('1','ccc','TRACY',1,'2021-01-11 23:00:00');
    insert into table test_collapsing values ('1','ccc','TRACY',-1,'2021-01-11 23:00:00');
    insert into table test_collapsing values ('1','ccc','Monica',-1,'2021-01-11 23:22:00');

场景举例

    在正常的数据库的CDC模式下，update、delete的操作都会带上数据更新或删除前的记录信息，如果想在ClickHouse里完成对应的update、delete的操作，只需要把更新或删除前的记录行带上sign为-1写进数据库即可，最新状态的数据带上sign为1写入即可

折叠数据的规则

    如果sign=1比sign=-1的数据多一行，则保留版本号最新的的一行sign为1的数据
    如果sign=-1比sign=1的数据多一行，则保留版本号最老的一行sign为-1的数据
    如果sign=1和sign=1的数据行一样多，并且最后一行是sign=1，则保留版本号最老的sign=-1的数据和版本号最新的sign=1的数据
    如果sign=1和sign=1的数据行一样多，并且最后一行是sign=-1，则什么也不保留
    使用时的注意点
VersionedCollapsingMergeTree的折叠并不是实时触发的，解决这个问题有以下两种方案：
1、在查询数据前进行分区合并

    optimize table test_collapsing final;
    
    2、改变查询方式
    原sql:

    select shop_code,product_code,sum(code) from test_collapsing group by shop_code,product_code;

    修改后：

    select shop_code,product_code,sum(code * sign) from test_collapsing group by shop_code,product_code having sum(sign) > 0;

VersionedCollapsingMergeTree数据折叠也是发生在分区合并时，只会对同分区的数据进折叠
