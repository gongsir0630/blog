---
title: mysql配置远程连接和修改密码
comments: true
auto_excerpt: true
toc: true
date: 2021-02-01 17:27:31
tags:
    - MySQL
categories:
    - 原创
description: mysql安装完成后修改密码、配置远程连接、新建用户、修改授权等操作.
---

### 场景
1. 基于阿里云轻量应用服务器,系统为CentOS 7;
2. 使用宝塔服务管理服务器;
3. 在服务器安装mysql 5.7, 本地使用dg工具连接管理.

### 安装mysql及配置
- 使用宝塔安装mysql服务,在宝塔商店直接安装即可,可选择主流版本:
![宝塔安装mysql](https://cdn.gongsir.club/blog/image/2021/02/01f4tUcw.png)
- 在宝塔面板 -> 安全中放行3306端口;
- 云服务器供应商处放行3306端口;
- 在宝塔面板 -> 数据库页面修改root初始密码.

### 创建新用户
1. 经过以上配置,可使用ssh远程login到服务器,使用命令连接mysql:
    ```sh
    mysql -uroot -p
    ```
2. 创建新用户:
    ```sql
    CREATE USER 'username'@'host' IDENTIFIED BY 'password';
    ```
    **说明:**
    - username: 用户名
    - host: 指定该用户在哪个主机上可以登陆，如果是本地用户可用localhost，如果想让该用户可以从任意远程主机登陆，可以使用通配符`%`
    - passwd: 该用户的登录密码
3. 删除用户:
    ```sql
    DROP USER 'username'@'host';
    ```

### 授权
对新创建的用户进行授权,授权可以访问、操作的库,表等.
```sql
GRANT privileges ON db.tab TO 'username'@'host'
```
**说明:**
- privileges: 用户的操作权限，如`SELECT`，`INSERT`，`UPDATE`等，如果要授予所的权限则使用`ALL`;
- db: 数据库名称;
- tab: db下的表名,全库全表可用`*.*`表示;

刷新权限:
```sql
flush privileges;
```

### 设置与修改用户密码
1. root用户可修改所有用户的密码,命令:
    ```sql
    SET PASSWORD FOR 'username'@'host' = PASSWORD('newpassword');
    ```
2. 当前用户语法糖:
    ```sql
    SET PASSWORD = PASSWORD("newpassword");
    ```
3. 修改简单密码会报错,此时可以修改密码策略:
    - 查看密码策略:
    ```sql
    show variables like '%validate%';
    ```
    ![密码策略](https://cdn.gongsir.club/blog/image/2021/02/01AWXivL.png)
    - 修改策略:
    ```sql
    set global validate_password_length=6;
    set global validate_password_policy=0;
    ```
4. 修改访问主机:
    ```sql
    use mysql;
    update user set host='host' where user='username';
    ```
    **说明:**
    - host: 允许的主机ip,修改为`%`可允许所有主机访问.



