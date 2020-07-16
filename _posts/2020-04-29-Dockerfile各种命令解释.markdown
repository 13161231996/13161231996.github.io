---

layout:     post
title:      "Dockerfile参数命令"
subtitle:   " \"Dockerfile参数命令\""
date:       2020-04-29 18:23:00
author:     "BaiDong"
header-img: "img/home-bg-art.jpg"
catalog: true
tags:
    - Dockerfile参数命令
---
1.使用-f标志with docker build指向文件系统中任何位置的Dockerfile。

    docker build -f /path/to/a/Dockerfile .

2.构建成功，则可以指定一个存储库和标记，用于在其中存储新图像

    docker build -t shykes/myapp .

3.构建后将映像标记到多个存储库中

    docker build -t shykes/myapp:1.0.2 -t shykes/myapp:latest .

4.FROM

    第一条指令必须为FROM 指令。并且，如果在同一个Dockerfile中创建多个镜像时，可以使用多个FROM 指令（每个镜像一次）。

5.MAINTAINER

    格式为 MAINTAINER <name>，指定维护者信息。
6.RUN

    格式为 RUN <command> 或 RUN ["executable", "param1", "param2"]。
    前者将在 shell 终端中运行命令 ，即 /bin/sh -c；后者则使用 exec 执行 。指定使用其它终端可以通过第二种方式实现，例如 RUN ["/bin/bash", "-c", "echo hello"]。
7.CMD

    CMD ["executable","param1","param2"] 使用 exec 执行，推荐方式；
    CMD command param1 param2 在 /bin/sh 中执行，提供给需要交互的应用；
    CMD ["param1","param2"] 提供给 ENTRYPOINT 的默认参数；
8.EXPOSE

    告诉 Docker 服务端容器暴露的端口号，供互联系统使用。在启动容器时需要通过 -P ，Docker 主机会自动分配一个端口转发到指定的端口。
9.ENV

    指定一个环境变量，会被后续 RUN 指令使用，并在容器运行时保持。
    例如：
    ENV PG_MAJOR 9.3
    ENV PG_VERSION 9.3.4
    RUN curl -SL http://example.com/postgres-$PG_VERSION.tar.xz | tar -xJC /usr/src/postgress && …
    ENV PATH /usr/local/postgres-$PG_MAJOR/bin:$PATH
10.ADD

    该命令将复制指定的 到容器中的 。 其中 可以是Dockerfile所在目录的一个相对路径；也可以是一个 URL；还可以是一个 tar 文件（自动解压为目录）
11.ENTRYPOINT

    配置容器启动后执行的命令，并且不可被 docker run 提供的参数覆盖。每个 Dockerfile 中只能有一个 ENTRYPOINT ，当指定多个时，只有最后一个起效。
    ------
    ENTRYPOINT ["executable", "param1", "param2"]
    ENTRYPOINT command param1 param2（shell中执行）。
12.VOLUME

    创建一个可以从本地主机或其他容器挂载的挂载点，一般用来存放数据库和需要保持的数据等。
13.USER

    指定运行容器时的用户名或 UID，后续的 RUN 也会使用指定用户。
    当服务不需要管理员权限时，可以通过该命令指定运行用户。并且可以在之前创建所需要的用户，例如：RUN groupadd -r postgres && useradd -r -g postgres postgres 。要临时获取管理员权限可以使用 gosu，而不推荐 sudo 
14.WOEKDIR

    WORKDIR /path/to/workdir。
    为后续的 RUN、CMD、ENTRYPOINT 指令配置工作目录。
    可以使用多个WORKDIR 指令，后续命令如果参数是相对路径，则会基于之前命令指定的路径。例如
    WORKDIR /a
    WORKDIR b
    WORKDIR c
    RUN pwd
15.ONBUILD

    格式为

    ONBUILD [INSTRUCTION]。
    配置当所创建的镜像作为其它新创建镜像的基础镜像时，所执行的操作指令。
    例如，Dockerfile 使用如下的内容创建了镜像 image-A。
    [...]
    ONBUILD ADD . /app/src
    ONBUILD RUN /usr/local/bin/python-build --dir /app/src
    [...]
    如果基于 image-A 创建新的镜像时，新的Dockerfile中使用 FROM image-A指定基础镜像时，会自动执行 ONBUILD 指令内容，等价于在后面添加了两条指令。
    FROM image-A
    ADD . /app/src
    RUN /usr/local/bin/python-build --dir /app/src  
    使用 ONBUILD 指令的镜像，推荐在标签中注明，例如 ruby:1.9-onbuild 。
16.SHELL

    SHELL用于设置执行命令（shell式）所使用的的默认 shell 类型：
    FROM microsoft/windowsservercore

    # Executed as cmd /S /C echo default
    RUN echo default

    # Executed as cmd /S /C powershell -command Write-Host default
    RUN powershell -command Write-Host default

    # Executed as powershell -command Write-Host hello
    SHELL ["powershell", "-command"]
    RUN Write-Host hello

    # Executed as cmd /S /C echo hello
    SHELL ["cmd", "/S"", "/C"]
    RUN echo hello
17.参数解释

    CMD 在docker run 时运行。
    RUN 是在 docker build。