---
layout: post
title: ubuntu-php7+apache2
excerpt: how to install php7+apache2
---

```
Ubuntu 14.04 安装并配置LAMP

标签：Ubuntu Linux Apache2.4 Mysql5.7 php7.0
开始之前

    系统版本 Ubuntu 14.04(LTS)

    本文是用 apt 来安装 LAMP 环境，并非使用源码编译

    更新你的系统资源

    sudo apt-get update && sudo apt-get upgrade

    现在开始！

Apache 2.4

    通过apt安装Apache

    sudo apt-get install apache2

    编辑apache主配置文件 /etc/apache2/apache2.conf ，修改 KeepAlive 设置

    KeepAlive Off

    Apache默认的 multi-processing 模块( MPM ) 是一个event 模块, 但是 php默认是使用 prefork 模块

    禁用event 模块，启用 prefork 模块

    sudo a2dismod mpm_event
    sudo a2enmod mpm_prefork

    重启Apache

    sudo service apache2 restart

    如果在重启Apache时，看见关于ServerName的报错，可以做如下修改

        编辑apache主配置文件 /etc/apache2/apache2.conf

        添加一行 ServerName localhost

        然后执行 sudo service apache2 restart

Mysql 5.7

    目前默认的源是找不到5.7版本的。如果想通过apt来安装mysql5.7，则需要添加源。

    目前网上给出的大部分答案是这样的

    $ sudo apt-get install software-properties-common
    $ sudo add-apt-repository -y ppa:ondrej/mysql-5.7
    $ sudo apt-get update
    $ sudo apt-get install mysql-server
    # 这样apt是找不到5.7版本的。

    通过Google，找到了正确的安装步骤

    wget http://dev.mysql.com/get/mysql-apt-config_0.6.0-1_all.deb
    sudo dpkg -i mysql-apt-config_0.6.0-1_all.deb
    sudo apt-get update
    sudo apt-get install mysql-server-5.7
    # 这样才能通过apt来安装mysql5.7
    # 在安装过程中，会要求输入root的密码。

    安装完成后，执行 mysql_secure_installation ，根据提示完成安全设置

PHP7.0

    首先查看下当前源中是否含有php7.0

    sudo apt-cache search php7.0

    如果没有，则添加源，并更新，然后安装

    sudo apt-get install software-properties-common
    sudo add-apt-repository ppa:ondrej/php
    sudo apt-get update

    如果有则直接安装

    sudo apt-get install php7.0

整合LAMP

    整合php和mysql

    sudo apt-get install php7.0-mysql

    整合php和Apache

    sudo apt-get install libapache2-mod-php7.0
    # ...
    sudo service apache2 restart

验证环境

    Apache默认的网站根目录位于 /var/www/html/ ,进入这个目录，并创建 info.php

    <?php 
    phpinfo();
    ?>

    在浏览器中输入 http://localhost/info.php 。

排错

    如果 http://localhost/info.php 页面空白，请尝试 Ctrl+F5 强制刷新页面。

    如果依然空白，说明php和apache之间还需要一些配置

    编辑 /etc/apache2/apache2.conf

    <FilesMatch \.php$>
    SetHandler application/x-httpd-php
    </FilesMatch>

    重启Apache

    sudo service apache2 restart

    刷新 http://localhost/info.php 。此时应该可以看见phpinfo中的内容了。

```
