# FastDFS安装配置

## 安装

- 编译安装libfastcommon

```text
git clone https://github.com/happyfish100/libfastcommon.git
cd libfastcommon/
./make.sh && ./make.sh install
```

- 下载并解压fastdfs压缩包

```text
wget -O fastdfs_V5.11.tar.gz https://github.com/happyfish100/fastdfs/archive/V5.11.tar.gz

tar xf fastdfs_V5.11.tar.gz
```

- 安装fastdfs

```text
cd fastdfs-5.11/

./make.sh && ./make.sh install
```

- 建立tracker数据存放目录

```text
mkdir /data/fastdfs
mkdir /data/fastdfs/fastdfs_tracker
```

- 建立storage存放目录

```text
mkdir /data/fastdfs/fastdfs_storage
mkdir /data/fastdfs/fastdfs_storage_data
```

- 配置tracker

```text
cd /etc/fdfs/
mv tracker.conf.sample tracker.conf

sed -i "s/base_path=\/home\/yuqing\/fastdfs/base_path=\/data\/fastdfs\/fastdfs_tracker/g" tracker.conf

sed -i "s/http.server_port=8080/http.server_port=6666/g" tracker.conf

```

- 配置storage

```text
cd /etc/fdfs/
mv storage.conf.sample storage.conf

sed -i "s/base_path=\/home\/yuqing\/fastdfs/base_path=\/data\/fastdfs\/fastdfs_storage/g" storage.conf

sed -i "s/store_path0=\/home\/yuqing\/fastdfs/store_path0=\/data\/fastdfs\/fastdfs_storage_data/g" storage.conf

sed -i "s/tracker_server=192.168.209.121:22122/tracker_server=10.3.172.149:22122/g" tracker.conf
```

- 启动服务

```text
service fdfs_trackerd start
service fdfs_storaged start

```

- 查看storage的状态

```text
/usr/bin/fdfs_monitor /etc/fdfs/storage.conf
```

- 配置client

```text
cd /etc/fdfs/
mv client.conf.sample client.conf
sed -i "s/base_path=\/home\/yuqing\/fastdfs/base_path=\/data\/fastdfs\/fastdfs_tracker/g" client.conf
sed -i "s/tracker_server=192.168.0.197:22122/tracker_server=10.3.172.149:22122/g" client.conf
sed -i "s/http.server_port=80/http.server_port=6666/g" client.conf

```

- 测试client，上传storage.conf文件

```text
/usr/bin/fdfs_upload_file /etc/fdfs/client.conf  /etc/fdfs/storage.conf

```

- 加入开机启动

```text
chmod +x /etc/rc.d/rc.local

vi /etc/rc.d/rc.local

最后添加

service fdfs_trackerd start
service fdfs_storaged start

```

- 下载fastdfs-nginx-module

```text
git clone https://github.com/happyfish100/fastdfs-nginx-module.git
```

- 修改fastdfs-nginx-module/src/config

```text
ngx_addon_name=ngx_http_fastdfs_module

if test -n "${ngx_module_link}"; then
    ngx_module_type=HTTP
    ngx_module_name=$ngx_addon_name
    ngx_module_incs="/usr/local/include"
    ngx_module_libs="-lfastcommon -lfdfsclient"
    ngx_module_srcs="$ngx_addon_dir/ngx_http_fastdfs_module.c"
    ngx_module_deps=
    CFLAGS="$CFLAGS -D_FILE_OFFSET_BITS=64 -DFDFS_OUTPUT_CHUNK_SIZE='256*1024' -DFDFS_MOD_CONF_FILENAME='\"/etc/fdfs/mod_fastdfs.conf\"'"
    . auto/module
else
    HTTP_MODULES="$HTTP_MODULES ngx_http_fastdfs_module"
    NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_fastdfs_module.c"
    CORE_INCS="$CORE_INCS /usr/local/include"
    CORE_LIBS="$CORE_LIBS -lfastcommon -lfdfsclient"
    CFLAGS="$CFLAGS -D_FILE_OFFSET_BITS=64 -DFDFS_OUTPUT_CHUNK_SIZE='256*1024' -DFDFS_MOD_CONF_FILENAME='\"/etc/fdfs/mod_fastdfs.conf\"'"
fi
```

为

