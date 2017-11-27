
## 环境介绍

- 系统版本：CentOS 7
- Nginx版本:1.12.2
- Php版本：7.1.11
- Mysql版本：5.6

## 准备工作
- 关闭selinux，开启selinux会引起一连串问题，甚至zabbix的discovery功能也不能正常使用

`sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config`

 确认是否修改成功

 `grep SELINUX /etc/selinux/config`

然后重启系统即可 

## 安装Nginx 

- 下载nginx `wget http://nginx.org/packages/centos/7/x86_64/RPMS/nginx-1.12.2-1.el7_4.ngx.x86_64.rpm`
- 安装 nginx `rpm -ivh  nginx-1.12.2-1.el7_4.ngx.x86_64.rpm`
- 设置nginx开机自启动  `systemctl enable nginx`
- 启动nginx   `systemctl start nginx`
- 查看nginx 启动状态  `systemctl status nginx`
- 不关闭nginx，修改配置文件生效方法 `nginx -s reload`
- 防火墙开放80端口  `firewall-cmd   --add-port=80/tcp   --permanent`
- 重新加载防火墙配置 `firewall-cmd --reload`
- 查看防火墙当前开放的端口 `firewall-cmd --list-ports`

## 安装Mysql

- Centos7自带的mysql是mariadb ,我们可以通过如下命令查看 `yum search mysql|tac`
- 开始安装mariadb `yum -y install mariadb mariadb-server`
- 设置开机自启动mysql  `systemctl enable mariadb.service`
- 启动mysql `systemctl start mariadb.service`
- 初始化mysql数据库，并配置root用户密码 `mysql_secure_installation`

## 安装PHP

- 下载php7 `wget http://cn2.php.net/distributions/php-7.1.11.tar.gz`
- 解压并进入php目录 `tar zxvf php-7.1.11.tar.gz && cd ./php-7.1.11	`
- 安装扩展包更新包	`yum  install epel-release`
- 安装php依赖包 

```
yum install -y libxml2 libxml2-devel openssl openssl-devel bzip2 bzip2-devel libcurl libcurl-devel libjpeg libjpeg-devel libpng libpng-devel freetype freetype-devel gmp gmp-devel libmcrypt libmcrypt-devel readline readline-devel libxslt libxslt-devel 

yum install -y libmcrypt libmcrypt-devel gcc
```

- 编译配置php

`./configure --prefix=/usr/local/php --with-config-file-path=/etc --enable-fpm --with-fpm-user=nginx  --with-fpm-group=nginx --enable-inline-optimization --disable-debug --disable-rpath --enable-shared  --enable-soap --with-libxml-dir --with-xmlrpc --with-openssl --with-mcrypt --with-mhash --with-pcre-regex --with-sqlite3 --with-zlib --enable-bcmath  --with-iconv --with-bz2 --enable-calendar --with-curl --with-cdb --enable-dom --enable-exif --enable-fileinfo --enable-filter --with-pcre-dir --enable-ftp --with-gd --with-openssl-dir  --with-jpeg-dir --with-png-dir --with-zlib-dir --with-freetype-dir --enable-gd-native-ttf --enable-gd-jis-conv --with-gettext --with-gmp --with-mhash --enable-json --enable-mbstring --enable-mbregex --enable-mbregex-backtrack --with-libmbfl --with-onig --enable-pdo --with-mysqli=mysqlnd --with-pdo-mysql=mysqlnd --with-zlib-dir --with-pdo-sqlite --with-readline --enable-session --enable-shmop --enable-simplexml --enable-sockets  --enable-sysvmsg --enable-sysvsem --enable-sysvshm --enable-wddx --with-libxml-dir --with-xsl --enable-zip --enable-mysqlnd-compression-support --with-pear --enable-opcache
`
- 编译与安装,需要一段时间慢慢等候  `make && make install`
- 添加 PHP 命令到环境变量 `vi /etc/profile`,在末尾加入 

```
PATH=$PATH:/usr/local/php/bin

export PATH
```
- 要使改动立即生效执行 `source /etc/profile`
- 查看环境变量 `echo $PATH`
- 查看php版本 `php -v`
- 配置php-fpm 

```
cp ./php.ini-production /etc/php.ini  #从php源代码中拷贝php.ini-production 到并命名为 /etc/php.ini
cp /usr/local/php/etc/php-fpm.conf.default /usr/local/php/etc/php-fpm.conf
cp /usr/local/php/etc/php-fpm.d/www.conf.default /usr/local/php/etc/php-fpm.d/www.conf
cp sapi/fpm/init.d.php-fpm /etc/init.d/php-fpm
chmod +x /etc/init.d/php-fpm    
```
- 修改/etc/php.ini 文件 `vi /etc/php.ini`,找到配置项，并修改成如下内容

```
max_execution_time = 300
max_input_time = 300
memory_limit = 128M
post_max_size = 16M
date.timezone = Asia/Shanghai
```

- 启动php-fpm ` /etc/init.d/php-fpm start`

- 配置nginx,绑定服务器 `vi /etc/nginx/conf.d/default.conf`

