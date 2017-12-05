---
title: Centos7安装Docker
date: 2017-12-05 15:56:59
tags: [Centos,Docker]
categories: Docker
---
> 在工作中,总是避免不了搭建各种开发环境，配置各种组件的环境变量，如果你厌倦了以往的方式，你可以尝试使用Docker，本文就从Docker的安装开始。
<!-- more -->

安装说明
----
>Docker支持以下的CentOS版本：
CentOS 7 (64-bit)
CentOS 6.5 (64-bit) 或更高的版本
前提条件
目前，CentOS 仅发行版本中的内核支持 Docker。
Docker 运行在 CentOS 7 上，要求系统为64位、系统内核版本为 3.10 以上。
Docker 运行在 CentOS-6.5 或更高的版本的 CentOS 上，要求系统为64位、系统内核版本为 2.6.32-431 或者更高版本。

yum安装docker
----

> 
1.使用root权限登陆系统。
>
2.更新系统包到最新。
>
    yum -y update
>
3、安装docker
>
    yum -y install docker
>
4、启动docker服务
>
    systemctl start docker.service
>
5、创建开机启动Docker服务
>
    systemctl enable docker.service
>
6、查看Docker版本号
>
    docker version
>
7、运行hello-world
>
    docker run hello-world

后续
----
后续的组件，我会尽量用Docker的方式来处理，这样既方便快捷，也完美的解决了一台机器中安装某个服务的多个版本，比如es，redis，后面也会在使用中总结更多的用法。
