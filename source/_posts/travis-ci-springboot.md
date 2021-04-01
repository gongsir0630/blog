---
title: 使用 Travis 打造 SpringBoot 应用持续集成和自动部署 | Travis CI 初体验
comments: true
auto_excerpt: true
toc: true
date: 2021-04-02 00:25:55
tags:
  - Travis CI
categories:
  - 原创
  - 后端
description: 最近整理 Github 项目，就想着体验一下使用 Travis CI 来实现项目的 CI，折腾了一天，无奈国内 Travis 文档稀缺，踩了很多坑，所以才来写了这篇文章，希望和同学们相互分享、学习！
---
## 前言
最近一直在写毕设项目（[柚子帮校招指导服务平台](https://github.com/gongsir0630/campus-recruitment-guidance)），顺便整理了一下自己 [GitHub](https://github.com/gongsir0630) 上的项目：

![My GitHub HomePage](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9f3066c64416408db5d7a593fc125062~tplv-k3u1fbpfcp-watermark.image)

大学这几年，还是写了一些能上简历的项目，每一个项目都是 `Pm->Dev->Op` 进行角色扮演，特别是从开发到运维部署这一阶段， 以前每次的版本迭代都要重复下面的流程：

* git add && git commit && git push -> ①代码的迭代
* mvn clean package -> ②项目打包
* scp -r target/xxx.jar user@host:/dir/xxx.jar -> ③上传应用到服务器
* ssh user@host -> ④登录服务器
* sh restart.sh -> ⑤执行部署脚本
* 发现bug -> 修改代码 -> goto ①

如果你也有类似经历，一定会深感**持续集成**的必要性，因为手动部署实在又繁琐又耗时，虽然部署流程基本固定，依然还是容易出错。

## 持续集成

如果你很熟悉**持续集成**，一定会同意这样的观点：「使用它已经成为一种标配」。

如果你还不了解**持续集成**，这里可以[快速入门](https://www.ruanyifeng.com/blog/2015/09/continuous-integration.html)。

市场上持续集成工具众多(Jenkins、Travis CI、Gitlab CI)，也有很多云平台集成了 CI/CID 服务（Coding、阿里云效等）。

之前我体验过 [Coding](https://e.coding.net) 平台的持续集成-构建计划，用于部署我自己的博客，相关文章和教程也发布在自己的博客，感兴趣的同学可以点击[传送门](https://blog.gongsir.club)学习。

最近整理 Github 项目，就想着体验一下使用 Travis CI 来实现项目的 CI，折腾了一天，无奈国内 Travis 文档稀缺，踩了很多坑，所以才来写了这篇文章，希望和同学们相互分享、学习！

## 正文开始

### 需求

要实现 CI，首先得有一个项目，本文使用的项目：[gongsir0630/wx-java-miniapp](https://github.com/gongsir0630/wx-java-miniapp)

项目基于 Spring-Boot 搭建，可以通过 Maven 打包运行。

这里我希望每次提交代码到远程仓库后，travis 会自动编译打包项目，并上传到我的服务器，执行部署脚本，以实现项目自动部署，流程如下：

* 修改代码
* git 提交到远程仓库
* travis 检测仓库变动，执行 `.travis.yml` 中的任务
* maven 编译打包
* 上传 jar 到我的云服务器
* 远程执行部署脚本
* 邮件通知

### GitHub 授权

进入[Travis 官网](https://www.travis-ci.com/)，直接使用 Github 账号登录，登入成功之后，根据页面引导开通 GitHub 仓库访问授权，在 [Dashboard](https://www.travis-ci.com/dashboard) 可以看到所有的授权仓库，可以给常用仓库加星标方便访问：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a4fbd78b8d99414b8ba448eb0cd5973a~tplv-k3u1fbpfcp-watermark.image)

>注意 Travis 目前有两个官网：[travis-ci.com](https://www.travis-ci.com/) 和 [travis-ci.org](https://www.travis-ci.org/). 官方通知后者即将关闭，所以我们使用前者。

### 开启项目监控

在 [Dashboard](https://www.travis-ci.com/dashboard) 页面选择想要开启 travis 功能的仓库，点击`Trigger a build`即可开启对该仓库的监听，travis 一旦监听到仓库变动，就会自动触发构建任务：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/73aebcdc54dc4238aa9923452bd60101~tplv-k3u1fbpfcp-watermark.image)

### travis 配置文件

在项目根目录添加`.travis.yml`文件，这里问题比较多，先上成功案例代码，再慢慢解释：

```yml
language: java
os: linux
dist: xenial
jdk:
- openjdk8

addons:
  ssh_known_hosts:
  - 39.106.230.88

script: mvn clean package -Dmaven.test.skip=true

after_success:
- scp -o stricthostkeychecking=no -r target/wx-java-miniapp-0.0.1-SNAPSHOT.jar travis@yzhelp.top:/www/wwwroot/travis-app/wx-java-miniapp
- ssh travis@yzhelp.top -o stricthostkeychecking=no "sh /www/wwwroot/travis-app/wx-java-miniapp/restart.sh"

branches:
  only:
  - main
  
notifications:
  email:
  - gongsir0630@gmail.com
  
before_install:
- openssl aes-256-cbc -K $encrypted_f217180e22ee_key -iv $encrypted_f217180e22ee_iv
  -in id_rsa.enc -out ~/.ssh/id_rsa -d
- chmod 600 ~/.ssh/id_rsa
- echo -e "Host yzhelp.top\n\tStrictHostKeyChecking no\n" >> ~/.ssh/config

```
配置思路：

1. 监测`main`分支变动，触发构建；
2. 配置 ssh 免密登录自己的云服务器；
3. maven 项目打包；
4. scp 上传 jar 文件到运费服务器（免密操作）；
5. ssh 远程执行`restart.sh`部署脚本。

>**注意** 本文不介绍 travis 文件编写方法，因为我也是第一次学习，建议同学到官网先了解一下相关配置和执行周期。

#### 配置 ssh 免密登录

在 travis 构建任务中不能进行命令交互，我们可以通过 ssh 命令免密登录自己的服务器，这就需要先生成密钥对，在本地生成秘钥对，执行以下命令会在`~/.ssh/`目录下生成`id_rsa`和`id_rsa.pub`两个文件：

```bash
ssh-keygen -t rsa
```
将公钥` id_rsa.pub`使用`ssh-copy-id`添加到服务器的受信列表：

```bash
ssh-copy-id -i id_rsa.pub root@hostname
```
修改本地`~/.ssh/config`文件，追加以下内容：

```js
# 远程主机名称
Host yzhelp.top
# 远程主机域名或者 ip 地址
HostName yzhelp.top
# 认证方式
PreferredAuthentications publickey
# 私钥位置
IdentityFile /Users/gongsir/.ssh/id_rsa_yzhelp
# 用户名
User root
```
>**注意** 如果你本机只有一对秘钥对，可以不用上述配置，如果有多对秘钥对，必须进行上述配置，否则登录时秘钥不会配对。

测试免密登录：

```bash
ssh root@hostname
```
这只是我们自己的电脑可以免密登录到服务器，我们需要配置 travis 的服务器可以免密登录到我们的云服务器，很简单，我们只需要把我们本地的私钥`id_rsa`复制一份给 travis 的服务器就可以了。

但是直接把私钥公布出去太危险了，travis 给我们提供了解决方案：

* 在本地使用`travis-cli`工具对文件进行加密，生成`*.enc`文件
* 将解密串写入 travis 网站的环境变量中
* 执行构建任务时，使用解密串对加密文件进行解密即可得到原文件

#### 文件加密

不多说，官网有详细文档，直接上步骤。

本地安装`travis-cli`工具：

```bash
gem install travis
```
登录 travis，这里坑就比较多了：

| 命令 | 提示信息 | 状态 |
| --- | --- | --- |
| `travis login --auto` | 提示输入 GitHub 的账号密码，登录失败 | ❌ Not Found |
| `travis login --github-token token` |  登录成功，但是后续操作会提示 token 失效 | ✅ 可以登录，但是文件加密会提示 token 无效 |
| `travis login --github-token token --pro` | 登录成功 | ✅ 登陆成功 |

这个环节折腾了大把时间，网上也没找到具体解决方法，这里说下流程：

* `travis-cli` 默认登录的是 [ttps://api.travis-ci.org/](https://api.travis-ci.org/)
* 而我们授权的是 [ttps://api.travis-ci.com/](https://api.travis-ci.com/)
* 因此我们需要使用第三种登录方式，即加上 `--pro` 选项，此时所有请求都发向 [ttps://api.travis-ci.com/](https://api.travis-ci.com/)

我们可以查看本地的 `~/.travis/config` 确认api：

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f0c0d58d19d1446cb7a5df86143289db~tplv-k3u1fbpfcp-watermark.image)

登陆成功之后，对`id_rsa`进行加密，在项目根目录执行以下操作：

```bash
# 此处的--add参数表示自动添加脚本到.travis.yml文件中
# 这里也需要加上 --pro 选项
travis encrypt-file ~/.ssh/id_rsa --add --pro
```
执行完上述命令，会产生的结果：

* 本地新增`id_rsa.enc`，我们需要把这个文件上传到远程仓库供 travis 使用
* `.travis.yml`文件中会增加类似内容：

    ```yml
    before_install:
    - openssl aes-256-cbc -K $encrypted_f217180e22ee_key -iv $encrypted_f217180e22ee_iv
      -in id_rsa.enc -out id_rsa -d
    ```
    * `-in` 表示输入文件，即我们要解密的文件
    * `-out` 表示解密后的文件，这里我们需要手动将路径修改为 `~/.ssh/id_rsa`
    
* travis 仓库中会新增两个环境变量
    
    ![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1b52ad2fd858454a8d3b17792015ac78~tplv-k3u1fbpfcp-watermark.image)
    
经过以上步骤，travis 服务器成功获取到我们的秘钥文件`id_rsa`，为了保证后面能成功执行 ssh 免密登录，我们也需要配置一下权限和主机认证使用的秘钥文件：

```yaml
addons:
  ssh_known_hosts:
  - 这里写自己的服务器的域名或 ip 地址
  
before_install:
- openssl aes-256-cbc -K $encrypted_f217180e22ee_key -iv $encrypted_f217180e22ee_iv
  -in id_rsa.enc -out ~/.ssh/id_rsa -d
- chmod 600 ~/.ssh/id_rsa
- echo -e "Host 自己的服务器域名或 ip 地址\n\tStrictHostKeyChecking no\n" >> ~/.ssh/config
```

#### 项目打包

配置 `script` 即可：

```yaml
script: mvn clean package -Dmaven.test.skip=true
```

#### 上传部署

使用 scp 命令上传 jar 文件到服务器指定目录（预先创建），并执行部署脚本：

```yaml
after_success:
- scp -o stricthostkeychecking=no -r target/wx-java-miniapp-0.0.1-SNAPSHOT.jar travis@yzhelp.top:/www/wwwroot/travis-app/wx-java-miniapp
- ssh travis@yzhelp.top -o stricthostkeychecking=no "sh /www/wwwroot/travis-app/wx-java-miniapp/restart.sh"
```

部署脚本（提供参考）：

```bash
source /etc/profile

jarName=/www/wwwroot/travis-app/wx-java-miniapp/wx-java-miniapp-0.0.1-SNAPSHOT

ps -ef|grep ${jarName}.jar|grep -v grep|awk '{print "kill -9 "$2}'|sh

echo "stop sucess"

nohup java -jar ${jarName}.jar > ${jarName}.log 2>&1 &

echo "${jarName}.jar deploy sucess"
```
>**注意** 这里说一下，部署脚本中包含 `nohup` 命令，一定要在脚本第一行声明 `source /etc/profile`，否则 ssh 获取不到环境变量，导致部署失败。

#### 邮件通知
travis 构建完成之后，无论成功还是失败，都会邮件通知你：

```yaml
notifications:
  email:
  - gongsir0630@gmail.com
```

### 测试

提交 travis 规则到远程仓库，进入 travis 官网即可看到构建日志：

![image.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b710401a479b42e4813cfd53cf0499f9~tplv-k3u1fbpfcp-watermark.image)

点击顶部的徽章还可以复制 md 格式添加到你的 readme 文件中：

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fd6ab71c767344bebfdf9aa0cae6e78a~tplv-k3u1fbpfcp-watermark.image)

## 总结 & 相关链接
以上就是使用 Travis 打造 SpringBoot 应用持续集成和自动部署的全过程，希望能和大家一起分享学习，如果对你有帮助，还请点个赞支持一下，谢谢！

相关链接：

* [示例项目 | wx-java-miniapp](https://github.com/gongsir0630/wx-java-miniapp)
* [Travis 官网文档](https://docs.travis-ci.com/)
* [参考文章：用 Travis CI 打造大前端持续集成和自动化部署](https://juejin.cn/post/6844903808758185998)
* [参考文章：利用Travis CI+GitHub实现持续集成和自动部署](https://juejin.cn/post/6844903957215576078)