```text
ngx_addon_name=ngx_http_fastdfs_module

if test -n "${ngx_module_link}"; then
    ngx_module_type=HTTP
    ngx_module_name=$ngx_addon_name
    ngx_module_incs="/usr/include/fastdfs /usr/include/fastcommon/"
    ngx_module_libs="-lfastcommon -lfdfsclient"
    ngx_module_srcs="$ngx_addon_dir/ngx_http_fastdfs_module.c"
    ngx_module_deps=
    CFLAGS="$CFLAGS -D_FILE_OFFSET_BITS=64 -DFDFS_OUTPUT_CHUNK_SIZE='256*1024' -DFDFS_MOD_CONF_FILENAME='\"/etc/fdfs/mod_fastdfs.conf\"'"
    . auto/module
else
    HTTP_MODULES="$HTTP_MODULES ngx_http_fastdfs_module"
    NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/ngx_http_fastdfs_module.c"
    CORE_INCS="$CORE_INCS /usr/include/fastdfs /usr/include/fastcommon/"
    CORE_LIBS="$CORE_LIBS -lfastcommon -lfdfsclient"
    CFLAGS="$CFLAGS -D_FILE_OFFSET_BITS=64 -DFDFS_OUTPUT_CHUNK_SIZE='256*1024' -DFDFS_MOD_CONF_FILENAME='\"/etc/fdfs/mod_fastdfs.conf\"'"
fi
```

- 安装依赖

```text
yum update

yum -y install pcre pcre-devel openssl openssl-devel gcc gcc-c++ autoconf automake zlib zlib-devel libxml2 libxml2-dev libxslt-devel gd-devel perl-devel perl-ExtUtils-Embed GeoIP GeoIP-devel GeoIP-update
```

- 创建nginx用户与用户组

```text
groupadd nginx
useradd nginx -g nginx -s /sbin/nologin -M
```

- 下载并编译安装nginx

```text
wget -O nginx-release-1.15.3.tar.gz https://github.com/nginx/nginx/archive/release-1.15.1.tar.gz

tar xf nginx-release-1.15.3.tar.gz
cd nginx-release-1.15.3

./configure --prefix=/usr/local/nginx --user=nginx --group=nginx --add-module=/data/fastdfs-nginx-module/src --with-http_ssl_module --with-http_realip_module --with-http_geoip_module e --with-http_sub_module --with-stream --with-stream=dynamic

 make && make install
 ```

- 添加nginx serveic

```text
cat <<EOF >  /usr/lib/systemd/system/nginx.service
[Unit]
Description=nginx
After=network.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t -q -g 'daemon on; master_process on;'
ExecStart=/usr/local/nginx/sbin/nginx -g 'daemon on; master_process on;'
ExecReload=/usr/local/nginx/sbin/nginx -g 'daemon on; master_process on;' -s reload
ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /usr/local/nginx/logs/nginx.pid
TimeoutStopSec=5
KillMode=mixed

[Install]
WantedBy=multi-user.target
EOF

```

- 启动nginx 服务 `systemctl start nginx`

- 配置fastdfs http支持

```text
cp /data/fastdfs-5.11/conf/http.conf /etc/fdfs/
cp /data/fastdfs-5.11/conf/mime.conf /etc/fdfs/
cp /data/fastdfs-nginx-module/src/mod_fastdfs.conf /etc/fdfs/

vi mod_fastdfs.conf

```

复制如下替换mod_fastdfs.conf内容

