---
title: Docker基本操作
date: 2019-05-07 11:10:23
categories:
- docker
tags: [Docker, 命令]
comments: true
---

记录一些常用的docker命令

## 基本查看命令

查看所有正在运行的容器

```shell
docker ps
```

查看所有容器

```shell
docker ps -a
```

查看所有镜像

```shell
docker images
```

## 容器基本操作

从镜像文件创建一个容器

```shell
docker run [options] image
# -d 后台运行
# -it ... /bin/bash 终端连接
# --name 设置容器名称
# --net=host/bridege... 设置网络
```

启动某个已存在的容器

```shell
docker start id/name
```

停止某个容器

```shell
docker stop id/name
```

删除某个容器

```shell
docker rm id/name
```

停止所有容器

```shell
docker stop $(docker ps -a -q)
```

删除所有容器

```shell
docker rm $(docker ps -a -q)
```

连接容器

```shell
docker exec -it id/name /bin/bash
```

保存容器的修改，提交到镜像

```shell
docker commit id name
```

保存容器为文件

```shell
docker export id > name
# docker export 3b86889153b0 > dk-test.tar
```

加载容器文件

```shell
docker import name newname
# docker import dk-test.tar dk-test-v2
```

## 镜像操作

加载镜像

```shell
docker load file_name < image_name
```

保存镜像到文件

```shell
docker save id > name
```

删除镜像

```shell
docker rmi id
```
