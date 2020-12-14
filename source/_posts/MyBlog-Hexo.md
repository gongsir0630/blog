---
title: MyBlog-Hexo搭建自己的博客
comments: true
auto_excerpt: true
toc: true
date: 2020-12-14 13:14:08
tags:
categories:
description: MyBlog 系列文章之一,介绍如何使用Hexo搭建博客环境
---

MyBlog 系列文章之一, 这篇文章, 介绍如何使用[Hexo](https://hexo.io/zh-cn/)搭建博客环境, 以及Hexo的基本用法.

### 系列 📒

我的博客([码之泪殇|技术博客](https://blog.gongsir.club))是基于Hexo搭建的, 代码托管在GitHub和Coding, 部署在阿里云服务器, 其中部署是基于Coding提供的持续集成.

#### 发布博客

在本地编写 Markdown 文件, 然后将代码push到托管平台, 即可自动触发持续集成, 将网页部署到云服务器, 全自动~

#### 系列文章目录

1. MyBlog-Hexo搭建自己的博客

### 关于Hexo

Hexo 是一个快速、简洁且高效的博客框架。Hexo 使用 [Markdown](https://daringfireball.net/projects/markdown/)（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。<br>
Hexo官网: [Hexo](https://hexo.io/zh-cn/)<br>

### 安装Git|Node 

本地环境: 我用的是Mac, 以下所有操作都是基于 MacOS 系统, 关于其他平台软件安装, 大同小异.<br>
安装 Hexo 相当简单，只需要先安装下列应用程序即可:
- [Node.js](http://nodejs.org/) (Node.js 版本需不低于 10.13，建议使用 Node.js 12.0 及以上版本)
- [Git](http://git-scm.com/)

MacOS使用 brew|brewcask 安装非常简单, 这里不做赘述, 其他系统也可去某度搜索, 安装很easy的~<br>

### 安装Hexo

1. 打开终端, 创建个人博客目录, eg: blog.gongsir.club
```nash
mkdir blog.gongsir.club
cd blog.gongsir.club
```
终端执行 Hexo 安装命令:
```bash
npm install -g hexo-cli
```

### 搭建博客

