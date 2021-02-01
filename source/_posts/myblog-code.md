---
title: 2️⃣MyBlog-多平台代码托管
comments: true
auto_excerpt: true
toc: true
sticky: 2
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
3. [MyBlog-Hexo项目持续集成发布](/2021/01/22/myblog-deploy.html)
<hr>

## 项目托管

博客迁移时，我们需要保留以下文件：
- scaffolds/    --> 模板
- source/   --> 资源文件和md
- themes/   --> 主题文件
- _config.yml   --> 配置文件
- package.json  --> 依赖文件
- package-lock.json --> 插件版本
- **🌟注意：`public/`目录内容可以通过hexo生成，所以不需要backup**

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

1. 登录GitHub，新建repository，复制仓库地址`@your_url`；
2. 本地代码文件夹`(blog.gongsir.club)`初始化，设置远程仓库地址：
   ```bash
   git remote add origin @your_url
   ```
   🌟注意：这里需要将`themes/blank`加为子模块（第一篇文章中提过，方便主题升级），此时项目根目录会自动生成一个`.gitmodules`文件，也需要上传到远程仓库：
   ```bash
   git submodule add https://github.com/a2396837/hexo-theme-blank.git themes/blank
   ```
3. 将本地代码push上去，文件结构如下：
   ![GitHub代码托管](https://cdn.gongsir.club/blog/img/20201221224358.png)
   

### Coding托管

一开始我代码直接托管在GitHub上的，博客部署是基于`Git Actions`生成`public/`内容，将`public/`上传到另一个分支`page`，然后在自己的服务器拉取`page`分支以部署网站.

这种方式有一些小问题：
- GitHub国内访问速度感人
- `.config.yml`需要配置[hexo-deploy]()
- 每次博客更新都需要手动部署
- 综上，个人觉得还是比较麻烦

所以后来采用了国内Coding进行托管，并基于Coding的持续集成对博客进行自动化部署.

1. 之前已经将代码托管在GitHub的，可以直接在Coding上克隆GitHub的仓库：
   ![Coding](https://cdn.gongsir.club/blog/img/20201222120853.png)
2. 修改本地git的config，将coding的url加入git配置：
   ```bash
   git remote set-url -add origin https://e.coding.net/xxx.git
   ```
   查看remote信息，此时应该包含两个push仓库，以后oush代码的时候，会同时push到github和coding：
   ![remote](https://cdn.gongsir.club/blog/img/20201222121345.png)

## ssh-key管理

1. 为了方便每次push代码时避免输入密码，需要在自己本地配置ssh-key，github和coding官方都有教程：
   - [generating SSH keys](https://docs.github.com/articles/generating-an-ssh-key/)
   - [配置SSH公钥](https://help.coding.net/docs/project-settings/features/ssh.html)

2. 本地多平台公钥管理，当本地存在多个平台的public-key时，需要添加config配置文件，否则某些平台公钥不会生效：
  - 在`.ssh/`目录下新建config文件，配置不同平台的ssh公钥：
    ![.ssh目录](https://cdn.gongsir.club/blog/img/20201222122841.png)
    config文件内容如下：
    ```bash
    # github
    Host github.com
    HostName github.com
    PreferredAuthentications publickey
    # 这里设置github的公钥文件路径
    IdentityFile C:\Users\gongsir\.ssh\id_ed25519

    # coding
    Host e.coding.net
    HostName e.coding.net
    PreferredAuthentications publickey
    # 这里设置coding的公钥文件路径
    IdentityFile C:\Users\gongsir\.ssh\id_rsa
    ```
  - 使用`ssh -T git@github.com`、`ssh -T git@e.coding.net`验证：
    ![ssh验证](https://cdn.gongsir.club/blog/img/20201222123241.png)
    