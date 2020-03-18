---

layout:     post
title:      "Flask_Nginx_Uwsgi"
subtitle:   " \"Flask_Nginx_Uwsgi服务配置\""
date:       2019-10-15 17:23:00
author:     "BaiDong"
header-img: "img/home-bg-art.jpg"
catalog: true
tags:
    - WEB服务配置
---
1、安装相关的依赖包  Centos

    yum install gcc -c++

    yum install -y pcre pcre-devel

    yum install -y zlib zlib-devel

    yum install -y openssl openssl-devel

2、安装nginx

    具体版本请自行修改

    wget -c https://nginx.org/download/nginx-1.10.1.tar.gz

解压、进入安装包

    tar -zxvf nginx -1.10.1.tar
    cd nginx-1.10.1

编译安装

    make

    make install

查找安装路径


    whereis nginx 

    进入相应目录

    cd /usr/local/nginx/sbin/

    ./nginx
    ./nginx -s stop
    ./nginx -s quit
    ./nginx -s reload

文件配置

        根据nginx版本不同默认路径不一样，
        我的机器默认在 /usr/local/nginx
        或者在  /etc/nginx下
        cd /usr/local/nginx/conf   修改nginx.conf 
        自上保留到    #gzip  on; 注意把最后一个 } 留下
        并在  #gzip  on; 下 添加      include /usr/local/nginx/conf/conf.d/*.conf;

        在  /usr/local/nginx/conf 下创建  conf.d 文件夹并编写文件 flask.conf

flask.conf 内容

        server{
            listen 8080;   
            server_name 0.0.0.0; #访问ip

            location / {
            include uwsgi_params;
            uwsgi_pass 127.0.0.1:5645;  #代理到uwsgi.ini里部署的ip+端口
            #uwsgi_pass unix:uwsgi.sock;#通过.sock文件通信的写法
            }
         }：wq


安装 uwsgi 


    pip install uwsgi

创建软连接

    ln -s /usr/local/python3/bin/uwsgi /usr/bin/uwsgi

创建flask_uwsgi.ini

    [uwsgi]
    #监听的ip和端口
    socket = 127.0.0.1:5645

    #项目目录
    chdir = /opt/

    #flask程序的启动文件，通常在本地是通过运行
    wsgi-file = app.py

    #程序内启用的application变量名
    callable = app

    #处理器个数
    processes = 2
    pidfile=uwsgi.pid

    #获取uwsgi统计信息的服务地址
    stats = 127.0.0.1:9191


创建 app.py

    # -*- coding: utf-8 -*-
    from flask import Flask, request, jsonify
    #from flask_cors import *
    app = Flask(__name__)
    #CORS(app,supports_credentials=True)
    @app.route('/', methods=['GET'])
    def index():
        return '312312312'

    if __name__ == '__main__':
        app.debug = True
        app.run()

创建 uwsgi.pid

    touch uwsgi.pid

创建log文件

    mkdir   /var/log/uwsgi
    touch   uwsgi.log



启动uwsgi
    
    后台运行    uwsgi --ini flask_uwsgi.ini -d /var/log/uwsgi/uwsgi.log

    非后台运行   uwsgi --ini flask_uwsgi.ini

停止uwsgi

    pkill -f uwsgi -9



