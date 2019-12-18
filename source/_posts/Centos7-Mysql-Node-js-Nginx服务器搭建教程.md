---
title: Centos7+Mysql+Node.js+Nginx服务器搭建教程
date: 2019-11-25 22:30:23
tags: [前端,服务端]
---

# 过程
## 安装`git`
```
# sudo install git
```

## 安装`node.js`
#### 先到官方网站获取最新版本的源码链接。

https://nodejs.org/zh-cn/download/

#### 进入`/usr/local/src`，通常这个目录都是存放软件的源码安装包的。

```
# cd /usr/local/src
```

#### 用`wget`下载。

```
# wget https://nodejs.org/dist/v8.12.0/node-v8.12.0.tar.gz
```

#### 解压

```
# tar -xzvf node-v8.12.0.tar.gz
```

因为我下载的是`node-v8.12.0.tar.gz`，你要解压自己下载的对应版本。

#### 编译安装
如果我们直接进入目录执行`./configure`是肯定不行的，因为这时候我们没有安装C编译器。（除非已经安装了，可以跳过这步）

#### 编译安装前
1. 安装`gcc`
```
# yum install gcc
```
2. 安装`c++`
```
# yum install gcc-c++
```
3. 安装`gfortran`
```
# yum install gcc-gfortran
```

#### 回到编译安装
现在已经把编译安装前需要的环境都搭好了，可以编译了。
```
# cd node-v8.12.0
# ./configure
# make
```

**整个编译安装的过程有点久......**

然后安装
```
# sudo make install
```

安装完后，就可以查看`node.js`版本了
```
# node -v
```

## 安装`nginx`
#### 先添加nginx仓库
```
# yum install epel-release
```

#### 然后使用`yum`安装
```
# yum install nginx
```

#### 启动服务
```
# service nginx start
```

#### 设置开机启动
```
# systemctl enable nginx
```

#### 修改你的`nginx`配置
```
# vim /etc/nginx/nginx.conf
```

主要是要利用`nginx`给`node.js`做反向代理，一般这样写：
```
server {
    listen 80;
    server_name abc.com;  #绑定的域名
    location /
    {
        proxy_set_header X-Real-IP  $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host   $http_host;
        proxy_set_header X-NginX-Proxy true;
        proxy_set_header Connection "";
        proxy_http_version 1.1;
        proxy_pass http://127.0.0.1:3000;  #对应该的Nodejs程序端口
    }
    access_log /mnt/log/www/abc_access.log; #网站访问日志
}
```

修改后可以输入命令测试是否成功：
```
# nginx -t
```

## 安装`Mysql`
在centOS 7中不能使用`yum -y install mysql mysql-server mysql-devel`安装，这样会默认安装mysql的分支`mariadb`。

`MariaD`B数据库管理系统是MySQL的一个分支，主要由开源社区在维护，采用GPL授权许可的`MariaDB`
是完全兼容MySQL，包括API和命令行，使之能轻松成为MySQL的代替品。

众所周知，Linux系统自带的repo是不会自动更新每个软件的最新版本（基本都是比较靠后的稳定版），所以无法通过yum方式安装MySQL的高级版本。

所以我们需要先安装带有当前可用的mysql5系列社区版资源的rpm包：

```
# rpm -Uvh http://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm
```

查看可用的版本：

```
# yum repolist enabled | grep “mysql.-community.“
```

现在可以直接安装最新版本了：
```
# yum -y install mysql-community-server
```

安装完成后，继续配置。

启动Mysql进程：
```
# systemctl start mysqld
```

添加开机启动：
```
# systemctl enable mysqld
```

然后配置数据库：
```
# mysql_secure_installation
```

然后填写信息：
```
Setting the root password ensures that nobody can log into the MySQL
root user without the proper authorisation.

// 是否设置root的密码
Set root password? [Y/n] y
New password:
Re-enter new password:
Password updated successfully!
Reloading privilege tables..
 ... Success!


By default, a MySQL installation has an anonymous user, allowing anyone
to log into MySQL without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

// 是否移除匿名用户
Remove anonymous users? [Y/n] y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

// 是否禁止远程登录
Disallow root login remotely? [Y/n] y
 ... Success!

By default, MySQL comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

// 是否移除测试数据库
Remove test database and access to it? [Y/n] y
 - Dropping test database...
ERROR 1008 (HY000) at line 1: Can't drop database 'test'; database doesn't exist
 ... Failed!  Not critical, keep moving...
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

// 是否刷新权限
Reload privilege tables now? [Y/n] y
 ... Success!




All done!  If you've completed all of the above steps, your MySQL
installation should now be secure.

Thanks for using MySQL!


Cleaning up...

```

OK了，然后我们就可以登入数据库，去解决远程登录的问题：
```
# mysql
```

允许root用户在任何地方进行远程登录，并具有所有库任何操作权限，具体操作：
```
mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'youpassword' WITH GRANT OPTION;

mysql> FLUSH PRIVILEGES;

mysql> exit
```

然后就可以使用本地的数据库操作软件去链接你的远程服务器了。

在此之前先看看3306端口是否开启了：
```
# netstat -tunlp
```

# 结束
这是我自己配置的全过程，也参考了很多个不同的网页，夹杂了很多不同的方法，由于时间和环境的不同，真实的操作可能有更改。