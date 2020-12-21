---
title: 2️⃣MyBlog-多平台代码托管
comments: true
auto_excerpt: true
toc: true
date: 2020-12-20 18:14:40
tags:
    - hexo
    - MyBlog
categories:
    - 原创
description: MyBlog 系列文章第二篇，如何将hexo项目托管到多个代码仓库(GitHub、Coding)
---

**MyBlog** 系列文章第二篇，上篇文章说了如何使用[Hexo](https://hexo.io/zh-cn/)搭建自己的专属博客，但自己有两台Mac，想在不同设备都可以发布博客，为了方便进行博客迁移，所以采用代码托管平台来存储我的hexo项目文件，这里采用国外的[GitHub](https://github.com)和国内的[Coding](https://e.coding.net).

<!-- more -->

## 系列 📒

我的博客([码之泪殇|技术博客](https://blog.gongsir.club))是基于Hexo搭建的, 代码托管在GitHub和Coding, 部署在阿里云服务器, 其中部署是基于Coding提供的持续集成.

### 发布博客

在本地编写 Markdown 文件, 然后将代码push到托管平台, 即可自动触发持续集成, 将网页部署到云服务器, 全自动~

### 系列文章目录

1. [MyBlog-Hexo搭建自己的博客](/2020/12/14/MyBlog-Hexo/)
2. [MyBlog-多平台代码托管](/2020/12/20/myblog-code.html)

## 项目托管

博客迁移时，我们需要保留以下文件：
- scaffolds/    --> 模板
- source/   --> 资源文件和md
- themes/   --> 主题文件
- _config.yml   --> 配置文件
- package.json  --> 依赖文件
- package-lock.json --> 插件版本
- **注意：`public/`目录内容可以通过hexo生成，所以不需要backup**

所以我们只需要将上述文件托管在GitHub、Coding等平台就可以，以后使用的时候，只需要本地安装node环境，然后拉取分支，执行：
1. 安装hexo-cli：
    ```bash
    npm install -g hexo-cli
    ```
2. 安装npm依赖：
    ```bash
    npm install
    ```
3. 解析生成静态文件：
    ```bash
    hexo g
    ```

### Github托管

1. 登录GitHub，新建repository，复制仓库地址

### Coding托管