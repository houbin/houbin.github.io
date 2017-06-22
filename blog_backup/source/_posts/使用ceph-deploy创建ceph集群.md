title: 使用ceph-deploy创建ceph集群
date: 2016-03-15 11:37:34
tags: ceph ceph-deploy
---

## 准备工作
- 最重要的是关闭防火墙和selinux（被这个坑惨了），就不说三遍了
- 所有的节点做好ssh无密码访问设置
- /etc/hosts中设置好所有节点的ip
- 安装ceph需要的rpm包
- 确认/etc/ceph目录中没有任何文件
- df -h命令查看用到的硬盘没有被挂载
- /var/lib/ceph清空，然后创建osd目录
- 确认当前使用的python版本是2.6而非2.7.X

这里只是使用一个虚拟机中的三块硬盘做一个单节点ceph集群

<!-- more -->

## mkdir ceph-deploy

``` ```
[root@pc hou]# mkdir ceph-deploy
```
创建一个用于deploy的目录

## Create a cluster

``` ```
[root@pc ceph]# ceph-deploy new pc
```
pc为本机的HOSTNAME，该命令会创建三个文件

``` ```
[root@pc ceph-deploy]# ls
ceph.conf  ceph.log  ceph.mon.keyring
```
由于我是在单个节点上部署集群，所以需要在ceph.conf中添加
``````
osd pool default size = 1
osd crush chooseleaf type = 0
```

## Add a monitor
``` ```
[root@pc ceph]# ceph-deploy mon create pc
```
该命令创建一个monitor

## Gather key

``` ```
[root@pc ceph-deploy]# ceph-deploy gatherkeys pc
```
该命令收集以下的keyring文件
``` ```
[root@pc ceph-deploy]# ls
ceph.bootstrap-mds.keyring  ceph.bootstrap-osd.keyring
ceph.client.admin.keyring  ceph.conf  ceph.log  ceph.mon.keyring
```
这三个keyring文件用户启动mds、启动osd和client连接使用。

## Add osds

1.list osd
``` ```
[root@pc ceph-deploy]# ceph-deploy disk list pc
。。。
[pc][WARNIN] WARNING:ceph-disk:Old blkid does not support ID_PART_ENTRY_* fields, trying sgdisk; may not correctly identify ceph volumes with dmcrypt
[pc][DEBUG ] /dev/sda :
[pc][DEBUG ]  /dev/sda1 other, ext4, mounted on /boot
[pc][DEBUG ]  /dev/sda2 other, LVM2_member
[pc][DEBUG ] /dev/sdb :
[pc][DEBUG ]  /dev/sdb1 ceph data, prepared, unknown cluster cc315a0b-b0bf-4540-bc58-92af48fb8708, osd.0, journal /dev/sdb2
[pc][DEBUG ]  /dev/sdb2 ceph journal, for /dev/sdb1
[pc][DEBUG ] /dev/sdc :
[pc][DEBUG ]  /dev/sdc1 ceph data, prepared, unknown cluster cc315a0b-b0bf-4540-bc58-92af48fb8708, osd.1, journal /dev/sdc2
[pc][DEBUG ]  /dev/sdc2 ceph journal, for /dev/sdc1
[pc][DEBUG ] /dev/sdd :
[pc][DEBUG ]  /dev/sdd1 ceph data, prepared, unknown cluster cc315a0b-b0bf-4540-bc58-92af48fb8708, osd.2, journal /dev/sdd2
[pc][DEBUG ]  /dev/sdd2 ceph journal, for /dev/sdd1
[pc][DEBUG ] /dev/sr0 other, unknown
```

2.zap disks
``` ```
[root@pc ceph-deploy]# ceph-deploy disk zap pc:sdb pc:sdc pc:sdd

```

3.create osd
``` ```
[root@pc ceph-deploy]# ceph-deploy osd create pc:sdb pc:sdc pc:sdd
```

4.ceph status
``` ```
[root@pc ceph-deploy]# ceph -s
    cluster 16d3b2f2-b4e9-4fd5-bd5b-bac2c2939220
     health HEALTH_OK
     monmap e1: 1 mons at {pc=192.168.222.128:6789/0}, election epoch 2, quorum 0 pc
     osdmap e13: 3 osds: 3 up, 3 in
      pgmap v20: 64 pgs, 1 pools, 0 bytes data, 0 objects
            69960 kB used, 30629 MB / 30697 MB avail
                  64 active+clean
```

## Add mds
若只需要对象存储和块存储，mds进程是不需要的。mds作用是提供文件系统的操作。

1.在对应节点添加/var/lib/ceph/mds目录
``` ```
[root@pc ceph]# mkdir mds
```
2.添加mds
``` ```
[root@pc ceph-deploy]# ceph-deploy mds create pc
```

3.ceph status
``` ```
[root@pc ceph-deploy]# ceph -s
    cluster 16d3b2f2-b4e9-4fd5-bd5b-bac2c2939220
     health HEALTH_OK
     monmap e1: 1 mons at {pc=192.168.222.128:6789/0}, election epoch 2, quorum 0 pc
     osdmap e13: 3 osds: 3 up, 3 in
      pgmap v25: 64 pgs, 1 pools, 0 bytes data, 0 objects
            103 MB used, 45943 MB / 46046 MB avail
                  64 active+clean
```

## 一些命令

1.停止集群
``` ```
[root@pc ceph-deploy]# service ceph stop

```
2.推送配置文件
``` ```
[root@pc ceph-deploy]# ceph-deploy admin pc
```

3.查看所有的osd信息
``` ```
[root@pc ceph-deploy]# ceph osd tree
# id	weight	type name	up/down	reweight
-1	0.02998	root default
-2	0.02998		host pc
0	0.009995			osd.0	up	1
1	0.009995			osd.1	up	1
2	0.009995			osd.2	up	1
```
