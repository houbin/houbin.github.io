title: ceph中使用civetweb运行radosgw
date: 2016-03-16 17:45:37
tags: ceph radosgw civetweb
---

## ceph中使用civetweb运行radosgw
此文章是参考下面两个文章所写，如要追溯求源，请直接查看下面两个连接内容。
[有云Ceph课堂：使用CivetWeb快速搭建RGW](https://www.ustack.com/blog/civetweb/)
[Run ceph radosgw with civetweb](http://hustcat.github.io/run-ceph-radosgw-with-civetweb/)

ceph从firefly开始支持使用civetweb作为radosgw的前端web协议引擎，相对于之前的使用apache，安装更加的方便，

<!-- more -->

## 准备
- ceph集群系统
- ceph集群已经创建rgw pool
- 需要运行radosgw的机器已经安装radosgw

## 创建RGW需要使用的存储池
``` ```
ceph osd pool create .rgw 64 64
ceph osd pool create .rgw.root 64 64
ceph osd pool create .rgw.control 64 64
ceph osd pool create .rgw.gc 64 64
ceph osd pool create .rgw.buckets 64 64
ceph osd pool create .rgw.buckets.index 64 64
ceph osd pool create .log 64 64
ceph osd pool create .intent-log 64 64
ceph osd pool create .usage 64 64
ceph osd pool create .users 64 64
ceph osd pool create .users.email 64 64
ceph osd pool create .users.swift 64 64
ceph osd pool create .users.uid 64 64
```

``` ```
[root@pc ceph-deploy]# rados lspools
rbd
.rgw
.rgw.root
.rgw.control
.rgw.gc
.rgw.buckets
.rgw.buckets.index
.log
.intent-log
.usage
.users
.users.email
.users.swift
.users.uid
```

## 在ceph集群中创建user和keyring

1.创建RGW用户houbin对应的keyring文件
``` powershell
[root@inspur44 ~]# ceph-authtool --create-keyring /etc/ceph/ceph.client.houbin.keyring
creating /etc/ceph/ceph.client.houbin.keyring 
```

2.在keyring文件中产生对应的key
``` powershell
[root@inspur44 ~]# ceph-authtool /etc/ceph/ceph.client.houbin.keyring -n client.houbin.gateway --gen-key
[root@inspur44 ~]# cat /etc/ceph/ceph.client.houbin.keyring 
[client.houbin.gateway]
	key = AQDwa+tW+FsqCRAA1aC7AN6bxfjuE2CuceEYjA==
```

3.keyring添加权限信息
``` powershell
[root@inspur44 ~]# ceph-authtool -n client.houbin.gateway --cap osd 'allow rwx' --cap mon 'allow rwx' /etc/ceph/ceph.client.houbin.keyring 
[root@inspur44 ~]# cat /etc/ceph/ceph.client.houbin.keyring 
[client.houbin.gateway]
	key = AQDwa+tW+FsqCRAA1aC7AN6bxfjuE2CuceEYjA==
	caps mon = "allow rwx"
	caps osd = "allow rwx"
```

4.将key添加到集群中
``` bash
[root@inspur44 ~]# ceph -k /etc/ceph/ceph.client.admin.keyring auth add client.houbin.gateway -i /etc/ceph/ceph.client.houbin.keyring 
added key for client.houbin.gateway
```

至此，已经在ceph集群中创建了RGW用户houbin对应的keyring文件，生成了key，添加了权限信息，并已经将key添加到ceph集群之中。

## radosgw节点配置
将ceph集群中/etc/ceph/ceph.conf和/etc/ceph/ceph.client.houbin.keyring文件拷贝到radosgw节点的/etc/ceph目录下，下面的配置便是在radosgw上进行配置。

配置/etc/ceph/ceph.conf文件如下
``` powershell
[root@inspur44 ~]# cat /etc/ceph/ceph.conf 
[global]
auth_service_required = cephx
filestore_xattr_use_omap = true
auth_client_required = cephx
auth_cluster_required = cephx
mon_host = 192.168.1.43
mon_initial_members = inspur43
fsid = 44f44941-6cb6-463e-8f73-01949ec3a703

[client.houbin.gateway]
    host=inspur44
    keyring=/etc/ceph/ceph.client.houbin.keyring
    rgw_socket_path=/tmp/radosgw_houbin.sock
    log_file=/var/log/radosgw/radosgw_houbin.log
    rgw frontends="civetweb port=80"
```
其中，keyring文件便是在ceph集群中添加并生成的文件

## 运行radosgw

``` bash
[root@inspur44 ~]# radosgw -c /etc/ceph/ceph.conf -n client.houbin.gateway
2016-03-18 10:59:56.219240 7ff3bef24840 -1 WARNING: libcurl doesn't support curl_multi_wait()
2016-03-18 10:59:56.219243 7ff3bef24840 -1 WARNING: cross zone / region transfer performance may be affected
[root@inspur44 ~]# radosgw -c /etc/ceph/ceph.conf --keyring=/etc/ceph/ceph.client.houbin.keyring --id=houbin.gateway
2016-03-18 11:01:54.364475 7f3318bfb840 -1 WARNING: libcurl doesn't support curl_multi_wait()
2016-03-18 11:01:54.364477 7f3318bfb840 -1 WARNING: cross zone / region transfer performance may be affected
```

## 使用radosgw
1.创建s3用户
``` bash
[root@inspur44 ~]# radosgw-admin user create --uid=user1 --display-name="user1"
{ "user_id": "user1",
  "display_name": "user1",
  "email": "",
  "suspended": 0,
  "max_buckets": 1000,
  "auid": 0,
  "subusers": [],
  "keys": [
        { "user": "user1",
          "access_key": "B6AUUDRNEY4ND3LN9ERP",
          "secret_key": "wHVjfeSIfuJydlg4\/DD0OIE9JUm7RAfi82tbaVuo"}],
  "swift_keys": [],
  "caps": [],
  "op_mask": "read, write, delete",
  "default_placement": "",
  "placement_tags": [],
  "bucket_quota": { "enabled": false,
      "max_size_kb": -1,
      "max_objects": -1},
  "user_quota": { "enabled": false,
      "max_size_kb": -1,
      "max_objects": -1},
  "temp_url_keys": []}
```

2.创建swift用户
``` bash
[root@inspur44 ~]# radosgw-admin subuser create --uid=user1 --subuser=user1:swift --access=full
{ "user_id": "user1",
  "display_name": "user1",
  "email": "",
  "suspended": 0,
  "max_buckets": 1000,
  "auid": 0,
  "subusers": [
        { "id": "user1:swift",
          "permissions": "<none>"}],
  "keys": [
        { "user": "user1",
          "access_key": "B6AUUDRNEY4ND3LN9ERP",
          "secret_key": "wHVjfeSIfuJydlg4\/DD0OIE9JUm7RAfi82tbaVuo"},
        { "user": "user1:swift",
          "access_key": "WVZQIOO7VRIFKN27W329",
          "secret_key": ""}],
  "swift_keys": [],
  "caps": [],
  "op_mask": "read, write, delete",
  "default_placement": "",
  "placement_tags": [],
  "bucket_quota": { "enabled": false,
      "max_size_kb": -1,
      "max_objects": -1},
  "user_quota": { "enabled": false,
      "max_size_kb": -1,
      "max_objects": -1},
  "temp_url_keys": []}
```

3.对swift用户生成swift secret key
``` bash
[root@inspur44 ~]# radosgw-admin key create --subuser=user1:swift --key-type=swift --gen-secret
{ "user_id": "user1",
  "display_name": "user1",
  "email": "",
  "suspended": 0,
  "max_buckets": 1000,
  "auid": 0,
  "subusers": [
        { "id": "user1:swift",
          "permissions": "<none>"}],
  "keys": [
        { "user": "user1",
          "access_key": "B6AUUDRNEY4ND3LN9ERP",
          "secret_key": "wHVjfeSIfuJydlg4\/DD0OIE9JUm7RAfi82tbaVuo"},
        { "user": "user1:swift",
          "access_key": "WVZQIOO7VRIFKN27W329",
          "secret_key": ""}],
  "swift_keys": [
        { "user": "user1:swift",
          "secret_key": "NKW36XGgTfxzyLavg72r7eYddg1wtuqP3Tt4ZSow"}],
  "caps": [],
  "op_mask": "read, write, delete",
  "default_placement": "",
  "placement_tags": [],
  "bucket_quota": { "enabled": false,
      "max_size_kb": -1,
      "max_objects": -1},
  "user_quota": { "enabled": false,
      "max_size_kb": -1,
      "max_objects": -1},
  "temp_url_keys": []}
```
在测试swift的api时，使用的X-Auth-Token字段的值即为"swift-keys"字段中的"secret_key"。

