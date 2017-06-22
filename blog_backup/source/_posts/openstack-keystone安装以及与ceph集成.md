title: openstack keystone安装以及与ceph集成
date: 2016-04-02 21:24:34
tags:
---

## 安装keystone
之前一直用的centos 6，所以在网上找了一些在centos 6上安装keystone的方法。所有使用yum安装的教程都是

``` ```
yum install epel-release-6-8.noarch
yum install openstack-utils openstack-keystone python-keystoneclient
```
但是，经过试验和分析，发现yum源centos和epel中没有openstack-keystone这个包，在mirror.centos.org中发现centos 6 cloud/x84_64/openstack-juno中没有openstack-keystone; 在centos 7 cloud/x86_64/openstack-kilo/中是有openstack-keystone安装包的。

综上，决定使用centos 7安装openstack-keystone，否则需要自己下载所有的依赖包，很痛苦。

<!-- more -->

1.安装centos 7中openstack对应的yum源
``` ```
[root@localhost hou]# wget http://mirror.centos.org/centos-7/7/cloud/x86_64/openstack-kilo/centos-release-openstack-kilo-1-2.el7.noarch.rpm

[root@localhost hou]# rpm -ivh centos-release-openstack-kilo-1-2.el7.noarch.rpm
```

2.安装keystone

``` ```
[root@localhost hou]# yum install openstack-utils openstack-keystone python-keystoneclient

```

安装完openstack-keystone之后，由于keystone使用了mysql，所以下面需要安装mysql。

## 安装和配置mysql

请参考[mysql安装和配置]()

## keystone初始化mariadb

比较简单的方法如下，该命令完成的功能是创建数据库keystone，创建本地和远程访问的用户名/密码分别为keystone/keystone。
``` ```
openstack-db --init --service keystone
```

但是由于openstack-db中会判断是否安装了mysql-server，我们安装的是mariadb-server，所以只能自己手动完成上述工作。

``` ```
MariaDB [(none)]> create database keystone;
Query OK, 1 row affected (0.00 sec)

MariaDB [(none)]> grant all privileges on keystone.* to 'keystone'@'localhost' identified by 'keystone';
Query OK, 0 rows affected (0.04 sec)

MariaDB [(none)]> grant all privileges on keystone.* to 'keystone'@'%' identified by 'keystone';
Query OK, 0 rows affected (0.00 sec)

MariaDB [(none)]> exit

```

修改/etc/keystone/keystone.conf中的connection为mysql://keystone:keystone@127.0.0.1:3306/keystone

## 设置admin-token

``` ```
openstack-config --set /etc/keystone/keystone.conf DEFAULT admin_token [admin_token]

openstack-config --set /etc/keystone/keystone.conf DEFAULT admin_token admin
```

## 同步数据库

``` ```
keystone-manage db_sync
```

## 启动keystone

``` ```
[root@localhost yum.repos.d]# systemctl start openstack-keystone
[root@localhost yum.repos.d]# ps -ef | grep keystone
keystone  9579     1 10 12:17 ?        00:00:02 /usr/bin/python /usr/bin/keystone-all
keystone  9586  9579  0 12:17 ?        00:00:00 /usr/bin/python /usr/bin/keystone-all
keystone  9587  9579  0 12:17 ?        00:00:00 /usr/bin/python /usr/bin/keystone-all
keystone  9588  9579  0 12:17 ?        00:00:00 /usr/bin/python /usr/bin/keystone-all
keystone  9589  9579  0 12:17 ?        00:00:00 /usr/bin/python /usr/bin/keystone-all
root      9596  5373  0 12:17 pts/1    00:00:00 grep --color=auto keystone
```







