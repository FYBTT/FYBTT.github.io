---
layout:     post
title:      在Docker中运行Jekyll
date:       2020-09-20
author:     bear
header-img: img/web.png
catalog:    true
categories: jekyll
tags:       blog docker
--- 
Jekyll ——静态网站生成框架，这个github的静态网站数据就是用Jekyll生成的。但Jekyll的安装过程非常琐碎，直接安装到 Host OS 上是一种污染环境的方法。而Docker提供了不需要自己配置的运行环境，并且这样的方式适用于 Linux 和 Windows。

1. 首先是获取 Jekyll 的最新 Docker 镜像：
```shell
docker pull jekyll/jekyll
```

2. 切换到你的 Jekyll 网站所在目录，执行这条命令启动 Jekyll：
```shell
#blog 表示运行的container的命名
docker run --mount type=bind,source=$(pwd),target=/srv/jekyll \
-p 4000:4000 --name blog -it jekyll/jekyll \
jekyll serve
```
3. 通过 localhost:4000 访问到这个 Jekyll 的动态生成结果了。

因为我们给这个 Container 赋予了名字 blog，所以之后如果再次需要这个 Container 的话，只需要这样就可以启动：

```shell
docker start -i blog
```
