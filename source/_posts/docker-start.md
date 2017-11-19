---
title: Docker 入门
date: 2017年11月19日21:19:59
thumbnail: http://blog.phusion.nl/wp-content/uploads/2013/11/docker.png
tags: 
 - Docker
 - Mysql
categories: 
 - Docker
---

# Docker是什么，能做什么？
Docker是一种虚拟化技术，是在操作系统上虚拟出的一个可移植的轻量容器，这个容器有独立的进程、权限、资源、网络等等。Docker能

 1. 隔离应用依赖
 2. 创建应用镜像并进行复制
 3. 创建容易分发的即启即用的应用
 4. 允许实例简单、快速地扩展
 5. 测试应用并随后销毁它们
 6. PaaS

<!--more-->
# Docker对安装环境的要求：
1 64位CPU
2 3.8+内核
3 存储驱动，
4 内核开启cgroup和命名空间

# Docker里面的核心组件：
Docker 客户端和服务端 &emsp;&emsp; 借助于微服务来理解最合适
Docker 镜像   &emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;  相当于maven jar包
Docker Registry       &emsp;&emsp;&emsp;&emsp;&emsp; 可以理解成maven仓库
Docker 容器 &emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;  一种镜像格式，一系列标准的操作，一个执行环境。

理解Docker，可以参考java，一次编译到处运行。可以参考maven里面的镜像、仓库、jar包、坐标等概念

# 使用Docker安装mysql环境
查找Docker Hub上的mysql镜像
```shell
docker search mysql
```
这里我们拉取官方的镜像，tag为5.5 [官方地址查看版本](https://hub.docker.com/_/mysql/)
```shell
docker pull mysql:5.5
```
等待下载完成后，我们就可以在本地镜像列表里查到REPOSITORY为mysql,Tag为5.5的镜像。
```shell
C:\Users\Wither>docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
mysql               5.5                 a6f9c7888779        2 weeks ago         257MB
```
使用mysql镜像，运行容器
```bash
简单命令，指定端口密码和名字
docker run -p 3306:3306 --name some-mysql -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.5

docker run -p 3306:3306 --name some-mysql -v $PWD/conf/my.cnf:/etc/mysql/my.cnf -v $PWD/logs:/logs -v $PWD/data:/mysql_data -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.5
```

命令说明：
> -p 3306:3306：将容器的3306端口映射到主机的3306端口
> -v $PWD/conf/my.cnf:/etc/mysql/my.cnf：将主机当前目录下的conf/my.cnf挂载到容器的/etc/mysql/my.cnf
> -v $PWD/logs:/logs：将主机当前目录下的logs目录挂载到容器的/logs
> -v $PWD/data:/mysql_data：将主机当前目录下的data目录挂载到容器的/mysql_data
> -e MYSQL_ROOT_PASSWORD=123456：初始化root用户的密码

查看容器启动情况
```shell
C:\Users\Wither>docker ps
CONTAINER ID IMAGE      COMMAND                 CREATED     PORTS                   NAMES
9ff178ece2c2 mysql:5.5  "docker-entrypoint.."   2 hours ago 0.0.0.0:3306->3306/tcp  some-mysql
```

# Docker的常用命令
容器方面：
```shell
运行容器：
sudo docker run -i -t ubuntu /bin/bash

查看容器列表：
docker ps -a

容器命名：
sudo docker run --name container_name -i -t ubuntu /bin/bash

启动、停止、重启：
sudo docker start|stop|restart some-mysql

创建守护容器：
sudo docker run --name daemon_dave -d ubuntu /bin/bash -c "while true;do echo hello world; sleep 1;done"

带时间查看日志：
sudo docker logs -ft some-mysql

查看容器内的进程：
sudo docker top some-mysql

容器内运行进程：
sudo docker exec -d daemon_dave touch /etc/new_config_file

交互式运行进程：
sudo docker exec -t -i daemon_dave /bin/bash

停止守护进程：
sudo docker stop daemon_dave或者sudo docker stop 3wqihds8324

自动重启容器：
sudo docker run --restart=always --name daemon_dave -d ubuntu /bin/bash -c "while true;do echo hello world; sleep 1;done"

查看容器详细信息：
sudo docker inspect daemon_dave

删除容器：
sudo docker rm 823919fdsfdas3

删除所有容器：
docker rm 'docker ps -a -q'
```
镜像方面:
```shell
查看docker镜像
sudo docker images 

从镜像网站上拉取镜像：
sudo docker pull ubuntu:latest

查找镜像：
sudo docker search mysql

推送镜像：
sudo docker push jamtur01/static_web

删除镜像：
sudo docker rmi jamtur01/static_web

通过dockerfile创建镜像：
touch Dockerfile
sudo docker build =t="xxx/xxx:xxx"

查询docker历史：
sudo docker history 2dfshfjksd
```

dockerfile方面：

> 1 CMD：启动时运行的命令 
> 2 ENTRYPOINT：不会被run的指令覆盖。 
> 3 WORKERID：创建新的镜像，设置工作目录 
> 4 ENV：设置环境变量 
> 5 USER：指定运行的用户 
> 6 VOLUME：向镜像添加卷 
> 7 ADD：把文件夹中的内容复制到镜像中 
> 8 COPY：与ADD类似，但是不会解压或者提取 
> 9 ONBUILD：添加触发器

## **参考**
- [非常详细的 Docker 学习笔记](https://www.cnblogs.com/chunguang/p/5656894.html)
- [Docker 安装 MySQL](http://www.runoob.com/docker/docker-install-mysql.html)
- [Docker Hub MySQL](https://hub.docker.com/_/mysql/)
- [Docker Windows安装](https://docs.docker.com/docker-for-windows/install/)