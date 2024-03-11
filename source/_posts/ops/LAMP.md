---
title: LAMP环境搭建
date: 2017-05-19 14:43:13
tags: 
    - Apache
    - Mariadb
    - PHP
    - Archlinux
categories:
  - [运维]
---

>LAMP是指一组通常一起使用来运行动态网站或者服务器的自由软件名称首字母缩写：
>
> &emsp;&emsp;Linux，操作系统
>
> &emsp;&emsp;Apache，网页服务器
>
> &emsp;&emsp;MariaDB或MySQL，数据库管理系统（或者数据库服务器）
>
> &emsp;&emsp;PHP、Perl或Python，脚本语言

——摘自维基百科


### 软件安装

这里我们选择的是 Linux+Apache+MariaDB+PHP。

Arch Linux安装LAMP非常简单，所有的包官方源里都有，所以：

    # pacman -S apache php php-apache mariadb libmariadbclient mariadb-clients 
一条命令就将所有需要的软件直接安装好。



### 启动PHP

#### 配置PHP

&emsp;&emsp;首先，我们来配置PHP。

&emsp;&emsp;php-apache 中包含的 libphp7.so 不支持 mod_mpm_event，仅支持 mod_mpm_prefork。所以需要在 /etc/httpd/conf/httpd.conf 中注释掉:

    #LoadModule mpm_event_module modules/mod_mpm_event.so
&emsp;&emsp;并取消下面行的注释:

    LoadModule mpm_prefork_module modules/mod_mpm_prefork.so
&emsp;&emsp;此外，将这一行放在LoadModule列表中 LoadModule dir_module modules/mod_dir.so 之后的任意地方：

    LoadModule php7_module modules/libphp7.so
&emsp;&emsp;将这一行放到Include列表的末尾：

    Include conf/extra/php7_module.conf
#### 测试PHP

&emsp;&emsp;要测试PHP，在 apache 文档根目录中创建test.php文件，在其中写入：

    <?php phpinfo(); ?>
&emsp;&emsp;然后访问启动apache服务，访问 http://localhost/test.php。若成功启动的话，应如下图所示：

![PHP](https://oem-1251690846.cos.ap-guangzhou.myqcloud.com/img/591edcc26ad0e.png)

### 启动MariaDB

#### 初始化MariaDB

&emsp;&emsp;首先，我们必须必须运行下面这条命令：

    # mysql_install_db --user=mysql --basedir=/usr --datadir=/var/lib/mysql
&emsp;&emsp;启动 mysqld 守护进程，运行安装脚本，然后重新启动守护进程：

    # systemctl start mysqld
    # mysql_secure_installation
    # systemctl restart mysqld
&emsp;&emsp;安装脚本的内容是配置数据库root密码和删除匿名权限还有远程权限之类的，大家各凭所需去选择就行。

#### 配置MariaDB

&emsp;&emsp;运行完脚本后便可以使用设置好的root账号登陆MariaDB：

    # mysql -p -u root
&emsp;&emsp;下面直接改下/etc/php/php.ini，取消这两行的注释：

    extension=pdo_mysql.so
    extension=mysqli.so
&emsp;&emsp;本次搭建便已经完成了,这时再重启下httpd.service 服务便可。

&emsp;&emsp;下面再额外介绍下MariaDB的其他配置。

#### 创建用户

&emsp;&emsp;以下是创建一个密码为’some_pass’的’monty’用户的示例，并赋予 mydb 完全操作权限：
![Mydb](https://oem-1251690846.cos.ap-guangzhou.myqcloud.com/img/591ee3307f03c.png)

#### 禁用远程访问

&emsp;&emsp;MariaDB 服务器默认可从网络访问。如果只有本机需要 MariaDB，可以通过不监听 TCP 端口 3306 来增强安全性。要拒绝远程连接，取消注释 /etc/mysql/my.cnf 中以下这行：

    skip-networking
#### 为数据库使用 UTF-8 编码

&emsp;&emsp;在 /etc/mysql/my.cnf 的 mysqld 下, 添加:

    [mysqld]
    init_connect                = 'SET collation_connection = utf8_general_ci,NAMES utf8'
    collation_server            = utf8_general_ci
    character_set_client        = utf8
    character_set_server        = utf8
    
<br><br>

### 结语
&emsp;&emsp;额😂，还是不滥竽充数了，Arch 的Wiki实在是太全了，一开始是打算参考Wiki写这篇博客，到后面几乎是直接copy了。

&emsp;&emsp;这篇就此结束吧，回头应该还会补上LNMP搭建，另外再推荐下官方Wiki，比我这个要全并且直观多了，Wiki不愧是Archer的一大财富。

[Apache](https://wiki.archlinux.org/index.php/Apache_HTTP_Server)

[PHP](https://wiki.archlinux.org/index.php/PHP)

[MySQL](https://wiki.archlinux.org/index.php/MySQL)
