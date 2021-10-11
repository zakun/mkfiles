## 如何搭建lamp(CentOS7+Apache+MySQL+PHP)环境

在网上搜资料,自己在本地虚拟机上尝试搭建,弄了整整一天一夜,终于弄好了.
网上的资料,虽然很多,但大多都是重复的,拿去试了之后,又很多都不能得到正确的结果.最终找到了适合我的linux环境的搭建方式;在这里贴出来:

这里还是要总结一下我的LAMP环境的搭建步骤。
我先在电脑里装了虚拟机，在虚拟机中测试了数次之后，再在服务器上搭建的。

说说我的环境：
虚拟机是：VMware® Workstation 12.1.1 Pro；
Linux系统用的是：CentOS-7-x86_64-DVD-1511.iso;（阿里云上也是用的CentOS7-64bit）
准备好这两个之后，就开始一步一步搭建我们的LAMP环境了。

### 1. 安装虚拟机

### 2. 安装CentOS7

注意：以下安装，我都是用的root权限。

### 3. 安装Apache
#### 3.1 安装
	yum -y install httpd

#### 3.2 开启apache服务
	systemctl start httpd.service

#### 3.3 设置apache服务开机启动
	systemctl enable httpd.service

#### 3.4 验证apache服务是否安装成功
在本机浏览器中输入虚拟机的ip地址，CentOS7查看ip地址的方式为：
	
	ip addr

（阿里云不需要用这种方式查看，外网ip已经在你主机列表那里给你写出来了的；）
这里是访问不成功的
（阿里云用外网访问，能成功，不需要做以下步骤）
查了资料，说法是，CentOS7用的是Firewall-cmd，CentOS7之前用的是iptables防火墙；要想让外网能访问到apache主目录，就需要做以下的操作：

	firewall-cmd --permanent --zone=public --add-service=http
	firewall-cmd --permanent --zone=public --add-service=https
	firewall-cmd --reload

然后再访问外网ip，如果看到apache默认的页面--有Testing 123...字样，便是成功安装了apache服务了；

### 4. 安装PHP
#### 4.1 安装
	yum -y install php

#### 4.2 重启apache服务

	systemctl restart httpd或者systemctl restart httpd.service

然后，你可以写一个php文件在浏览器中运行一下了;
eg:

	vi /var/www/html/info.php
	i
	<?php phpinfo(); ?>
	Esc
	:wq

然后，在自己电脑浏览器输入 192.168.1.1/info.php
运行，会出现php的一些信息

### 5. 安装MySQL

我这里根据所学的那个教程，也安装了MariaDB

####5.1 安装

	yum -y install mariadb*
#### 5.2 开启MySQL服务

	systemctl start mariadb.service
#### 5.3 设置开机启动MySQL服务

	systemctl enable mariadb.service
#### 5.4 设置root帐户的密码

	mysql_secure_installation

然后会出现一串东西，可以仔细读一下，如果你懒得读，就在提示出来的时候，按Enter就好了，让你设置密码的时候，你就输入你想要的密码就行，然后继续在让你选择y/n是，Enter就好了；当一切结束的时候，你可以输入mysql -uroot -p的方式，验证一下；

### 6. 将PHP和MySQL关联起来

	yum search php，选择你需要的安装：yum -y install php-mysql

### 7. 安装常用的PHP模块
例如，GD库，curl，mbstring,...
#### 7.1 安装：

	yum -y install php-gd php-ldap php-odbc php-pear php-xml php-xmlrpc php-mbstring php-snmp php-soap curl curl-devel

####7.2 重启apache服务
	
	systemctl restart httpd.service

然后，再次在浏览器中运行info.php，你会看到安装的模块的信息；

至此，LAMP环境就搭建好了。

 