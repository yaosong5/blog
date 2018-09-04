---
title:  git命令总结
date:
tags:  [git]
categories: 工程框架
toc: true
---



![](https://ws2.sinaimg.cn/large/006tNbRwgy1fu55a5sqb0j313s0cwmxj.jpg)

## 提交

>git add .
>git commit -m " "
>git push origin master
>git push origin master -f
## 拉取
git pull <远程主机名> <远程分支名>:<本地分支名>
如拉取远程的 master 分支到本地 wy 分支：
git pull origin master:wy

## 分支切换
<!-- more -->
查看分支：git branch
创建分支：git branch <name>
切换分支：git checkout <name>
创建 + 切换分支：git checkout -b <name>
合并某分支到当前分支：git merge <name>
删除分支：git branch -d <name>
