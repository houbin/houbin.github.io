---
title: Ceph与HDFS的UGO权限研究
date: 2017-06-23 00:48:07
tags: ceph HDFS ugo
---

# 文章目的
写本文章是由于想使用ceph替换HDFS，来作为hadoop的底层存储。ceph的文件目录权限完全兼容posix，HDFS的文件目录权限只是兼容了一部分，另外有一些是特别的，所以需要也用于比较ceph与HDFS的权限区别，尽量使得ceph与HDFS的UGO权限检查兼容，介于个人能力有限，所以可能有所纰漏，请指正。

 <!-- more -->

# CEPH的ugo权限
这里以fuse挂载为例说明。

当通过fuse客户端访问ceph文件系统时，fuse client获取操作用户的uid，在client完成UGO权限的验证，代码如下图所示。但是这样需要保证集群和客户端所有机器设置的uid和gid保持一致。

fuse客户端默认UGO权限：目录为755，文件为644。

# HDFS的ugo权限
HDFS用户和组确定有两种方式：simple和kerberos，由参数“hadoop.security.authentication”来进行配置，这里只讨论simple。

HDFS的文件和目录权限模型共享了POSIX模型的很多部分，比如每个文件和目录都有user和group，文件对于user、group、other分别有不同的权限设置，HDFS在文件和目录的rwx权限检查与posix一致。

HDFS默认UGO权限：目录为755，文件为644，这个与ceph一致。

不同的是HDFS的权限检查是在namenode进行的，HDFS的超级用户为namenode的启动用户，可以通过配置“dfs.permissions.superusergroup=hdfs”来配置超级用户所在的组，目前所有的hortonworks的发行版，超级用户为hdfs，超级用户组为hdfs。

HDFS的权限控制有两种：simple和kerberos，

与ceph实现区别是HDFS的客户端获取到当前进程的用户名后，将用户名和操作发送到namenode，namenode根据用户名获取用户所在的组，因此使用的用户 - 组的映射关系是namenode所在机器上的映射关系。

## ceph支持HDFS的权限
目前想法是ceph中添加一个超级用户列表配置，对于超级用户列表中的用户，在fuse client认为是超级用户，从而检查永远不失败。

