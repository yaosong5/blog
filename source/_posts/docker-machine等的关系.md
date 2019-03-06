---
title:  Docker-machine、engine、swarm集群等相关
date: 2018年06月21日 22时15分52秒
tags:  [Docker]
categories: 原理理解
toc: true
---
[TOC]

# 目标：

  搞懂docker的工作机制，（网络机制后面来搞懂）
  docker的集群，监控机制

  如何对镜像更改更小的配置，对外部文件进行修改读取就可以运行docker容器
【参考】<https://toutiao.io/posts/rgo564/preview，有创建swarm集群的命令，通过ip加入集群的命令>

Docker Engine：作为 Docker 镜像构建与容器化启动的工作引擎；
Docker Machine：安装与管理 Docker Engine 的工具；
Docker Swarm：是 Docker 自 1.12 后自带的集群技术，将多个独立的 Docker Engine 利用 Swarm 技术进行集群化管理；
简单来讲，Docker 集群的实现是通过 Docker Machine 在多主机或虚拟机上创建多个 Docker Engine，在利用 Docker Swarm 以某个 Engine 作为集群的初始管理节点，其他 Engine 以管理或工作节点的方式加入，形成完成的集群环境。



swarm 是集群工具，compose 是运行容器的快捷工具 本来就能结合使用，他们之间不冲突，
再加 machine 可以创建多个 docker 虚拟环境（如果你没有多台服务器测试的话）三者一起，
建立开发测试集群环境简单方便快捷！

http://dockone.io/question/160
Machine：解决的是操作系统异构安装 Docker 困难的问题，没有 Machine 的时候，CentOS 是一种，Ubuntu 又是一种，AWS 又是一种。有了 Machine，所有的系统都是一样的安装方式。

Swarm：我们有了 Machine 就意味着有了 docker 环境，但是那是单机的，而通常我们的应用都是集群的。这正是 Swarm 要做的事情，给你提供 docker 集群环境和调度策略等。

Compose：有了环境，我们下一步要做什么？部署应用啊。然后我们需要 docker run image1、docker run image2... 一次一次不厌其烦的重复这些操作，每次都写大量的命令参数。Compose 简化了这个流程，只需要把这些内容固话到 docker-compose.yml 中。

目前 Machine、Swarm、Compose 已经可以结合使用，创建集群环境，简单的在上面部署应用。但是还不完善，比如对于有 link 的应用，它们只能跑在 Swarm 集群的一个机器上，即使你的集群有很多机器。可以参考我的另一个问题。

SocketPlane 是 Docker 最近收购的产品，猜想应该是为了强化 Docker 的网络功能，比如提供原生跨主机的网络定制、强化 Swarm 和 Compose 的结合等。

## docker挂载

https://www.cnblogs.com/ivictor/p/4834864.html 宿主机目录和容器目录的关系

## Docker-engine ：
> 当人们说 “Docker” 时，他们通常意味着 Docker Engine，即由 Docker 守护程序组成的客户端 - 服务器应用程序，指定用于与守护程序进行交互的接口的 REST API 以及与守护程序（通过 REST API 包装器）通信的命令行界面（CLI）客户端。Docker Engine 从 CLI 接受docker命令，例如docker run <image>，docker ps列出运行的容器，docker images以列出镜像等



## Docker-machine:
> Mac OS X 默认是没有虚拟机的，所以要使用 Docker Machine，需要事先安装好 VirtualBox，这在 Docker 的官方文档上有相关说明
> Docker Machine 是一个工具，用来在虚拟主机上安装 Docker Engine
> 并使用 docker-machine 命令来管理这些虚拟主机
>  Docker Machine
> Docker Machine 是用来配置和管理 Docker Engine 的工具。
> Docker Machine 主要有两个用途：一是为了能在旧的 Mac 和 Windows 系统上使用 Dokcer。二是为了可以管理远程的 Docker。

要知道，Docker Engine 是只能在 Linux 内核上运行的。
为什么 Docker Machine 可以使得在旧的 Mac 和 Windows 系统上也能使用 Docker 呢？因为，安装 Docker Toolbox 会安装 Docker Machine 和 Virtual Box 虚拟机，它会配置一个 Docker Engine 到一个虚拟机里的主机上（Linux 内核，默认名字是 default，又称托管主机），后续 Docker 容器就是运行在这个托管主机里的 Docker Engine 之上。

将 Machine CLI 指向正在运行的托管主机，就可以直接该运行docker命令到该主机上。这里貌似东西不少：docker-machine env default会显示 default 主机的环境信息以及对应的配置命令指引：

```bash
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.99.100:2376"
export DOCKER_CERT_PATH="/Users/Eason/.docker/machine/machines/default"
export DOCKER_MACHINE_NAME="default"
```
Run this command to configure your shell:
```
eval "$(docker-machine env default)"
```
提示很明白了，在 SHELL 里运行 eval "$(docker-machine env default) 就可以把当前 Machine CLI 指向 default 主机了。好处就是，当我们有多个主机时，这样一行命令就能切换环境指向不同的主机，然后后续运行的docker命名也是运行到对应的 Docker Enigne 上。

注：Docker for Mac 和 Docker for Windows 以及 (https://docs.docker.com/toolbox/overview/) 都是自带 Docker Machine 工具的。

这一节的个人理解，也不知道我写得大家看不看得懂。其实主要辨析点就是：Container 容器是 Image 镜像运行在内存中的实例，Container 容器要运行在 Docker Engine 之上，然而 Docker Engine 仅限运行在 Linux 内核之上。Docker Machie 是配置和管理 Docker Engine 的工具，并且可以为旧版的 Mac 和 Windows 系统提供 Docker Engine 的运行环境

## Docker-machine:

```
Docker Machine 是一个工具，用来在虚拟主机上安装 Docker Engine
并使用 docker-machine 命令来管理这些虚拟主机
```


## swarm
swarm 是集群工具，compose 是运行容器的快捷工具 本来就能结合使用，他们之间不冲突，
再加 machine 可以创建多个 docker 虚拟环境（如果你没有多台服务器测试的话）三者一起，

<http://dockone.io/question/160>

Machine：解决的是操作系统异构安装 Docker 困难的问题，没有 Machine 的时候，CentOS 是一种，Ubuntu 又是一种，AWS 又是一种。有了 Machine，所有的系统都是一样的安装方式。

Swarm：我们有了 Machine 就意味着有了 docker 环境，但是那是单机的，而通常我们的应用都是集群的。这正是 Swarm 要做的事情，给你提供 docker 集群环境和调度策略等。

Compose：有了环境，我们下一步要做什么？部署应用啊。然后我们需要 docker run image1、docker run image2... 一次一次不厌其烦的重复这些操作，每次都写大量的命令参数。Compose 简化了这个流程，只需要把这些内容固话到 docker-compose.yml 中。

目前 Machine、Swarm、Compose 已经可以结合使用，创建集群环境，简单的在上面部署应用。但是还不完善，比如对于有 link 的应用，它们只能跑在 Swarm 集群的一个机器上，即使你的集群有很多机器。可以参考我的另一个问题。

SocketPlane 是 Docker 最近收购的产品，猜想应该是为了强化 Docker 的网络功能，比如提供原生跨主机的网络定制、强化 Swarm 和 Compose 的结合等。

## docker挂载

<https://www.cnblogs.com/ivictor/p/4834864.html> 宿主机目录和容器目录的关系

# swarm集群搭建的步骤

首先 `docker-machine ssh`
在default 使用命令  docker swarm init
