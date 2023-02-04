---
title: 3️⃣MyBlog-Hexo项目持续集成发布
comments: true
auto_excerpt: true
toc: true
sticky: 1
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

我的博客([程序员Kyle✨|技术博客](https://blog.gongsir.club))是基于Hexo搭建的, 代码托管在GitHub和Coding, 部署在阿里云服务器, 其中部署是基于Coding提供的持续集成.

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
    ```sh
    pipeline {
    agent any
    stages {
        stage('检出') {
        steps {
            checkout([
            $class: 'GitSCM', branches: [[name: env.GIT_BUILD_REF]],
            userRemoteConfigs: [[
                url: env.GIT_REPO_URL,
                credentialsId: env.CREDENTIALS_ID
            ]]
            ])
            sh 'ls -la'
            sh 'git submodule init'
            sh 'git submodule update'
        }
        }
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
            sh 'tar -zcf /tmp/tmp.tar.gz public/'
            echo '构建完成.'
        }
        }
        stage('测试') {
        steps {
            sh '''ls -lh public/
                    if [ ! -s public/index.html ]
                    then
                    exit 1
                    fi'''
        }
        }
        stage('部署') {
        steps {
            echo '部署中...'
            script {
            def remote = [:]
            remote.name = 'aliyun-server'
            remote.allowAnyHosts = true
            remote.host = 'gongsir.club'
            remote.port = 22
            remote.user = 'root'

            // 把「CODING 凭据管理」中的「凭据 ID」填入 credentialsId，而 username 和 password 无需修改
            withCredentials([usernamePassword(credentialsId: "1f981c07-2c51-4a64-a93b-be09ba76a406", usernameVariable: 'username', passwordVariable: 'password')]) {
                remote.user = username
                remote.password = password

                // SSH 上传文件到远端服务器
                sshPut remote: remote, from: '/tmp/tmp.tar.gz', into: '/tmp/'
                // 解压缩
                sshCommand remote: remote, command: "tar -zxf /tmp/tmp.tar.gz -C /tmp/"
                sshCommand remote: remote, sudo: true, command: "mkdir -p /www/wwwroot/blog.gongsir.club"
                sshCommand remote: remote, sudo: true, command: "cp -R /tmp/public/* /www/wwwroot/blog.gongsir.club/"
                // 重启 nginx
                sshCommand remote: remote, sudo: true, command: "nginx -s reload"
            }
            }
            echo '部署完成'
        }
        }
    }
    }
    ```
    「CODING 凭据管理」中的「凭据 ID」:
    ![凭据ID设置](https://cdn.gongsir.club/blog/image/2021/03/19fDIDPk.png)

7. 设置触发规则,设置代码更新到制定分支时自动触发:
![22TkLg3F](https://cdn.gongsir.club/blog/image/2021/01/22TkLg3F.png)

8. 服务器使用`NGINX`部署静态文件夹即可:
![22fpDGmY](https://cdn.gongsir.club/blog/image/2021/01/22fpDGmY.png)

9. 看成果:
![自动部署展示](https://cdn.gongsir.club/blog/gif/myblog-deploy-show.gif)

## 参考文档
- [快速开始持续集成](https://help.coding.net/docs/ci/start.html)
- [自动部署到Linux服务器](https://help.coding.net/docs/ci/deploy/ssh.html)