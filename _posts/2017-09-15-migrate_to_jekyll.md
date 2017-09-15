---
title: 通过 Jekyll 迁移 Blog 
author: bluezd
layout: post
permalink: /archives/781
categories:
  - VPS
tags:
  - VPS
---

#### Why ? 

为什么不用 wordpress 了呢？先说说痛点:

  1. wordpress 安装较为繁琐 (LAMP) 
  2. 需要定期维护，比如备份数据库
  3. 迁移过程很繁琐，环境需要重新安装配置。

#### Jekyll 优势 

[jekyll](https://jekyllrb.com/) 是一个基于 Ruby 开发的简单的免费的Blog生成工具, 可以通过它把 wordpress blog 转换成静态网页，不需要数据库。

可以通过如下方式进行 migrate:

  * [官方脚本](http://import.jekyllrb.com/docs/wordpress/)
  * 或者在 wordpress 上安装插件 [jekyll-exporter](https://wordpress.org/plugins/jekyll-exporter/) 导出成一个 jekyll-export.zip 的文件
  * 评论的话可以通过 [Disqus](https://disqus.com/) 迁移, 像我这样没啥评论的就不用费这劲了。


好处有哪些呢？

  * 用 markdown 写 blog 
  * 所有 posts 可以保存在 git respository 中, 发布新的文章很方便
  * 不需要定期维护备份

#### 部署 

部署在哪呢 ?

  * [github pages](https://pages.github.com/)  
    可以免费部署在这里，通过 git 发布维护 blog, 很是方便
  * VPS  
    当然也可以部署在 VPS上，不过你要事先安装好环境，比如 `ruby`, `jekyll` 等等。
    那如果以后迁移到其他环境岂不是又要配置环境吗 ? 是的，不过我把 jekyll 运行环境做成了一个 Docker image, 通过如下方式运行你的 blog:
      - `docker pull bluezd/jekyll-build` 
      - 把你的博客 `git clone` 到某个路径下 `/Path/To`
      - `docker run --name blog -p 80:4000 -v /Path/To:/jekyll-home/blog bluezd/jekyll-build:latest`

    这样你的 blog 就能跑起来了，是不是很简单，你不需要在配置环境上花费时间，把省下的时间放在写 blog 上。
