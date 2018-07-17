---
title: 手把手教你部署NUXT项目
---

下载好脚手架后我们在开发模式下直接使用 <code>npm run dev</code>即可观看、调试我们的项目页面。

项目完毕就准备部署预发布走测试流程。

[nuxt](https://zh.nuxtjs.org)的部署可以分为两种,一种是静态应用（站点）部署，一种动态应用部署（服务端渲染应用部署）。[可以参考这里](https://zh.nuxtjs.org/guide/commands)

<!--more-->
# 静态应用部署

静态部署没什么好说的，和hexo一样，写好了内容，执行一下<code>nuxt generate</code>
然后把根目录下生成的dist文件丢在服务器下即可。

可利用下面的命令生成应用的静态目录和文件：
``` bash
npm run generate
```

这个命令会创建一个 dist 文件夹，所有静态化后的资源文件均在其中。

# 服务端渲染应用（动态）部署
部署 Nuxt.js 服务端渲染的应用不能直接使用 nuxt 命令，而应该先进行编译构建，然后再启动 Nuxt 服务，可通过以下两个命令来完成：

``` bash
nuxt build
nuxt start
```

但是这个命令不能再远端使用,所以我们一般把这个命令放在package.json里通过npm进行执行。

推荐的 package.json 配置如下：

``` bash
{
  "name": "my-app",
  "dependencies": {
    "nuxt": "latest"
  },
  "scripts": {
    "dev": "nuxt",
    "build": "nuxt build",
    "start": "nuxt start"
  }
}
```
呐，官方的介绍就这么多，很简单吧，很容易吧。

但是你发现，这样起的服务，使用的是你本地的ip和端口。也就是你在本地开发的时候使用什么端口，这里启动的就是什么端口。我们需要的是，在本地启动的时候，使用本地ip和端口。在生产环境启动的时候就是用环境的ip和端口
那么我们就需要到<code>server/index.js</code>这里改一下

``` bash

const host = process.env.HOST || '0.0.0.0'
const port = process.env.PORT || 8090

```

好了，这样你启动的项目就是你环境的ip端口了。

但是，当你关闭远端的窗口后你会发现你的项目被kill了。就像你在本地开发的时候一样，使用<code>npm run dev</code>启动服务后，按一下Ctrl + C项目就会被kill。

所以我们需要一个守护我们这个进程的东西，就算我离开了远端或者本地开发的时候不小心在控制台按下Ctrl + C，项目还是能正常。

上网查了一下守护进程的工具有很多，因为公司使用的是[pm2](http://pm2.keymetrics.io/docs/usage/pm2-doc-single-page/)，使用我这里就只针对[pm2](http://pm2.keymetrics.io/docs/usage/pm2-doc-single-page/)进行部署。
还是简单的举例一下[pm2](http://pm2.keymetrics.io/docs/usage/pm2-doc-single-page/)的优势吧！

<div class="tip">
<p>1、内建负载均衡（使用Node cluster 集群模块）</p>

<p>2、守护进程，后台运行。
<p>3、0秒停机重载，我理解大概意思是维护升级的时候不需要停机.</p>
<p>4、具有Ubuntu和CentOS 的启动脚本</p>
<p>5、停止不稳定的进程（避免无限循环）</p>
<p>6、控制台检测</p>
<p>7、提供 HTTP API</p>
<p>8、远程控制和实时的接口API ( Nodejs 模块,允许和PM2进程管理器交互 )</p>
<p>9、简洁明了的可视化窗口和调试信息。</p>
</div>

上面都是网上找的优点，在使用过程中我只有明显使用到1、2、3、6、9这几个而已。

1、负载均衡。你完全可以使用Node cluster建立自己的子线程，同时开启多个进程来监听同一个端口，分发http请求处理。但是都有pm2帮你做了你何必又自己做呢？所以对于node的Node cluster作为了解即可啦。

2、守护进程，上面已经说了离开了远端项目还是能正常使用。我们一般启动的时候，pm2会默认根据机器具有几核，智能的给你开启多核。除非你自己手动限制开启。pm2 start xxx.js -i 就能启动你需要的server服务，然后-i 指定多开数量,默认为自动根据机器配置开启。

3、0秒停机重载。这个是不是和负载均衡有点异曲同工？


# pm2配置

### 生成脚本
就像webpack的webpack.config.js和npm包管理的package.json一样。
pm2的配置脚本也是需要的，我们可以在终端pm2 ecosystem

会在工程下面生成一个ecosystem.config.js。（当然你可以手动新建一个该文件）

### 修改脚本

根据项目简简单单的编写一下脚本
``` bash
{
    "apps": [{
        "name": "my-app",
        "max_memory_restart": "300M",
        "script": "./build/main.js",
        "out_file": "/home/efun/logs/my-app-logs/my-app-out.log",
        "error_file": "/home/efun/logs/my-app-logs/my-app-error.log",
        "merge_logs": true,
        "instances": 2,
        "exec_mode": "cluster",
        "env": {
            "NODE_ENV": "development"
        },
        "env_production": {
            "NODE_ENV": "production"
        }
    }]
}
```

当然，你要是想进行更为复杂的操作你可以企业看官网;[呐，地址都给你了](http://pm2.keymetrics.io/docs/usage/quick-start/)

配置好了脚本怎么使用？

如果你不需要使用npm命令启动，你完全可以使用pm2的命令启动。后面跟你项目的入口js文件。
pm2 start xxx.js

### 使用脚本启动项目
当然，启动nuxt（说白了,使用node作为前端服务器，都应该使用这种启动方法），结合package.json的script脚本进行操作。

下面是package.json可以作为参考

``` bash
"scripts": {
      "dev": "backpack dev",
      "build": "nuxt build && backpack build",
      "start": "cross-env NODE_ENV=production node build/main.js",
      "precommit": "npm run lint",
      "lint": "eslint --ext .js,.vue --ignore-path .gitignore .",
      "prod": "PORT=34004 pm2 start ecosystem.json --env production",
      "pre-prod": "PORT=34104 pm2 start ecosystem-test.json --env production",
      "pre-restart": "pm2 restart my-app",
      "prod-restart": "pm2 restart my-app",
      "pre-stop": "pm2 delete my-app",
      "prod-stop": "pm2 delete my-app"
  },

```
启动预发布环境就执行 npm run pre-prod启动项目，执行的是ecosystem-test.json，通过PORT把端口传过去进行区分预发布和正式环境。
启动预正式环境就执行 npm run prod启动项目，执行的是ecosystem.json。


查看程序运行：
[efun@Nodejskr75 pf-kr-pre]$ pm2 list


停用运行的程序
pm2 stop my-app

[efun@Nodejskr75 pf-kr-pre]$ pm2 stop my-app
[PM2] Applying action stopProcessId on app [my-app](ids: 51,52)
[PM2] [pf-kq-pre](51) ✓
[PM2] [pf-kq-pre](52) ✓


停用后想重启程序，可以使用pm2命令reload。 pm2 reload my-app
也可以使用npm命令 npm run pre-restart (实际上还是执行pm2 reload my-app)


stop并不能清理程序，也不会释放端口，而是要delete：

[azuo1228@Server Meanjs-MMM]$ pm2 delete my-app
[PM2] Applying action deleteProcessId on app [all](ids: 0,1)
[PM2] [server](1) ✓
[PM2] [server](0) ✓

 Use `pm2 show <id|name>` to get more details about an app



好咯，说了一堆pm2。下面说说使用pm2部署遇到的几个坑点。

#坑一：
部署到线上，需要执行<code>npm run build</code>编译成静态文件（同时压缩js、css等） 。

或者按照文档说的执行 <code>nuxt build</code>,这样项目才读取的才是硬盘里的文件。这一点和使用vue-cli脚手架部署一样。

有次就是忘记执行了，导致访问页面路由一直在loading，但是就是不出来内容，或者出来个502.
这里最好就一起写在npm命令的脚本里，以免下次又漏了。

因为公司是使用jenkins进行自动化部署，所以我这个build是放在jenkins进行操作的。

所以在提交git后，使用compile命令进行编译，rsync进行同步，restart进行重启，stop进行停用，start进行启动。

因为并不是所有公司都使用jenkins发布，所以这里就不多说。不过jenkins真的好处多多，好用又方便。


#坑二：

一个域名被解析到一个静态服务器，上面有很多静态html等各种资源(域名根目录未被使用)。后来需要加一个平台上去，同时这个平台使用该域名(根目录映射平台首页)。

这个时候，就需要运维同事的配合了。在处理前端访问url的时候，先经过Nginx的应用层，然后Nginx先判断改url是否带有静态资源特征，也就是.js .css .jpg .png ....。根据不同的资源路径或者路由进行分发指向node服务器还是原有的静态服务器。

这里上个Nginx的过滤规则：

``` bash
location ~* ^.+.(jpg|jpeg|gif|css|png|js|ico|txt|srt|swf)$ {
    root /var/www/mywebsite/site/;
    expires 30d;
}
```

现在nuxt部署后。

首页路由为：www.mywebsite.com ,页面依赖的其中一个js路径为https://www.mywebsite.com/_nuxt/manifest.e6cf8cf6462faca1a03e.js


这个时候，页面上所有的js引用都是404

原因是因为运维同事的过滤规则导致页面读取js静态资源，跑到静态服务器上面找去了。当然找不到。因为这个js文件我们部署build的时候，是在node服务器上的。

这个时候需要在Nginx的过滤规则上加多权重更高的一个过滤。
``` bash
location ~^/_nuxt/ {
	alias /mywebsite/.nuxt/dist/;
}
```


上面是Nginx具有中间应用层的情况下的规则。那如果不需要应用层Nginx规则则为如下：

``` bash
location /_nuxt/ {
	alias /var/www/mywebsite/.nuxt/dist/;
}

location ~* ^.+.(jpg|jpeg|gif|css|png|js|ico|txt|srt|swf|woff|woff2)$ {
	rewrite	^/_nuxt(/.*) $1 break;
	root /var/www/mywebsite/.nuxt/dist;
	expires 30d;
}
``` 

这个在nuxt的issue地址有这个[提问](https://github.com/nuxt/nuxt.js/issues/2795)



暂时说这么多，下次想到有补充再补充一下。

下一篇应该就是整理webpack的单页面和多页面的配置了。