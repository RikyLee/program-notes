---
title: OpenSSH 实现基于PublicKey认证实现无密登录
date: 2018-02-07 16:12:00
tags: 
  - OpenSSH 
categories: 
  - linux  
---

准备两台服务器A、B，其中A(192.168.1.100)为客户端，B(192.168.1.101)为服务端，实现A服务器通过公钥（publickey）认证，无密码登录B服务器

- 在A服务器上生成用于认证的公钥，默认使用rsa加密 `ssh-keygen`，然后一直按回车键

  ```bash
  [root@localhost ~]# ssh-keygen
  Generating public/private rsa key pair.
  Enter file in which to save the key (/root/.ssh/id_rsa):
  Enter passphrase (empty for no passphrase):
  Enter same passphrase again:
  Your identification has been saved in /root/.ssh/id_rsa.
  Your public key has been saved in /root/.ssh/id_rsa.pub.
  The key fingerprint is:
  c7:35:8f:c1:9f:2c:cb:32:29:2d:ac:57:f4:90:d9:70 root@localhost
  The key's randomart image is:
  +--[ RSA 2048]----+
  |                 |
  |          ..E    |
  |           *=    |
  |         .=..B . |
  |        S.ooo =  |
  |       . o.o.o   |
  |        +.= o    |
  |       ..o o     |
  |      ..         |
  +-----------------+
  ```

  即可在`/root/.ssh`目录下找到 id_rsa,id_rsa.pub两个文件，id_rsb.pub就是我们本次操作需要用到的公钥。

- 拷贝A服务器上生成的公钥到B服务器上

  ```bash
  [root@localhost ~]$ ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.1.101
  /usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "~/.ssh/id_rsa.pub"
  /usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
  /usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
  root@10.3.175.53's password:

  Number of key(s) added: 1

  Now try logging into the machine, with:   "ssh 'root@192.168.1.101'"
  and check to make sure that only the key(s) you wanted were added.

  ```

- SSH登陆B(192.168.1.101)服务器，查看`~/.ssh/authorized_keys`是否存在

- 修改`/etc/ssh/sshd_config`文件

  ```bash
  # CentOS 7.4 #PubkeyAuthentication yes 前面的#去掉即可
  PubkeyAuthentication yes

  # CentOS 7.4以前版本 把以下几项前面的#去掉
  #RSAAuthentication yes
  RSAAuthentication yes
  #PubkeyAuthentication yes
  PubkeyAuthentication yes
  #AuthorizedKeysFile     .ssh/authorized_keys
  AuthorizedKeysFile      .ssh/authorized_keys

  ```

- 重启B(192.168.1.101)服务器sshd进程,根据系统版本，以下三个命令都可以达到相同的效果

  ```bash
  /etc/init.d/sshd restart

  service sshd restart

  systemctl restart sshd.service

  ```

- 在A(192.168.1.100)服务器上使用 `ssh root@192.168.1.101`,即可直接登录B(192.168.1.101)服务器。

- 填坑

  ```txt
  1、关闭B(192.168.1.101)服务器上的selinux sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config

  2、如果要B(192.168.1.101)服务器用户不是root用户，请重新在A(192.168.1.100)服务器上执行 ssh-copy-id -i ~/.ssh/id_rsa.pub username@192.168.1.101,并且保证该用户的根目录,username/.ssh目录，以及authorized_keys文件的权限同下面一致

  [username@localhost home]$ ll
  total 4
  drwx------. 14 username username 4096 Feb  7 11:12 username

  [username@localhost ~]$ ll
  drwx------.  2 username username   48 Feb  7 15:30 .ssh

  [username@localhost ~]$ ll -alh .ssh/
  total 28K
  drwx------.  2 username username   48 Feb  7 15:30 .
  drwx------. 14 username username 4.0K Feb  7 11:12 ..
  -rw-------   1 username username  399 Feb  7 15:30 authorized_keys
  -rw-r--r--.  1 username username  19K Feb  7 11:13 known_hosts

  ```