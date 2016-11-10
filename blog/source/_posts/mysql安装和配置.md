title: mysql安装和配置
date: 2016-04-02 00:37:01
tags:
---

由于众所周知的mysql已经被oracle收购了，考虑mysql的开发进度和闭源风险，centos 7已经将mysql替换成了同样源码的mariadb。
mariadb是mysql中不满oracle收购的核心开发者(包含创始人)出来在mysql源码基础上做的东西，使用方式与mysql完全一样，非常靠谱，这里就安装了mariadb。

<!-- more -->

## 安装mariadb
``` ```
[root@localhost yum.repos.d]# yum install mariadb-server mariadb
```

## 启动mariadb
``` ```
[root@localhost yum.repos.d]# systemctl start mariadb
```

设置开机自起
``` ```
[root@localhost yum.repos.d]# systemctl enable mariadb
```

## 设置mysql中编码为utf8

``` ```
[root@localhost yum.repos.d]# cat /etc/my.cnf
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
user=mysql
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
collation-server = utf8_unicode_ci
character-set-server=utf8

[mysqld_safe]
log-error=/var/log/mariadb/mysqld.log
pid-file=/var/run/mariadb/mysqld.pid

[client]
default-character-set=utf8

[mysql]
default-character-set=utf8
```

重启mariadb
``` ```
[root@localhost yum.repos.d]# systemctl restart mariadb
```

查看mariadb字符编码
``` ```
MariaDB [(none)]> show variables like 'character%';
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | utf8                       |
| character_set_connection | utf8                       |
| character_set_database   | utf8                       |
| character_set_filesystem | binary                     |
| character_set_results    | utf8                       |
| character_set_server     | utf8                       |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
8 rows in set (0.00 sec)
```

## 查看当前用户信息(这里已经做了允许远程登录mariadb和修改root的密码)
``` ```
MariaDB [(none)]> use mysql;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed

MariaDB [mysql]> select user,host,password from user;
+------+-----------------------+-------------------------------------------+
| user | host                  | password                                  |
+------+-----------------------+-------------------------------------------+
| root | localhost             | *FD571203974BA9AFE270FE62151AE967ECA5E0AA |
| root | localhost.localdomain | *FD571203974BA9AFE270FE62151AE967ECA5E0AA |
| root | 127.0.0.1             | *FD571203974BA9AFE270FE62151AE967ECA5E0AA |
| root | ::1                   | *FD571203974BA9AFE270FE62151AE967ECA5E0AA |
|      | localhost             |                                           |
|      | localhost.localdomain |                                           |
+------+-----------------------+-------------------------------------------+
6 rows in set (0.00 sec)

```

其实，之后对于允许远程登录和修改root密码的操作都是操作mysql.user表。

## 允许远程登录mariadb
``` ```
MariaDB [mysql]> update user set host='%' where user='root' and host='localhost';
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MariaDB [mysql]> flush privileges;
Query OK, 0 rows affected (0.00 sec)

```

sql语句的含义是将mysql.user表中针对原来user='host' and host='localhost'只是针对localhost的权限设置为'%'。

## 添加和修改root密码

``` ```
MariaDB [mysql]> update user set password=password('000000') where user='root';
Query OK, 4 rows affected (0.00 sec)
Rows matched: 4  Changed: 4  Warnings: 0

MariaDB [mysql]> flush privileges;
Query OK, 0 rows affected (0.00 sec)

```

sql语句的含义是修改所有user='root'行中的密码均为password('000000')，密码不能为明文。

这样，mariadb的安装和基本配置就完成了，以后会继续更新mysql的配置部分。



一些mysql的配置命令如下

## root的localhost权限
如果不小心把root用户的localhost登录权限从user表中删除，需要重新加一下，可以使用如下命令。

``` ```
MariaDB [mysql]> insert into user values('localhost','root',password('000000'),'Y','Y','Y','Y','Y','Y','Y','Y','Y','Y','Y','Y','Y','Y','Y','Y','Y','Y','Y','Y','Y','Y','Y','Y','Y','Y','Y','Y','Y','','','','',0,0,0,0,'','');
Query OK, 1 row affected (0.04 sec)

MariaDB [mysql]> flush privileges;
Query OK, 0 rows affected (0.00 sec)

```

其中，user表的行数与mysql的版本匹配。

## 查看mysql中某表的详细信息

``` ```
MariaDB [(none)]> show full columns from mysql.user;
+------------------------+-----------------------------------+-------------------+------+-----+---------+-------+---------------------------------+---------+
| Field                  | Type                              | Collation         | Null | Key | Default | Extra | Privileges                      | Comment |
+------------------------+-----------------------------------+-------------------+------+-----+---------+-------+---------------------------------+---------+
| Host                   | char(60)                          | utf8_bin          | NO   | PRI |         |       | select,insert,update,references |         |
| User                   | char(16)                          | utf8_bin          | NO   | PRI |         |       | select,insert,update,references |         |

```

