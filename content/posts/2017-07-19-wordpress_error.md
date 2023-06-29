---
title: nginx代理wordpress无法访问的问题
author: haohongfan
type: post
date: 2017-07-19T09:22:00+00:00
categories:
  - ops
---

先说说我遇到的问题, 这简直就是坑中之王.

最近心血来潮, 购买了一台ECS服务器. 就在服务器上搭建了一个WordPress服务. 由于当时**域名还没有通过备案**. 只好使用IP的方式进行WordPress安装, 安装完成后, 也能使用IP的方式访问网站. 突然今天通知域名通过备案了, 当时那个激动的, 立马给自己的网站换成域名的方式. 然后傻眼了. 

无论我怎么在浏览器中输入我的域名`www.haohongfan.com`, 怎么莫名其妙的自动给我跳转到`www.haohongfan.com:7654`上来. 莫名其妙不是!!!
![](http://images.haohongfan.com/7654.png?imageView2/2/w/800/h/500)

## 下面是我解决这个问题的过程:

### 1. 是否是nginx配置出错呢? 
![](http://images.haohongfan.com/nginx.png?imageView2/2/w/800/h/500)
不管怎么看, 发现这个配置都是正确的.

为了更加验证我的想法, 我就在`/var/www/html/wordpress/`文件夹下放了一个`phpinfo.php`
```
<?php
   echo phpinfo();
```
在浏览器中输入`www.haohongfan.com/phpinfo.php`,然后就出现了这个:
![](http://images.haohongfan.com/php.png?imageView2/2/w/800/h/500)

这更加验证了我的想法: Nginx是没有配置错误, 可以排除这一点.

### 2. 通过nginx日志查看网络请求日志
既然已经验证了nginx配置是正确的. 我打开nginx的日志`access.log`,`error.log`. 当我在浏览器中再输入`haohongfan.com`的时候, 神奇的事情发生了, 我竟然没有看到任何关于这条请求的日志信息.这明显不合理~~~~

这个时候, 我又开始怀疑我的Nginx配置错误了. 然后查了好久资料, 还是没有头绪. 不过过了很久`access.log`出现了一条日志

![](http://images.haohongfan.com/wordpress_error.png?imageView2/2/w/800/h/500)

为什么会这样, 到现在还是没有搞懂. 有知道的同学给我留言.不过有了这个日志, 还是提示我Nginx配置是没有错误.

### 3. 7654这个端口是哪里来的?
竟然第2点无法走通, 那么我的怀疑点就跑到7654上. 这个7654是哪里来的? 这个7654是我当初用IP安装wordpress的时候临时启用的端口. 不过这个时候我已经停掉了这个端口. 使用`sudo lsof -i:7654`完全查不到这个端口启用的任何信息.
 
然后我就在回忆到底在哪里写了这个`7654`端口的配置呢??? 
这个时候我就在nginx和wordpress中
1. 在nginx配置目录中搜索:`ack "7654"`, 没有任何发现
2. 在wordpress目录中搜索:`ack "7654"`, 没有任何发现

这个时候, 我基本已经搞不定了, 打算重装`WordPress`了.当我在浏览器中输入`haohongfan.com/wp-admin/install.php`
![](http://images.haohongfan.com/installed.png?imageView2/2/w/800/h/500)

这个时候, 我突然想到, 为啥能够记录到我已经安装过了呢? 是不是什么信息在数据库中记录了呢?

果然, 在`wp-options`表中发现了为啥访问不了的原因了!!! 这也太坑爹了.

![](http://images.haohongfan.com/db.png?imageView2/2/w/800/h/500)

终于找到问题的所在了, 修改`siteurl`, `home`为`www.haohongfan.com`就可以了