---
layout: post
title: "github pages 绑定域名"
description: ""
category: dev
tags: [github, domain]
---
{% include JB/setup %}

## github page 介绍
github page 给开发者提供介绍开源项目的另一种途径。之所以说是另一种途径，是因为 github 本身有 wiki 功能，在 wiki 中足可以介绍项目的使用方法、实现说明等。但 wiki 也有自己的局限性，它是附属于具体项目的，入口还是比较繁琐。因此 github 提供了 page 功能，提供了更为快捷的入口，当然像我这种无名鼠辈可以把它当做记录 blog 的地方。

## 绑定域名的方法
github page 可以绑定自己的域名，则入口就更为简单了。网上有很多设置方法，这里也只是记录一下，并没有两样：

### yourdomain.com 类型

#### github 上的设置
在 yourgitname.github.io 根目录下生成名为 CNAME 的文件，内容就是你的域名：

    yourdomain.com

#### 在 DNS 上的设置
推荐使用 dnspod，免费好用
新建 A 类型的记录

    yourdomain.com => 204.232.175.78

### www.yourdomain.com 类型

#### github 上的设置
在 yourgitname.github.io 根目录下生成名为 CNAME 的文件，内容就是你的域名：

    www.yourdomain.com

#### 在 DNS 上的设置
新建 A 类型的记录

    www.yourdomain.com => yourgitname.github.io
    
新建 CNAME 类型的记录

    yourgitname.github.io => 204.232.175.78
    
参考官方文档: [Setting up a custom domain with Pages
](https://help.github.com/articles/setting-up-a-custom-domain-with-pages)

