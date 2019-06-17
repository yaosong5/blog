---
title:  部署博客到云服务器
date: 2018年05月21日 00时54分48秒
tags:  [Hexo]
categories: 博客
toc: true
---



![](https://img.gangtieguo.cn/006tNbRwgy1fu566e9ybjj30kg05q0t0.jpg)

简单记录转移到博客到云服务器
<!-- more -->

#  原理及准备
>我们在自己的电脑上写好博客, 使用 git 发布到代码仓库进行备份, git 仓库接收到 push 请求后, 使用 webhook 配合 nodejs 自动进行服务器端页面的更新.
## 准备
安装Git和NodeJS (CentOS 环境)

``` 
yum install git
```
安装NodeJS

``` vim
curl --silent --location https://rpm.nodesource.com/setup_5.x | bash -
```

# 服务器构建
## webhook方式
服务器端的” 钩子”
我们借助一个 node 插件 github-webhook-handler 来快速完成配合 github webhook 的操作, 其他 git 平台也有相应的插件, 如配合 coding 的 coding-webhook-handler.

### 监听脚本
>我们借助一个 node 插件 *github webhook-handler*来快速完成配合 github webhook 的操作, 其他 git 平台也有相应的插件, 如配合 coding 的 coding-webhook-handler.

使用 `npm install -g github-webhook-handler` 命令来安装到服务器端.
conding则为`npm install -g coding-webhook-handler` 


>切换到服务器站点目录，如我的是 /root/blog,新建一个public目录，将你的github仓库中的master分支pull到该目录中，这个目录作为这个博客的根目录了

```bash
cd /root/blog
mkdir public 
cd public 
git init
git remote add origin https://github.com/yaosong5/yaosong5.github.io
git pull origin master
```


然后我们创建一个webhooks.js文件，将以下的内容粘贴，这相当于Node.js **服务器**的代码构建



``` javascript
var http = require('http')
var createHandler = require('github-webhook-handler')
var handler = createHandler({ path: '/', secret: 'yao' })

function run_cmd(cmd, args, callback) {
  var spawn = require('child_process').spawn;
  var child = spawn(cmd, args);
  var resp = "";

  child.stdout.on('data', function(buffer) { resp += buffer.toString(); });
  child.stdout.on('end', function() { callback (resp) });
}

http.createServer(function (req, res) {
  handler(req, res, function (err) {
    res.statusCode = 404
    res.end('no such location')
  })
}).listen(7777)

handler.on('error', function (err) {
  console.error('Error:', err.message)
})

handler.on('push', function (event) {
  console.log('Received a push event for %s to %s',
    event.payload.repository.name,
    event.payload.ref);
    run_cmd('sh', ['./deploy.sh',event.payload.repository.name], function(text){ console.log(text) });
})
```

注意上段代码中第 3 行 { path: '/', secret: '改为你的secret' } 中 secret 可以改为你喜欢的口令, 这口令将在下面的步骤中起到作用 ,配置github webhooks的时候填入的口令, 请留意. 第 19 行 listen(7777) 中 7777 为监听程序需要使用的端口.

### 执行脚本
上面的 javascript 代码是用来捕捉 github 发来的信号并发起一个执行 ./deploy.sh 的脚本, 接下来我们还需要写 deploy.sh 的内容.

``` bash
#!/bin/bash

WEB_PATH='/root/blog/public'

echo "Start deployment"
cd $WEB_PATH
echo "pulling source code..."
git reset --hard origin/master
git clean -f
git pull
git checkout master
echo "Finished."
```

将以上代码的第 3 行改为你服务器中的实际目录. 接下来只需要开启监听就可以了.


tips: 在此之前你可以使用 node webhook.js 来测试一下监听程序是否能够正常运行.
我在这里碰到了一个 node 环境变量的问题, 读取不到 github-webhook-handler 这个模块, 找了很多办法也没有解决, 后来我直接在项目根目录的上级目录安装了这个模块, 问题就解决了.

>cd /root/blog
>npm install github-webhook-handler
>npm 会从当前目录依次向上寻找含有 node_modules 目录并访问该模块.

### 普通方式运行 webhook.js
利用 Linux 提供的 nohup 命令，让 webhooks.js 运行在后台

```
nohup node webhook.js > deploy.log &
```
###  Forever方式运行webhook.js
>我在实际使用的时候发现，我的 Node 服务器时不时会自动停掉，具体原因我暂时还没有弄清楚。不过似乎很多人都遇到了这样的困扰，要解决这个问题，forever 是个不错的选择。借助 forever 这个库，它可以保证 Node 持续运行下去，一旦服务器挂了，它都会重启服务器。

安装 forever：
` npm install -g forever`
运行：
``` 
cd { 部署服务器的根目录 }
 nohup forever start webhook.js > deploy.log &
```
 Ubuntu 中原本就有一个叫 node 的包。为了避免冲突，在 Ubuntu 上安装或使用 Node 得用 nodejs 这个名字。而 forever 默认是使用 node 作为执行脚本的程序名。所以为了处理 Ubuntu 存在的这种特殊情况，在启动 forever 时得另外添加一个参数：(其它则忽略)

`forever start webhook.js -c nodejs`
### Github配置webhooks
配置好 Webhook 后，Github 会发送一个 ping 来测试这个地址。如果成功了，那么这个 Webhook 前就会加上一个绿色的勾；如果你得到的是一个红色的叉，那就好好检查一下哪儿出问题了吧！
## git-hook方式
可采用一种更为简单的部署方式
>这种方式和webhook可二选一

### 服务器上建立git裸库

创建一个裸仓库，裸仓库就是只保存git信息的Repository, 首先切换到git用户确保git用户拥有仓库所有权
一定要加 --bare，这样才是一个裸库。

```
cd 
git init --bare blog.git
```
### 使用 git-hooks 同步网站根目录


在这里我们使用的是 post-receive这个钩子，当git有收发的时候就会调用这个钩子。 在 ~/blog.git 裸库的 hooks文件夹中，
新建post-receive文件。
>vim ~/blog.git/hooks/post-receive

填入以下内容

``` bash
#!/bin/sh
git --work-tree=/root/blog/public --git-dir=/root/blog.git checkout -f
```

*work-tree=/root/blog/public这个目录是网站的网页文件目录，--git-dir=/root/blog.git目录为裸库地址，裸库监听git提交会将文件提交到网页目录*
保存后，要赋予这个文件可执行权限`chmod +x post-receive`
### 配置博客根目录_config.yml
完成自动化部署
打开 _config.yml, 找到 deploy
```
deploy:
    type: git
    repo: 用户名@SERVER名:/home/git/blog.git（裸库地址）    //<repository url>
    branch: master            //这里填写分支   [branch]
    message: 提交的信息         //自定义提交信息 (默认为 Site updated: {{ now('YYYY-MM-DD HH:mm:ss') }})
```

# Nginx服务
npm 安装nginx
启动nginx 

``` bash
service nginx start
```
`nginx -t` 查看nginx配置文件
若nginx服务启动，访问报403错误
 则将首行 user nginx 改为user root

``` 
vim /etc/nginx/nginx.conf
server {
    listen          80;  # 监听端口
    server_name     47.98.141.252:80 gangtieguo.cn wwww.gangtieguo.cn;  # 你的域名
    location / {
        root        /root/blog/public;
        index       index.html;
    }
}
```
**重载 nginx，使配置生效**

```nginx -s reload```

参考
[Hexo 静态博客搭建并实现自动部署到远程 vps](https://www.jianshu.com/p/b29a2e40501f)
[将 Hexo 博客发布到自己的服务器上](https://blog.mutoe.com/2017/deploy-hexo-website-to-self-server/)
[利用 Github 的 Webhook 功能和 Node.js 完成项目的自动部署](https://www.jianshu.com/p/e4cacd775e5b)
[Webhook 实践 —— 自动部署](https://segmentfault.com/a/1190000003908244)
[Hexo 快速搭建静态博客并实现远程 VPS 自动部署](http://imlianer.com/a/deploy-hexo-on-vps)
[阿里云 VPS 搭建自己的的 Hexo 博客](https://segmentfault.com/a/1190000005723321)