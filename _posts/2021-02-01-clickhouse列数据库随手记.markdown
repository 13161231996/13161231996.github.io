---

layout:     post
title:      "clickhouse随手记录"
subtitle:   " \"clickhouse随手记录Catboost 模型\""
date:       2021-02-01 19:23:00
author:     "BaiDong"
header-img: "img/home-bg-art.jpg"
catalog: true
tags:
    - 列数据库
---
在 ClickHouse 中应用 Catboost 模型

    CatBoost是Yandex为机器学习开发的免费开源梯度提升库。

    通过此说明，您将学习通过从 SQL 运行模型推理，在 ClickHouse 中应用预训练模型。

在 ClickHouse 中应用 CatBoost 模型：

    创建一个表。
    将数据插入表中。
    将 CatBoost 集成到 ClickHouse（可选步骤）。
    从 SQL 运行模型推理。
    有关训练 CatBoost 模型的更多信息，请参阅训练和应用模型。

    如果配置已更新而无需使用RELOAD MODEL和RELOAD MODELS系统查询重新启动服务器，则您可以重新加载 CatBoost 模型。

先决条件
如果您还没有Docker，请安装它。

    笔记

    Docker是一个软件平台，允许您创建容器，将 CatBoost 和 ClickHouse 安装与系统的其余部分隔离开来。

在应用 CatBoost 模型之前：

    1.从注册表中拉取Docker 镜像：

    $ docker pull yandex/tutorial-catboost-clickhouse
    此 Docker 映像包含运行 CatBoost 和 ClickHouse 所需的一切：代码、运行时、库、环境变量和配置文件。

    2.确保已成功拉取 Docker 镜像：

    $ docker image ls
    REPOSITORY                            TAG                 IMAGE ID            CREATED             SIZE
    yandex/tutorial-catboost-clickhouse   latest              622e4d17945b        22 hours ago        1.37GB
    3、基于这个镜像启动一个Docker容器：

    $ docker run -it -p 8888:8888 yandex/tutorial-catboost-clickhouse
创建表
要为训练样本创建 ClickHouse 表：

    1.以交互方式启动ClickHouse控制台客户端：

    $ clickhouse client
    笔记

    ClickHouse 服务器已经在 Docker 容器内运行。

    2.使用以下命令创建表：

    :) CREATE TABLE amazon_train
    (
        date Date MATERIALIZED today(),
        ACTION UInt8,
        RESOURCE UInt32,
        MGR_ID UInt32,
        ROLE_ROLLUP_1 UInt32,
        ROLE_ROLLUP_2 UInt32,
        ROLE_DEPTNAME UInt32,
        ROLE_TITLE UInt32,
        ROLE_FAMILY_DESC UInt32,
        ROLE_FAMILY UInt32,
        ROLE_CODE UInt32
    )
    ENGINE = MergeTree ORDER BY date
    3.从 ClickHouse 控制台客户端退出：

    :) exit
插入数据到表中
插入数据：

    1.运行以下命令：

    $ clickhouse client --host 127.0.0.1 --query 'INSERT INTO amazon_train FORMAT CSVWithNames' < ~/amazon/train.csv
    2.以交互方式启动ClickHouse控制台客户端：

    $ clickhouse client
    3.确保数据已上传：

    :) SELECT count() FROM amazon_train

    SELECT count()
    FROM amazon_train

    +-count()-+
    |   65538 |
    +-------+
将 CatBoost 集成到 ClickHouse

    可选步骤。Docker 映像包含运行 CatBoost 和 ClickHouse 所需的一切。

    将 CatBoost 集成到 ClickHouse：

    1.构建评估库。

    评估 CatBoost 模型的最快方法是编译libcatboostmodel.<so|dll|dylib>库。有关如何构建库的更多信息，请参阅CatBoost 文档。

    2.例如，在任何地方以任何名称创建一个新目录，data并将创建的库放入其中。Docker 镜像已经包含库data/libcatboostmodel.so。

    3.在任何地方用任何名称为配置模型创建一个新目录，例如，models.

    4.创建一个任意名称的模型配置文件，例如models/amazon_model.xml.

5.描述模型配置：

    <models>
        <model>
            <!-- Model type. Now catboost only. -->
            <type>catboost</type>
            <!-- Model name. -->
            <name>amazon</name>
            <!-- Path to trained model. -->
            <path>/home/catboost/tutorial/catboost_model.bin</path>
            <!-- Update interval. -->
            <lifetime>0</lifetime>
        </model>
    </models>
6.将CatBoost的路径和模型配置添加到ClickHouse配置中：

    <!-- File etc/clickhouse-server/config.d/models_config.xml. -->
    <catboost_dynamic_library_path>/home/catboost/data/libcatboostmodel.so</catboost_dynamic_library_path>
    <models_config>/home/catboost/models/*_model.xml</models_config>
    笔记

    您可以稍后更改 CatBoost 模型配置的路径，而无需重新启动服务器。

    从 SQL 运行模型推理 
    对于测试模型，运行 ClickHouse 客户端$ clickhouse client。

让我们确保模型正常工作：

    :) SELECT
        modelEvaluate('amazon',
                    RESOURCE,
                    MGR_ID,
                    ROLE_ROLLUP_1,
                    ROLE_ROLLUP_2,
                    ROLE_DEPTNAME,
                    ROLE_TITLE,
                    ROLE_FAMILY_DESC,
                    ROLE_FAMILY,
                    ROLE_CODE) > 0 AS prediction,
        ACTION AS target
    FROM amazon_train
    LIMIT 10

    函数modelEvaluate返回带有多类模型的每类原始预测的元组。

让我们预测一下概率：

    :) SELECT
        modelEvaluate('amazon',
                    RESOURCE,
                    MGR_ID,
                    ROLE_ROLLUP_1,
                    ROLE_ROLLUP_2,
                    ROLE_DEPTNAME,
                    ROLE_TITLE,
                    ROLE_FAMILY_DESC,
                    ROLE_FAMILY,
                    ROLE_CODE) AS prediction,
        1. / (1 + exp(-prediction)) AS probability,
        ACTION AS target
    FROM amazon_train
    LIMIT 10
笔记

有关exp()函数的更多信息。

让我们计算样本的 LogLoss：

    :) SELECT -avg(tg * log(prob) + (1 - tg) * log(1 - prob)) AS logloss
    FROM
    (
        SELECT
            modelEvaluate('amazon',
                        RESOURCE,
                        MGR_ID,
                        ROLE_ROLLUP_1,
                        ROLE_ROLLUP_2,
                        ROLE_DEPTNAME,
                        ROLE_TITLE,
                        ROLE_FAMILY_DESC,
                        ROLE_FAMILY,
                        ROLE_CODE) AS prediction,
            1. / (1. + exp(-prediction)) AS prob,
            ACTION AS tg
        FROM amazon_train
    )
