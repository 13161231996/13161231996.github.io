---
layout:     post
    title:      "MongoDB集群部署"
subtitle:   " \"快速搜索\""
date:       2018-3-15 12:00:00
author:     "BaiDong"
header-img: "img/home-bg-art.jpg"
catalog: true
tags:
    - MongoDB
---
采用副本集，支持大数据，支持数据库分片，三份

手动创建文件夹  mkdir /data      mkdir  /data/db

mongo副本集与分片集群配置

 mongo副本集与分片集群配置

 测试环境使用'

mongo'

replicaSe= new ReplSetTest("nodes": 3)

replicaSe. startSet()

replicaSe initiate()

现在已经有了3个 mongo进程,分别运行在31000、31001和31002端口。开

启一个新的shel,在第二个 shell中,连接到运行在31000端口的 mongo:

conn =new Mongo("localhost: 31000")'

connection to localhost: 31000

testReplSet: PRIMARY>

testReplSet: PRIMARY> primaryDB= conn.getDB(test''片集群

mongo--nodb

cluster =new Sharding Test("shards":3, "chunksize: 1)'

运行这个命令就会创建一个包含3个分片( mongo进程)的集群,分别运行在

30000、30001、30002端口。默认情况下, ShardingTest会在30999端

口启动 mongos。接下来就连接到这个 manos开始使用集群

---


