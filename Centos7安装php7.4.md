## 在CentOS 7系统上安装PHP 7.4

### 添加EPEL和REMI存储库和软件源

```
yum -y install epel-release

yum -y install https://rpms.remirepo.net/enterprise/remi-release-7.rpm

```

### 安装YUM管理工具

```
yum install yum-utils
```


### 查看PHP

```
yum search php73
yum search php74
```

### 安装php74

```
yum install -y php74-php-gd php74-php-pdo php74-php-mbstring php74-php-cli php74-php-fpm php74-php-mysqlnd

```

### 启动

```
systemctl start php74-php-fpm
```
