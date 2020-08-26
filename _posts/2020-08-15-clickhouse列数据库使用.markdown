---

layout:     post
title:      "clickhouse使用方式"
subtitle:   " \"clickhouse使用方式\""
date:       2020-08-15 18:23:00
author:     "BaiDong"
header-img: "img/home-bg-art.jpg"
catalog: true
tags:
    - 列数据库
---

1.异步查询

    python3.7及以上
    pip 安装aiochclient,aiohttp

    async with ClientSession() as s:
    client = ChClient(s, url='http://10.255.175.94:8123'),#为数据库连接
                        user=‘default’ ),#默认为‘default’
                        password='123456'),
                        database='default'))
    return await client.fetch(sql, json=True) #json默认为false ,True后返回值为字典，否则为对象

2.创建任务

    async def main(sql):
    task = list()
    task.append(asyncio.create_task(CKgetconn(sql)))
    return await asyncio.gather(*task)

    
3.传入sql，启动协程

    def func():
        sql="select * from student"
        result = asyncio.run(main(sql))
        return result