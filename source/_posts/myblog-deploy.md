---
title: 3️⃣MyBlog-Hexo项目持续集成发布
comments: true
auto_excerpt: true
toc: true
date: 2021-01-22 15:18:35
tags:
    - hexo
    - MyBlog
categories:
    - 原创
description: MyBlog 系列文章第三篇，将hexo项目通过Coding平台持续集成发布到云服务器.
---

很久没更了,这段时间开始做毕设了,所以更新慢了,见谅! 这是 **MyBlog** 系列文章第三篇，上文讲了如何将hexo项目进行远程托管.本文说一下怎么使用Coding平台的持续集成对我们的Hexo项目进行自动部署.

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

## It's Show~Time 🌟

1. 注册Coding团队,进入工作台,创建项目:
![22UIPJbh](https://cdn.gongsir.club/blog/image/2021/01/22UIPJbh.png)

2. 创建DevOps模板项目,填写自定义信息:
![22XkYjq6](https://cdn.gongsir.club/blog/image/2021/01/22XkYjq6.png)

3. 进入项目后,界面如下:
![22Av92HB](https://cdn.gongsir.club/blog/image/2021/01/22Av92HB.png)

4. 代码仓库可以托管我们的Hexo项目,可以直接从GitHub克隆过来,并且提供了定时更新;也可以在本地开发环境直接提交到这里:
![22EcRyhE](https://cdn.gongsir.club/blog/image/2021/01/22EcRyhE.png)

5. 选择`持续集成`,新建构建计划,配置仓库信息,选择hexo项目的托管平台,我的是Coding,选择静态配置Jenkinsfile:
![22RpeXuR](https://cdn.gongsir.club/blog/image/2021/01/22RpeXuR.png)

6. 编写Jenkinsfile:
    ```shell
    pipeline {
    agent any
    stages {
        ## 第一步:从代码仓库拉取代码
        stage('检出') {
        steps {
            ## 从Coding平台拉取代码必须这样写,如果是GitHub平台的代码,可以直接写git clone ~
            checkout([
            $class: 'GitSCM', branches: [[name: env.GIT_BUILD_REF]],
            userRemoteConfigs: [[
                url: env.GIT_REPO_URL,
                credentialsId: env.CREDENTIALS_ID
            ]]
            ])
            ## 项目包含主题子模块,必须拉取子模块,否则主题不会生效
            sh 'git submodule init'
            sh 'git submodule update'
        }
        }
        ## 第二步:构建Hexo项目,将Hexo项目打包成Public静态资源
        stage('构建') {
        steps {
            echo '构建中...'
            sh '''pwd
            node -v
            npm install -g hexo-cli
            npm install'''
            npmAuditInDir(directory: '/', collectResult: true)
            sh '''pwd
                hexo clean
                hexo generate'''
            echo '构建完成.'
        }
        }
        ## 第三步:测试是否生成public/文件夹
        stage('测试') {
        steps {
            sh '''ls -lh public/
                if [ ! -s public/index.html ]
                then
                    exit 1
                fi'''
        }
        }
        ## 第四步:将pulic文件夹下的内容上传到自己的云服务器,这里使用sshpass工具
        stage('部署') {
        steps {
            echo '部署中...'
            ## 将密码等隐私信息配置成环境变量
            sh 'sshpass -p ${SSH_PASS} scp -r public/ ${SSH_USER}@${SSH_HOST}:/www/wwwroot/'
            echo '部署完成'
        }
        }
    }
    }
    ```
    设置隐私数据环境变量:
    ![22gOdKXI](https://cdn.gongsir.club/blog/image/2021/01/22gOdKXI.png)

7. 设置触发规则,设置代码更新到制定分支时自动触发:
![22TkLg3F](https://cdn.gongsir.club/blog/image/2021/01/22TkLg3F.png)

8. 服务器使用`NGINX`部署静态文件夹即可:
![22fpDGmY](https://cdn.gongsir.club/blog/image/2021/01/22fpDGmY.png)

9. 看成果:
![自动部署展示](https://cdn.gongsir.club/blog/gif/myblog-deploy-show.gif)