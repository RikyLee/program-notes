# Centos 7 添加交换区

- 新增交换区

```shell
dd if=/dev/zero of=/var/swap bs=1024 count=8192000
chmod 600 /var/swap
mkswap /var/swap
swapon /var/swap
```

- 添加到/etc/fstab

```text
/var/swap                                 swap                    swap    default         0 0
```
