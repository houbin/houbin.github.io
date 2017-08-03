---
title: firewalld入门
date: 2017-08-03 10:33:03
tags: firewalld 端口转发
---
## firewalld入门
由于需要使用端口转发功能，选来选去还是使用centos 7的firewalld来作为端口映射工具。

下面对firewalld的使用详细说明一下。

本文参考链接<http://www.jb51.net/article/112698.htm>

<!-- more -->

firewall-cmd需要firewalld进程处于运行状态。当我们修改一些配置之后，可以采用下面两种方式激活配置。
``` shell
# systemctl restart firewalld
# firewall-cmd --reload
```
第一种方式会中断现有的tcp会话，第二种方式不会中断正在连接的tcp会话。但是需要注意的是man firewall的页面中后面带有[P]应该是需要加上--permanent，以保证使用上述两个命令之后仍然有效。我就败在了下面命令下，导致端口转发一直不生效。
``` shell
# firewall-cmd --add-masquerad  --permanent
```

## 启动/停止

```shell
# systemctl list-unit-files | grep firewalld
# systemctl start firewalld
# systemctl enable firewalld
# systemctl enable firewalld
```

## 防火墙功能
firewalld的一个功能是防火墙功能，它可以屏蔽一些端口的使用，一般如果连接到互联网上，则只开启80端口提供访问，对内可以开启mysql、redis的端口。

``` shell
# firewall-cmd --add-service=mysql --permanent          // 开放mysql端口
# firewall-cmd --remove-service=http --permanent        // 阻止http服务端口
# firewall-cmd --list-services --permanent              // 查看开发的服务
# firewall-cmd --add-port=3306/tcp --permanent          // 开放通过tcp访问3306
# firewall-cmd --remove-port=3306/tcp --permanent       // 阻止通过tcp访问3306
# firewall-cmd --list-ports --permanent                 // 查看开发的端口

```

上述命令只是记录，没有做验证。

## 伪装ip
防火墙可以启动伪装ip的作用，端口转发功能会用到这个功能。个人理解是只有开启了伪装ip的功能，才可以起到端口转发的功能。

``` shell
# firewall-cmd --query-masquerade --permanent
# firewall-cmd --add-masquerade --permanent
# firewall-cmd --remove-masquerade --permanent
```

相同的是，上述命令也需要通过 firewall-cmd --reload 命令来生效。

## 端口映射
端口转发配置如下

``` shell
# firewall-cmd --add-forward-port=port=80:proto=tcp:toport=8080 --permanent
# firewall-cmd --add-forward-port=port=80:proto=tcp:toaddr=192.168.1.2 --permanent
# firewall-cmd --add-forward-port=port=80:proto=tcp:toaddr=192.168.1.2:toport=8080 --permanent
# firewall-cmd --list-forward-ports
```

上述命令需要通过 firewall-cmd --reload 命令来生效。

## 配置文件
上述配置完成之后，firewall对应的配置文件目录为
``` shell
[root@inspur zones]# pwd
/etc/firewalld/zones
[root@inspur zones]# cat public.xml
<?xml version="1.0" encoding="utf-8"?>
<zone>
  <short>Public</short>
  <description>For use in public areas. You do not trust the other computers on networks to not harm your computer. Only selected incoming connections are accepted.</description>
  <service name="dhcpv6-client"/>
  <service name="ssh"/>
  <port protocol="udp" port="22"/>
  <port protocol="tcp" port="22"/>
  <masquerade/>
  <forward-port to-addr="100.7.44.66" to-port="22" protocol="tcp" port="16012"/>
</zone>
```

End