```
server{
	listen 80;
	server_name  127.0.0.1;
	root /data/www;  # 该项要修改为你准备存放相关网页的路径
	location / {
		index  index.php index.html index.htm;
		#如果请求既不是一个文件，也不是一个目录，则执行一下重写规则
		if (!-e $request_filename)
		{
			#地址作为将参数rewrite到index.php上。
			rewrite ^/(.*)$ /index.php/$1;
			#若是子目录则使用下面这句，将subdir改成目录名称即可。
			#rewrite ^/subdir/(.*)$ /subdir/index.php/$1;
		}
	}
	#proxy the php scripts to php-fpm
	location ~ \.php {
		include fastcgi_params;
		##pathinfo支持start
		#定义变量 $path_info ，用于存放pathinfo信息
		set $path_info "";
		#定义变量 $real_script_name，用于存放真实地址
		set $real_script_name $fastcgi_script_name;
		#如果地址与引号内的正则表达式匹配
		if ($fastcgi_script_name ~ "^(.+?\.php)(/.+)$") {
			#将文件地址赋值给变量 $real_script_name
			set $real_script_name $1;
			#将文件地址后的参数赋值给变量 $path_info
			set $path_info $2;
		}
		#配置fastcgi的一些参数
		fastcgi_param SCRIPT_FILENAME $document_root$real_script_name;
		fastcgi_param SCRIPT_NAME $real_script_name;
		fastcgi_param PATH_INFO $path_info;
		###pathinfo支持end
	    fastcgi_intercept_errors on;
		fastcgi_pass   127.0.0.1:9000;
	}
		
    location ^~ /data/runtime {
        return 404;
    }
		
    location ^~ /application {
        return 404;
    }
    
    location ^~ /simplewind {
        return 404;
    }
}
```
- 重启nginx `nginx -s reload`
- 创建测试php文件 `vi /data/www/info.php`

```
<?php

     phpinfo();
```
- 访问 http://127.0.0.1/info.php 

## 安装zabbix-server

- 下载安装zabbix的YUM源 `rpm -ivh http://repo.zabbix.com/zabbix/3.4/rhel/7/x86_64/zabbix-release-3.4-2.el7.noarch.rpm`
- 安装zabbix `yum -y install zabbix-server-mysql zabbix-agent`
- 在mysql中创建zbbix帐号

```
create database zabbix character set utf8 collate utf8_bin;
grant all privileges on zabbix.* to 'zabbix'@'%' identified by 'zabbixpass';
flush privileges;
```

- 导入zabbix表结构和初始化到数据库

```
cd /usr/share/doc/zabbix-server-mysql-3.41/
zcat create.sql.gz | mysql -u root -p zabbix
```

- 配置Zabbix服务器端 `vi /etc/zabbix/zabbix_server.conf`,找到配置项，并更改配置

```
DBHost=localhost
DBName=zabbix
DBUser=zabbix
DBPassword=zabbixpass
```
- 切换文件夹所有人

```
chown -R zabbix:zabbix /etc/zabbix
chown -R zabbix:zabbix /usr/lib/zabbix
```
- 设置开机启动zabbixserver `systemctl enable zabbix-server`
- 启动zabbix server `systemctl start zabbix-server`
- 防火墙开放10051端口，zabbix agent会用到这个端口  `firewall-cmd   --add-port=10051/tcp   --permanent`
- 重新加载防火墙配置 `firewall-cmd --reload`
- 查看防火墙当前开放的端口 `firewall-cmd --list-ports`

## 安装zabbix-web

- 下载 zabbix源代码 `wget -O zabbix-3.4.4.tar.gz  http://sourceforge.net/projects/zabbix/files/ZABBIX%20Latest%20Stable/3.4.4/zabbix-3.4.4.tar.gz/download`
- 解压并进入zabbix目录 ` tar zxvf zabbix-3.4.4.tar.gz `
- 拷贝`zabbix-3.4.4/frontends/php` 文件夹中所有内容到 `/data/www/zabbix`文件夹中  `cp -rf ./zabbix-3.4.4/frontends/php/ /data/www/zabbix`
- 新建配置文件 `mv /data/www/zabbix/conf/zabbix.conf.php.example /data/www/zabbix/conf/zabbix.conf.php` 
- 修改配置文件 `vi /data/www/zabbix/conf/zabbix.conf.php`,修改其中数据ip地址，数据库名，用户名密码

```
<?php
// Zabbix GUI configuration file.
global $DB;

$DB['TYPE']				= 'MYSQL';
$DB['SERVER']			= 'localhost';
$DB['PORT']				= '0';
$DB['DATABASE']			= 'zabbix';
$DB['USER']				= 'zabbix';
$DB['PASSWORD']			= 'zabbixpass';
// Schema name. Used for IBM DB2 and PostgreSQL.
$DB['SCHEMA']			= '';

$ZBX_SERVER				= 'localhost';
$ZBX_SERVER_PORT		= '10051';
$ZBX_SERVER_NAME		= 'zabbix server';

$IMAGE_FORMAT_DEFAULT	= IMAGE_FORMAT_PNG;

``` 
- 进入zabbix web配置，访问 http://127.0.0.1/zabbix

## 配置 zabbix-agent

- 编辑zabbix-agent配置文件 `vi /etc/zabbix/zabbix_agentd.conf`,找到如下配置

```
Server=127.0.0.1
ServerActive=127.0.0.1
Hostname=Zabbix server
```
其中Server、ServerActive 填的是zabbix-server所在服务器ip地址，Hostname填写的zabbix-agent的名称，唯一不可重复，配置主机监控时填写的主机名称必须和Hostname一致

- 设置开机启动zabbixserver `systemctl enable zabbix-agent`
- 启动zabbix server `systemctl start zabbix-agent`
