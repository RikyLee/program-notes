## 编辑并启用网卡ens160  `vi  /etc/sysconfig/network-scripts/ifcfg-ens160` 
``` 
TYPE=Ethernet
# 使用静态IP，如果想要自动分配，改为dhcp
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
# 当前网卡名称
NAME=ens160
UUID=b36e2765-67c6-477f-abcc-43d7ab6935cf
# 设备名称，改为当前编辑的网卡名称
DEVICE=ens160
# yes为启用网卡，no 为禁用
ONBOOT=yes

#当BOOTPROTO=static或者none使用 开始
#ip地址
IPADDR=10.3.175.21

PREFIX=22

#网关
GATEWAY=10.3.175.253

#子网掩码
NETMASK=255.255.252.0

#DNS
DNS1=10.3.175.253
#当BOOTPROTO=static或者none使用 结束

IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_PRIVACY=no
ZONE=public
````
## 编辑并启用网卡ens192  ` vi /etc/sysconfig/network-scripts/ifcfg-ens192` 
```
TYPE=Ethernet
BOOTPROTO=static
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ens192
UUID=d4df7f6a-17a8-425c-ac37-3359eb4f2eef
DEVICE=ens192
ONBOOT=yes
IPADDR=172.16.11.21
NETMASK=255.255.255.0
ZONE=public
```
## 添加静态路由  `vi /etc/sysconfig/network-scripts/route-ens192 `
``` 
172.16.20.0/24 via 172.16.11.1 dev ens192
172.16.10.0/24 via 172.16.11.1 dev ens192
```
## 重启网络服务 ` service network restart`
