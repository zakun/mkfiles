## CentOS 7.x samba 服务器安装


以下以root用户执行
### 1、安装：

	# yum install samba samba-client -y

### 2、设置开机启动：

	systemctl enable smb.service

	ln -s '/usr/lib/systemd/system/smb.service' '/etc/systemd/system/multi-user.target.wants/smb.service'
 

### 3、查看是否设置成功

	#systemctl status smb.service

	smb.service - Samba SMB Daemon
	   Loaded: loaded (/usr/lib/systemd/system/smb.service; enabled)
	   Active: inactive (dead)
 

 
### 4、启动samba服务

	# systemctl start smb.service

### 5、再次查看启动状态

	# systemctl status smb.service

	smb.service - Samba SMB Daemon
	   Loaded: loaded (/usr/lib/systemd/system/smb.service; enabled)
	   Active: active (running) since Sat 2014-12-27 00:15:48 EST; 11s ago
	 Main PID: 2372 (smbd)
	   Status: "smbd: ready to serve connections..."
	   CGroup: /system.slice/smb.service
		   ├─2372 /usr/sbin/smbd
		   └─2373 /usr/sbin/smbd
	 
	Dec 27 00:15:48 localhost.localdomain smbd[2372]: [2014/12/27 00:15:48.521059,  0] ../lib/util/become...dy)
	Dec 27 00:15:48 localhost.localdomain systemd[1]: Started Samba SMB Daemon.
	Dec 27 00:15:48 localhost.localdomain smbd[2373]: STATUS=daemon 'smbd' finished starting up and ready...FUL
	Hint: Some lines were ellipsized, use -l to show in full.
 

### 6、配置配置文件
进入目录：

	# cd /etc/samba
 

备份：

	# cp smb.conf smb.conf.bak

修改smb.conf文件，找到“[homes]”，修改以下设置：
	
	[homes]
	comment = Home Directories
	browseable = no
	writable = yes
	valid users = %S
	valid users = MYDOMAIN\%S

	create mask = 0664
	force create mode = 0664
	directory mask = 0775
	force directory mode = 0775
 

补充：
发现直接从windows拷进去的文件，都会有执行的权限
这里要在smb.conf添加以下
**（*20131203记录，新版的samba一定要在[homes]后面追加，放在smb.conf最后是无效的）**
 

	create mask = 0664
	force create mode = 0664
	directory mask = 0775
	force directory mode = 0775
 

说明：

	默认创建文件是-rw-rw-r-- 664权限
	默认创建目录是rwxrwxr-x 775权限

 
### 7、添加用户

	# smbpasswd -a username

如果出现bash: smbpasswd: command not found，就是没有安装samba-client了

	-------------------------------------------------
	附： smbpasswd命令的常用方法
	smbpasswd -a 增加用户（要增加的用户必须以是系统用户）
	smbpasswd -d 冻结用户，就是这个用户不能在登录了
	smbpasswd -e 恢复用户，解冻用户，让冻结的用户可以在使用
	smbpasswd -n 把用户的密码设置成空. 要在global中写入 null passwords -true
	smbpasswd -x 删除用户
	-----------------------------------------------
 
### 8、selinux设置

	# getsebool -a | grep samba

	# setsebool -P samba_enable_home_dirs on
 

### 9、防火墙

使用新的防火墙firewall添加就可以，比iptables更方便

	# firewall-cmd --list-services

	# firewall-cmd --permanent --add-service=samba

	# firewall-cmd --reload

	# firewall-cmd --list-services
 

由于redhat7开始，iptables被firewalld代替了，所以使用firewalld的方法
关于firewalld的说明，可以看fedora官网介绍
https://fedoraproject.org/wiki/FirewallD/zh-cn
 
### 10、重启samba服务