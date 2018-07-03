---
title: openx 的部署
date: 2018-06-11
description: ubuntu 16.04 apache php 环境
---

　　先配置环境，主要是搭建 ubuntu 16.04 下 apache 与 php
```shell
# 安装 apache 服务器并设置为本地测试
sudo apt install update
sudo apt install apache2
sudo echo "ServerName 127.0.0.1" >> /etc/apache2/apache2.conf
if apache2ctl configtest do
    sysctemctl restart apache2
    # 此时在浏览器中输入 http://localhost ,应该可以看到 apache 欢迎页面
```
　　安装 `mysql-server`, 和 `php`
```shell
apt install mysql-server
apt install php libapache2-mod-php php-mcrypt php-mysql php-cli php-xml
```
　　接下来需要手动编辑 /etc/apache2/mods-enabled/dir.conf 文件
```
<IfModule mod_dir.c>
    DirectoryIndex index.html index.cgi index.pl index.php index.xhtml index.htm
</IfModule>
```
　　更改为
```
<IfModule mod_dir.c>
    DirectoryIndex index.php index.html index.cgi index.pl index.xhtml index.htm
</IfModule>
```
　　测试 php 的正常运行，创建　`/var/www/html/info.php` 并加入如下内容
```php
<?php phpinfo();?>
```
　　浏览器输入 `localhost/info.php` 应该可以看到 `php` 的相关信息

　　剩下是 openX 系统的部署
　　下载 openx 源码(现更名为 Revive-adserver), 这个只能在 [Revive-adserver](https://www.revive-adserver.com) 官网中中下载，复制的下载连接经测试无效.
　　将源代码拷贝到 `/var/www/html/` 中，浏览器中打开 `localhost/adserver` (假定源代码目录为 adserver),接下来按照指定说明安装即可.
  