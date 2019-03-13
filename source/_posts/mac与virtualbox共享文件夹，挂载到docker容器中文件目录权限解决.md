---
title: mac与virtualbox共享文件夹，挂载到docker容器中文件目录权限解决
date: 2018年08月06日 22时15分52秒
tags: [Docker,Mac]
categories: 问题解决
toc: true
---

[TOC]



<!-- more -->



# 共享文件夹的权限问题

参考：[Virtualbox设置共享文件夹](http://yukai.space/2018/05/02/VirtualBox%E5%85%B1%E4%BA%AB%E6%96%87%E4%BB%B6%E5%A4%B9/)

由于共享文件夹并不是虚拟机的本地目录，我们在虚拟机中可以配置共享文件夹的权限是有限的。

手动挂载或自动挂载的目录，所属用户默认为root，组为vboxsf，并且使用 `chmod chown` 等命令是无法改变的。

如果想要配置挂载目录的权限，需要在手动挂载的时候指定一些选项：

```shell
// uid gid指定挂载目录的所属用户和组
sudo mount -t vboxsf -o uid=500,gid=500  <folder name given in VirtualBox>
// fmode指定文件权限，dmode指定目录权限
// 注意，若同时指定挂载目录的所属用户和组，则fmode和dmode选项失效
sudo mount -t vboxsf -o fmode=700,dmode=700  <folder name given in VirtualBox>
```

我使用的命令为：

```shell
sudo mount -t vboxsf -o uid=500,gid=500 Y  /Users/yaosong/Yao
```

此处Y为我设置的共享路径

![共享路径](https://ws3.sinaimg.cn/large/006tKfTcgy1g0ztnzxosnj310o09wgm5.jpg)





踩过的坑：

> 之前由于在挂载时，使用的 **sudo mount -t vboxsf  Yao  /Users/yaosong/Yao**
>
> 并未设置所属用户组及所属用户，然后在虚拟机中使用chown，无论如何都未成功，
>
> ![使用sudo mount -t vboxsf  Yao  /Users/yaosong/Yao](https://ws4.sinaimg.cn/large/006tKfTcgy1g0zs97hho1j3130078gph.jpg)
>
> 看到上一篇博文，茅塞顿开
>
> 使用**sudo mount -t vboxsf -o uid=500,gid=500 Y  /Users/yaosong/Yao**过后
>
> > 由于elk用户对应用户id，用户组id为500，
> >
> > ![elk用户对应用户id，用户组id为500](https://ws4.sinaimg.cn/large/006tKfTcgy1g0zsdwrv03j30ma0160st.jpg)
>
> 可以得到一下所属用户和用户组：
>
> ![挂载时设置用户和组过后](https://ws2.sinaimg.cn/large/006tKfTcgy1g0zsbf3r1kj312u07odjp.jpg)
>
> 此处设置用户id和组id，下文也要用到
>
> 


# 挂载到容器的文件目录与宿主机的设置关系

   参考：[关于Docker目录挂载的总结](https://www.cnblogs.com/ivictor/p/4834864.html)，[Docker Volume - 目录挂载以及文件共享](https://kebingzao.com/2019/02/25/docker-volume/)

   之所以设置共享文件夹权限时设置用户id和组id，还有一个重要原因：

   ​	原来，宿主与容器的UID有关系，UID，即“用户标识号”，是一个整数，系统内部用它来标识用户。一般情况下它与用户名是一一对应的。即为：在宿主中UID为多少，那么在容器中的所属用户所属组就为容器中和宿主UID相对应的用户。

   1. 启动容器时挂载目录

      ```shell
      docker run -itd --net=br --name elk1 --privileged=true -v /Users/yaosong/Yao/share/source/es1:/usr/es  --hostname elk1 --ip=192.168.33.16 yaosong5/elk:1.0 &> /dev/null
      ```

      可知宿主机/Users/yaosong/Yao/share/source/es1目录挂载到容器/usr/es中

      ​

   2. 在容器内设置权限：

      ```shell
      chown -R elk.elk $ES_HOME
      ```

      注意是点不是冒号，区别我还没研究


# 查看结果

![大功告成](https://ws2.sinaimg.cn/large/006tKfTcgy1g0ztcg8e6jj30w20eu0yg.jpg)



# 参考文章

[Virtualbox设置共享文件夹](http://yukai.space/2018/05/02/VirtualBox%E5%85%B1%E4%BA%AB%E6%96%87%E4%BB%B6%E5%A4%B9/)

[关于Docker目录挂载的总结](https://www.cnblogs.com/ivictor/p/4834864.html)

[Docker Volume - 目录挂载以及文件共享](https://kebingzao.com/2019/02/25/docker-volume/)