```text
# connect timeout in seconds
# default value is 30s
connect_timeout=2

# network recv and send timeout in seconds
# default value is 30s
network_timeout=30

# the base path to store log files
base_path=/data/fastdfs/fastdfs_storage

# if load FastDFS parameters from tracker server
# since V1.12
# default value is false
load_fdfs_parameters_from_tracker=true

# storage sync file max delay seconds
# same as tracker.conf
# valid only when load_fdfs_parameters_from_tracker is false
# since V1.12
# default value is 86400 seconds (one day)
storage_sync_file_max_delay = 86400

# if use storage ID instead of IP address
# same as tracker.conf
# valid only when load_fdfs_parameters_from_tracker is false
# default value is false
# since V1.13
use_storage_id = false

# specify storage ids filename, can use relative or absolute path
# same as tracker.conf
# valid only when load_fdfs_parameters_from_tracker is false
# since V1.13
storage_ids_filename = storage_ids.conf

# FastDFS tracker_server can ocur more than once, and tracker_server format is
#  "host:port", host can be hostname or ip address
# valid only when load_fdfs_parameters_from_tracker is true
tracker_server=10.3.142.149:22122

# the port of the local storage server
# the default value is 23000
storage_server_port=23000

# the group name of the local storage server
group_name=group1

# if the url / uri including the group name
# set to false when uri like /M00/00/00/xxx
# set to true when uri like ${group_name}/M00/00/00/xxx, such as group1/M00/xxx
# default value is false
url_have_group_name = true

# path(disk or mount point) count, default value is 1
# must same as storage.conf
store_path_count=1

# store_path#, based 0, if store_path0 not exists, it's value is base_path
# the paths must be exist
# must same as storage.conf
store_path0=/data/fastdfs/fastdfs_storage_data
#store_path1=/home/yuqing/fastdfs1

# standard log level as syslog, case insensitive, value list:
### emerg for emergency
### alert
### crit for critical
### error
### warn for warning
### notice
### info
### debug
log_level=info

# set the log filename, such as /usr/local/apache2/logs/mod_fastdfs.log
# empty for output to stderr (apache and nginx error_log file)
log_filename=

# response mode when the file not exist in the local file system
## proxy: get the content from other storage server, then send to client
## redirect: redirect to the original storage server (HTTP Header is Location)
response_mode=proxy

# the NIC alias prefix, such as eth in Linux, you can see it by ifconfig -a
# multi aliases split by comma. empty value means auto set by OS type
# this paramter used to get all ip address of the local host
# default values is empty
if_alias_prefix=

# use "#include" directive to include HTTP config file
# NOTE: #include is an include directive, do NOT remove the # before include
#include http.conf


# if support flv
# default value is false
# since v1.15
flv_support = true

# flv file extension name
# default value is flv
# since v1.15
flv_extension = flv


# set the group count
# set to none zero to support multi-group on this storage server
# set to 0  for single group only
# groups settings section as [group1], [group2], ..., [groupN]
# default value is 0
# since v1.14
group_count = 3

# group settings for group #1
# since v1.14
# when support multi-group on this storage server, uncomment following section
[group1]
group_name=group1
storage_server_port=23000
store_path_count=1
store_path0=/data/fastdfs/fastdfs_storage_data

# group settings for group #2
# since v1.14
# when support multi-group, uncomment following section as neccessary
[group2]
group_name=group2
storage_server_port=23000
store_path_count=1
store_path0=/data/fastdfs/fastdfs_storage_data

[group3]
group_name=group3
storage_server_port=23000
store_path_count=1
store_path0=/data/fastdfs/fastdfs_storage_data

```

- 修改nginx.conf

```text
#user  nobody;
worker_processes  1;

error_log  logs/error.log warn;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

pid        logs/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  logs/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    #gzip  on;

    include /usr/local/nginx/conf/conf.d/*.conf;
}

```

- 创建nginx自定义配置文件存放路径 `mkdir -p /usr/local/nginx/conf/conf.d`

- 配置storage nginx

```text
cat <<EOF> /usr/local/nginx/conf/conf.d/storage.conf
server {
        listen       8080;
        server_name  localhost;

        location / {
            root   html;
            index  index.html index.htm;
        }

        location ~/group1/M00 {
            root /data/fastdfs/fastdfs_storage_data/data;
            ngx_fastdfs_module;
        }

        location = /50x.html {
            root   html;
        }
}
EOF
```

- 配置 tracker nginx

```text
cat <<EOF> /usr/local/nginx/conf/conf.d/storage.conf
upstream fdfs_group1 {
        server 127.0.0.1:9999;
}
server {
    listen       80;
    server_name  localhost;

    location /group1/M00 {
        proxy_pass http://fdfs_group1;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   html;
    }
}
```

- 重新加载nginx配置文件 `nginx -s reload`

- 开放防火墙端口

```text
firewall-cmd --add-port=80/tcp --permanent
firewall-cmd --add-port=8080/tcp --permanent
firewall-cmd --add-port=22122/tcp --permanent
firewall-cmd --add-port=23000/tcp --permanent
firewall-cmd --reload
```
