---
title: Docker基础
date: 2018-04-24 21:58:19
tags:
    - Docker
categories:
    - Docker
---

## 一、什么是 Docker

[Docker](https://www.docker.com/) 是基于 Go 语言实现的开源容器项目。早期 Docker 直接基于 LXC 实现，0.9 版后 Docker 开发了 libcontainer 项目，作为更广泛的容器驱动实现。Docker 通过引入分层文件系统构建和高效的镜像机制，降低了迁移难度。可以这样简单的理解，相比于 LXC，Docker 是一种更高层次的多功能的容器管理工具。

![LXC 和 Docker 的区别](difference.png)
<!-- more -->
### Docker 优势

1. 更快速的交付和部署
2. 更高效的资源利用
3. 更轻松的迁移和扩展
4. 更简单的更新管理

### Docker 虚拟化

传统的虚拟化都需要额外的虚拟机管理应用，和虚拟机操作系统层。Docker 属于操作系统级虚拟化，不需要额外的 supervisor 支持，直接服用本地主机的操作系统，因此开销更小，速度更快。

## 二、三大核心概念和安装配置

### 三大核心概念

1. 镜像（image）
2. 容器（container）
3. 仓库（repository）

### [安装](https://www.docker.com/community-edition) 

## 三、基础命令

### 操作镜像

* 下载：`docker pull ubuntu:14.04`
* 启动：`docker run -it ubuntu:14.04`
* 列出所有镜像：`docker image`
* 添加标签：`docker tag ubuntu:latest myubuntu:latest`
* 查看信息：`docker inspect`
    * 只看一项（eg: Architecture）：`-f {{".Architecture"}}`
* 查看镜像历史（各层创建信息）：`docker history ubuntu:14.04`
* 搜索：`docker search 关键字`
    * `--automated=true|false`: 仅显示自动创建的镜像
    * `--no-truc=true|false`: 输出信息不截取显示
    * `-s X, --stars=X`: 指定显示评价为指定星级以上的镜像
* 删除：`docker rmi tag/id`
    * 多于一个标签的镜像，只删除一个标签，不影响其他的镜像
    * 删除 id 时，会删除所有标签并且删除镜像
    * `-f`: 强制删除正在运行的容器镜像 （不推荐），建议先删除 (`docker rm`) 容器，再删除镜像
* 创建：`docker commit -m "Add new file" -a "chrhong" a929b8d76f4 test:1.0`
* 导入：`cat ubuntu-14.04-x86_64-minimal.tar.gz | docker import - ubuntu:14.04`
* 存入：`docker save -o ubuntu_14.04.tar ubuntu:14.04`
* 载入：`docker load --input ubuntu_14.04.tar`
    * `docker load < ubuntu_14.04.tar`
* 上传：`docker push chrhong/test:latest`
    * 已经加过标签：`docker tag test:latest chrhong/test:latest`

### 操作容器

* 创建： `docker create -it ubuntu:latest`
* 启动一个已创建的容器： `docker start containerid`
    * `contianerid` 可以只取可以唯一标识的前几位
* 直接启动一个容器：`docker run ubuntu /bin/echo 'Hello ubuntu'`
    * 执行完`echo`容器自动退出
* 守护态运行：`docker run -d ubuntu /bin/sh -c "while true; do echo 'hello world'; sleep 1; done"`
* 查看容器日志：`docker logs containerid`
* 终止容器：`docker stop contaienrid`
    * `docker kill` 会直接发送 SIGKILL 信号强制终止容器
* 查看容器信息： `docker ps`
    * `-qa` 查看所有容器 contaienrid
* 重启容器： `docker restart containerid`
* 进入容器：
    * `docker attach containername`
        * 所有 attach 到同一个容器的窗口将同步显示，有阻塞问题
    * `docker exec -it contianerid /bin/bash`
        * 进入容器的推荐方式
    * `nsenter --target $PID --mount --uts --ipc --net --pid`
        * 利用 `docker inspect` 检查信息中` .State.Pid` 字段获得 PID
        * `PID=$(docker-pid contianerid)`
        * util-linux 软件包包含 nsenter 工具
* 删除容器：`docker rm containerid`
    * `-f`: 强制删除
    * `-l`: 删除容器链接但保留容器
    * `-v`: 删除容器挂在的容器
* 导入容器：`docker import test.tar - test/ubuntu:v1.0`
* 导出容器：
    * `docker export -o test.tar containerid`
    * `docker export containerid > test.tar`

### 操作仓库

* 搭建本地仓库：
    * `docker run -d -p 5000:5000 registry`
        * 自动下载并启动一个 registry 容器，仓库默认创建在容器的 `/tmp/registry`
        * 监听端口为 5000
    * `docker run -d -p 5000:5000 -v /opt/data/registry:/tmp/registry registry`
        * 设置放在 `/opt/data/registry` 下
* 查看仓库中的镜像：
    * `curl http://10.0.2.2:5000/v1/search`

### 数据管理

* 创建一个数据卷：`docker run -d -P --name web -v /webapp training/webapp python app.py`
    * `-P` 将容器服务暴露的端口，是自动映射到本地主机的临时端口
    * `-v` 创建一个容器卷
* 挂在一个主机目录作为数据卷：`docker run -d -P --name web -v /src/webapp:/opt/webapp training/webapp python app.py`
    * 加载主机目录 `/src/webapp` 目录到 `/opt/webapp`
    * 再加载目录之后可设置权限，如 `-v /src/webapp:/opt/webapp:ro` 设置 `read-only` 权限
* 挂在一个本地文件作为数据卷（不推荐）：`-v ~/.bash_history:/.bash_history`
    * 使用文本编辑工具时，可能造成 inode 改变， Docker 1.1.0 起会导致错误
* 数据卷容器：
    * `docker run -it -v /dbdata --name dbdata ubuntu`
        * 创建数据卷容器 dbdata 并创建一个数据卷挂在到 /dbdata
    * `docker run -it --volumes-from dbdata --name db1 ubuntu`
    * `docker run -it --volumes-from dbdata --name db2 ubuntu`
        * db1 和 db2 挂载同一个数据卷到相同的 dbdata
    * `docker run -d --name db3 --volumes-from db1 training/postgres`
        * 从已经挂载了容器卷的容器来挂在数据卷
        * 挂在数据卷容器自身不需要保持在运行状态
* 迁移数据：
    * 备份：`docker run --volumes-from dbdata -v $(pwd):/backup --name worker ubuntu tar cvf /backup/backup.tar /dbdata`
    * 恢复：
        * `docker run -v /dbdata --name dbdata2 ubuntu /bin/bash`
        * `docker run --volumes-from dbdata2 -v $(pwd):/backup busybox tar xvf /backup/backup.tar `

### 端口映射和容器互联

* 使用 `-P`， Docker 随机映射一个 49000~49900 的端口到内部容器开放的网络端口：
    * `docker run -d -P training/webapp python app.py`
* 映射所有接口地址：`-p 5000:5000`
    * 多次使用可同时映射多个端口
* 映射到指定地址的指定端口： `-p 127.0.0.1:5000:5000`
    * 指定 udp 端口：`-p 127.0.0.1:5000:5000/udp`
* 查看端口映射配置：`docker port nostalgic_morse 5000`
* 容器互联：`--link name:alias`
    * 要链接的容器名字：链接的别名
    * `docker run -t -i --rm --link db:db training/webapp /bin/bash`
    * `cat /etc/hosts`: 查看链接信息
    * `ping db`: 测试链接

### Dockerfile

* 字段：
```
FROM #基础镜像
MAINTAINER #维护者信息
RUN #运行命令
CMD #容器启动时默认执行的命令
LABEL #生成镜像的元数据标签信息
EXPOSE #声明镜像内服务所监听的端口
ENV #指定环境变量 
ADD #复制<src>路径下的内容到容器中的<dest>的路径下
COPY #复制本地主机<src>路径下的内容到镜像中的<dest>的路径下（推荐）
ENTRYPOINT #镜像默认入口
VOLUME #数据卷挂载点
USER #创建容器运行时的用户名或者UID
WORKDIR #工作目录
ARG #参数
ONBUILD #配置当所创建的镜像作为其他镜像的基础镜像时，所执行的创建操作指令
STOPSIGNAL #容器退出的信号值
HEALTHCHECK #如何进行健康检查
SHELL #指定使用 shell 时的默认 shell 类型
```

* 创建镜像：`docker build -t built_repo/first_image /tmp/docker_builer/`
    * Dockerfile 放在 /tmp/docker_builder
    * 读取指定目录下（包含子目录）的 Dockerfile，并将该路径下所有内容发给 Docker 服务器，由服务端来创建镜像。除非有需要，建议防止 Dockerfile 的目录为空
    * `-f`: 使用非内容路径下的 Dockerfile，指定其路径
* 使用 `.dockerignore` 来让 Docker 忽略匹配模式路径下的目录和文件
