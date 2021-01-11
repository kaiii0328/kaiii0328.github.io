---
layout: post
title: 树莓派3B安装Owncloud问题汇总
description: >
  把在树莓派 3B 上搭建 Owncloud 私有云盘遇到的奇奇怪怪的问题做了一个汇总。新手上路，希望帮助到有需要的人！
image: /assets/img/blog/example-content-ii.jpg
sitemap: false
---

疫情在家闲着无聊想做一些小项目，刚好手上有一块树莓派 3B 的板子，上网搜了搜一些小项目就开始做了，今天这篇介绍如何在树莓派3上搭建一个个人私有云盘，可以存储自己的一些文件，主要参考了[这篇博客]( https://www.lxx1.com/2515/comment-page-1#comment-59883) 如果还没有安装搭建过 LNMP 系统的可以参考[这篇博客]( http://www.lxx1.com/3696)

基本上安装的步骤不太难，参考这两篇就可以了，但我作为一个新手遇到了很多奇奇怪怪的问题，我写这篇文章其实更想要做的是把自己的错误整理一下，希望后面做这个小项目的人可以少走点弯路，本人在网上查遇到的奇奇怪怪的问题的解决方法浪费了不少时间，对这些流逝的时间深感痛心。

1. `PHP intl 模块未安装`

   出现这个问题的同学是因为没有安装 php intl 拓展导致的（如果采用我推荐的第二篇博客链接安装 LNMP 会遇到这个问题），解决方法很简单就是安装这个模块

   ```
   sudo apt-get install php-intl
   ```

2. 创建 owncloud 账号的时候网页提示 `Can’t create or write into the apps-external directory /var/www/html/owncloud/apps-external`

   出现这个问题主要是权限问题，解决方法也很简单

   ```
   $sudo mkdir /var/www/html/owncloud/apps-external #(先手动创建这个文件夹)
   $sudo chmod 755 /var/www/html/owncloud/apps-external #(把这个文件夹的权限改为 755)
   $ sudo chown -hR www-data owncloud #(继续更改权限)
   $ sudo chgrp -hR www-data owncloud
   ```

3. 点击安装完成的时候出现 `SQLSTATE[HY000] [1045] Access denied for user ‘kaiii’@’localhost’ (using password: YES)`

4. 或者点击安装完成的时候 mysql 可能还会出现问题，重启一下就行

5. 或者点击安装完成的时候网页加载很久之后出现 `504 Gateway Timeout`

   我们是使用 Nginx+FastCGI（php-fpm）进行网页加载解析的，504 错误意味着响应时间过长，我们只需要修改一下 Nginx 和 php 的一些文件就行 先打开 /etc/php/7.3/fpm/php.ini 修改

   ```
   max_execution_time= 300
   ```

   然后打开 /etc/php/7.3/fpm/pool.d/www.conf 文件 修改

   ```
   request_terminate_timeout=300
   ```

   接着打开 /etc/nginx/sites-available/default文件  加入 `fastcgi_read_timeout=300`

   ```
   location ~ \.php$ {
                   include snippets/fastcgi-php.conf;
           #
           #       # With php-fpm (or other unix sockets):
                   fastcgi_pass unix:/run/php/php7.3-fpm.sock;
           #       # With php-cgi (or other tcp sockets):          
           #       fastcgi_pass 127.0.0.1:9000;
                   fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                   include fastcgi_params;
                   fastcgi_read_timeout 300;
   }
   ```

   最后重启 Nginx

   ```
   sudo nginx -s reload
   ```

6. Nginx 重启时报错  `[error] open() “/run/nginx/nginx.pid” failed`

   可能原因是每次启动，系统都会自动删除文件，所以解决方法是修改存储位置或者重新开机。该文件里面只存储了nginx主进程的序列号，把数字记住新建一个就行，然后去 nginx.conf 修改 pid 的路径

7. 启动 nginx 时 `nginx: [alert] could not open error log file: open() "/var/log/nginx/error.log" failed (13: Permission denied)`
   `2020/03/18 20:35:10 [warn] 8089#8089: the "user" directive makes sense only if the master process runs with super-user privileges, ignored in /etc/nginx/nginx.conf:1`
   `2020/03/18 20:35:10 [emerg] 8089#8089: open() "/var/log/nginx/access.log" failed (13: Permission denied)`

   一种分析是端口号被占用了，杀死占用端口的进程即可，先使用语句查看端口状态

   ```
   netstat -nltp| grep 80
   ```

   然后可以看 80 端口有没有被占用或者直接看有没有进行中的 nginx 进程

   ```
   ps -ef |grep 80   或者  ps -ef |grep nginx
   ```

   找到占用端口的进程杀死就行

   ```
   sudo kill -9 序列号
   ```

   之后重启 nginx

   另一种分析是因为 80 端口先于 ipv4 绑定，之后又和 ipv6 绑定，我们需要打开 /etc/nginx/sites-available/default文件，将 default 改成 ipv6 即可。一般来说第二种方法更治本一点

   ```
   listen [::]:80 ipv6only=on  default_server;
   ```

   

以上就是本人遇到过的所有问题汇总，希望对你有帮助😉

​        

​        







