---

layout:     post
title:      "clickhouse随手记录"
subtitle:   " \"clickhouse随手记录故障排除\""
date:       2021-02-15 21:23:00
author:     "BaiDong"
header-img: "img/home-bg-art.jpg"
catalog: true
tags:
    - 列数据库
---
安装

    您无法使用 Apt-get 从 ClickHouse 存储库获取 Deb 包 
    检查防火墙设置。
    如果您因任何原因无法访问存储库，请按照入门文章中的说明下载软件包并使用sudo dpkg -i <packages>命令手动安装它们。您还需要该tzdata软件包。
连接到服务器可能的问题：

    服务器没有运行。
    意外或错误的配置参数。
    服务器未运行 
    检查服务器是否正在运行

命令：

    $ sudo service clickhouse-server status
    如果服务器未运行，请使用以下命令启动它：

    $ sudo service clickhouse-server start
    检查日志

    默认情况下，主日志clickhouse-server在/var/log/clickhouse-server/clickhouse-server.log。

如果服务器成功启动，您应该看到字符串：

    <Information> Application: starting up. — 服务器启动。
    <Information> Application: Ready for connections. — 服务器正在运行并准备好连接。
    如果clickhouse-server启动失败并出现配置错误，您应该会看到<Error>带有错误描述的字符串。例如：

    2019.01.11 15:23:25.549505 [ 45 ] {} <Error> ExternalDictionaries: Failed reloading 'event2id' external dictionary: Poco::Exception. Code: 1000, e.code() = 111, e.displayText() = Connection refused, e.what() = Connection refused
    如果在文件末尾没有看到错误，请从字符串开始查看整个文件：

    <Information> Application: starting up.
如果您尝试clickhouse-server在服务器上启动第二个实例，您会看到以下日志：

    2019.01.11 15:25:11.151730 [ 1 ] {} <Information> : Starting ClickHouse 19.1.0 with revision 54413
    2019.01.11 15:25:11.154578 [ 1 ] {} <Information> Application: starting up
    2019.01.11 15:25:11.156361 [ 1 ] {} <Information> StatusFile: Status file ./status already exists - unclean restart. Contents:
    PID: 8510
    Started at: 2019-01-11 15:24:23
    Revision: 54413

    2019.01.11 15:25:11.156673 [ 1 ] {} <Error> Application: DB::Exception: Cannot lock file ./status. Another server instance in same directory is already running.
    2019.01.11 15:25:11.156682 [ 1 ] {} <Information> Application: shutting down
    2019.01.11 15:25:11.156686 [ 1 ] {} <Debug> Application: Uninitializing subsystem: Logging Subsystem
    2019.01.11 15:25:11.156716 [ 2 ] {} <Information> BaseDaemon: Stop SignalListener thread
查看 system.d 日志

如果在clickhouse-server日志中没有找到有用的信息或者没有日志，可以system.d使用以下命令查看日志：

    $ sudo journalctl -u clickhouse-server
    以交互模式启动 clickhouse-server

    $ sudo -u clickhouse /usr/bin/clickhouse-server --config-file /etc/clickhouse-server/config.xml
    此命令使用自动启动脚本的标准参数将服务器作为交互式应用程序启动。在这种模式下clickhouse-server打印控制台中的所有事件消息。

配置参数
查看：

    码头工人设置。

    如果您在 IPv6 网络中的 Docker 中运行 ClickHouse，请确保network=host已设置。

    端点设置。

    检查listen_host和tcp_port设置。

    默认情况下，ClickHouse 服务器仅接受本地主机连接。

    HTTP 协议设置。

    检查 HTTP API 的协议设置。

    安全连接设置。

查看：

    该tcp_port_secure设置。
    SSL 证书的设置。
    连接时使用适当的参数。例如，将port_secure参数与clickhouse_client.

    用户设置。

    您可能使用了错误的用户名或密码。

    查询处理 
    如果 ClickHouse 无法处理查询，它会向客户端发送错误描述。在 中，clickhouse-client您会在控制台中获得错误的描述。如果您使用的是 HTTP 接口，ClickHouse 会在响应正文中发送错误描述。例如：

    $ curl 'http://localhost:8123/' --data-binary "SELECT a"
    Code: 47, e.displayText() = DB::Exception: Unknown identifier: a. Note that there are no tables (FROM clause) in your query, context: required_names: 'a' source_tables: table_aliases: private_aliases: column_aliases: public_columns: 'a' masked_columns: array_join_columns: source_columns: , e.what() = DB::Exception
如果你开始clickhouse-client与stack-trace参数，ClickHouse返回一个错误的说明服务器堆栈跟踪。

您可能会看到有关断开连接的消息。在这种情况下，您可以重复查询。如果每次执行查询时连接都中断，请检查服务器日志是否有错误。

查询处理的效率

    如果您发现 ClickHouse 运行速度太慢，则需要为您的查询分析服务器资源和网络上的负载。

    您可以使用 clickhouse-benchmark 实用程序来分析查询。它显示每秒处理的查询数、每秒处理的行数以及查询处理时间的百分位数。
