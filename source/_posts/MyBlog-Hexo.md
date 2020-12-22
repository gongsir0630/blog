---
title: 1️⃣MyBlog-Hexo搭建自己的博客
comments: true
auto_excerpt: true
toc: true
date: 2020-12-14 13:14:08
sticky: 1
tags:
    - hexo
    - hexo-theme-blank
    - MyBlog
categories: 
    - 原创
description: MyBlog 系列文章之一,介绍如何使用Hexo搭建博客环境
---

**MyBlog** 系列文章之一, 这篇文章, 介绍如何使用[Hexo](https://hexo.io/zh-cn/)搭建博客环境, 以及Hexo的基本用法.

<!-- more -->

## 系列 📒

我的博客([码之泪殇|技术博客](https://blog.gongsir.club))是基于Hexo搭建的, 代码托管在GitHub和Coding, 部署在阿里云服务器, 其中部署是基于Coding提供的持续集成.

### 发布博客

在本地编写 Markdown 文件, 然后将代码push到托管平台, 即可自动触发持续集成, 将网页部署到云服务器, 全自动~

### 系列文章目录

1. [MyBlog-Hexo搭建自己的博客](/2020/12/14/MyBlog-Hexo/)
2. [MyBlog-多平台代码托管](/2020/12/20/myblog-code.html)
<hr/>

## 关于Hexo

Hexo 是一个快速、简洁且高效的博客框架。Hexo 使用 [Markdown](https://daringfireball.net/projects/markdown/)（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。
Hexo官网: [Hexo](https://hexo.io/zh-cn/)<br>

## 安装Git | Node

1. 本地环境: 我用的是Mac, 以下所有操作都是基于 MacOS 系统, 关于其他平台软件安装, 大同小异.<br>
2. 安装 Hexo 相当简单，只需要先安装下列应用程序即可:
    - [Node.js](http://nodejs.org/) (Node.js 版本需不低于 10.13，建议使用 Node.js 12.0 及以上版本)
    - [Git](http://git-scm.com/)
3. MacOS使用 brew|brewcask 安装非常简单, 这里不做赘述, 其他系统也可去某度搜索, 安装很easy的~<br>

## 安装Hexo

1. 打开终端, 创建个人博客存放目录(eg: blog.gongsir.club): 
    ```Bash
    mkdir blog.gongsir.club
    ```
    ```Bash
    cd blog.gongsir.club
    ```
2. 终端执行 Hexo 安装命令:
    ```Bash
    npm install -g hexo-cli
    ```
3. 终端输入以下命令查看hexo信息，成功即可：
    ```Bash
    hexo -v
    ```

## 开始 ✨

### 基础环境

接下来就可以开始使用hexo搭建一个 “Hello World” 页面了。
1. 使用hexo的hello-world初始化你的博客（此时会自动使用Git克隆项目）：
    ```bash
    hexo init
    ```
2. 安装hexo所需依赖：
    ```bash
    npm install
    ```
3. 启动，预览：[http://localhost:4000](http://localhost:4000)
    ```bash
    hexo s
    ```

成功看到Hexo的Hello——World页面即表示Hexo建站成功，但是似乎UI不太喜欢，想改下主题，所以后续来了！！！

### 主题配置

1. 我尝试过很多主题，在hexo的主题官网有很多themes，但是感觉都不是我想要的，最后在github上发现了一款不错的主题——[hexo-theme-blank](https://github.com/a2396837/hexo-theme-blank)
   - GitHub: https://github.com/a2396837/hexo-theme-blank
   - Theme配置文档（很详细）： https://dmx.pub/

2. 主题安装这里需要注意一下，假设我们希望将整个博客文件放在Github进行托管，而我们安装主题时通常是直接在themes文件夹下克隆主题仓库：
    ```bash
    git clone https://github.com/a2396837/hexo-theme-blank.git blank
    ```
    当我们修改了我们的博客内容，向github提交代码的时候，就会出现冲突，因为我们的themes/blank包含在博客目录下（即git工程包含了git工程），解决这个问题最简单的方法就是将themes/blank下的`.git`文件夹删除，此时themes/blank就是一个普通文件夹，博客项目就可以直接push到github。

    但是我们考虑到主题在今后可能会有升级的情况，主题目录使用`git pull`即可完成升级，因此就不能删除主题目录的`.git`，建议使用**Git子模块**对整个项目进行代码托管：
    在博客根目录执行：
    ```bash
    ## 下载主题
    git clone https://github.com/a2396837/hexo-theme-blank.git themes/blank
    ## 将主题文件夹添加为git子模块
    git submodule add https://github.com/a2396837/hexo-theme-blank.git themes/blank
    ```
    之后我们的博客根目录就会自动生成一个`.gitmodules`的文件，记录了子模块的信息，接下来我们正常操作父项目就可以了，想学习更多关于`git子模块`的知识，可以去官网学学~

3. 为了主题在今后能够进行平滑升级，我们不要在主题文件夹直接修改配置，解决方法是将主题中的`._config.yml`拷贝到博客根目录的`source/_data`（需要自己新建`_data`）目录下，并重命名为`blank.yml`，然后在`blank.yml`进行主题配置:
    ```bash
    mkdir source/_data
    cp -p themes/blank/_config.yml source/_data/blank.yml
    ```
### 开始创作

1. hexo可以直接解析md文档，我们只需新建一个md，然后编辑md即可：
   ```bash
   hexo new "blog-title"
   ```
2. 以上会在`source/_post`目录下生成一个名叫“blog-title”的md文件，使用`vscode`编辑：
   ![vscode编写md](https://cdn.gongsir.club/blog/img/2020-12-17-1.png)
3. 生成静态网页文件：使用hexo的命令将md和主题样式解析成html文件，并保存在`public/`下：
   ```bash
   hexo g
   ```
4. 运行方式：
   1. 因为生成的是静态html，因此可以直接打开`public/`，双击`index.html`运行；
   2. 使用`hexo s | hexo s --debug (debug模式)`.

### 博客部署

将生成的`public/`目录下的静态文件（html、csss、js等）直接部署到你的服务器（可以使用scp、ftp等服务），再配置nginx即可访问.<br>
但是这样很麻烦，每次写一篇，都需要以下步骤：
1. `hexo clean`: 清除`public/`缓存;
2. `hexo g`: 重新生成静态文件；
3. 手动部署到你的服务器.

其实我们不用这么麻烦，当我们写完一篇博客之后，直接将代码push到托管平台，之后就可以自动解析、自动部署.如何实现? 看后续... ...

## 参考文档

- Hexo官方文档：[这里可以从零开始学习hexo](https://hexo.io/zh-cn/docs/)
- hexo-theme-blank：[这里可以学习hexo-theme-blank的详细配置](https://dmx.pub/)
    
<hr>