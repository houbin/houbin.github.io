title: keystone认证流程分析
date: 2016-05-08 09:41:06
tags: keystone ceph openstack
---

## 起源
使用ceph作为openstack的对象存储时，ceph的radosgw需要与keystone对接。经过网上的一番搜索，对接ok，但是下面三个东西到底发生了怎样的验证流程，本篇文章希望能够解释该问题。
python-swiftclient  ---> keystone ---> radosgw

<!-- more -->

## keystone四种token

### uuid token
token直接保存在keystone的数据库中，swift需要向keystone请求token的合法性信息，对token的验证是在swift上做的。下面基本符合交互流程，只是细节有点出入，不影响理解。

uuid token交互图：
![uuid token](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/keystone_analyze/uuid_token.png)

### PKI token(pubic key infrastructure)
PKI token中携带了更多的用户信息，同时还附上了数字签名，支持swift在本地验证。

PKI token交互图：
![pki token](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/keystone_analyze/PKI_token.png)

### PKIZ token
PKIZ token是PKI token的压缩版，但压缩效果有限。
前三种都会持久化到数据库中。

### Fernet token
这个token没有看。

以上都是参考链接[理解 Keystone 的四种Token](http://www.openstack.cn/?p=5120)

## 测试环境
|tool                 | ip address       | detail              |
| ------------------- | ---------------- | ------------------- |
| python-swiftclient  | 192.168.222.129  | swift测试工具         |
| keystone            | 192.168.1.28     | keystone服务         |
| radosgw             | 192.168.1.50     | swift对象存储服务     |
其中，radosgw中配置了tenant/user/password/role分别为aaa/aaa/aaa/admin的管理账户，且允许admin/member两种role访问。
python-swiftclient使用tenant/user/password/role分别为houbin/houbin/houbin/admin的用户发起请求。

我们测试的时候使用的是四种token的哪个呢？

在keystone配置/etc/keystone/keystone.conf中找到如下内容，默认为uuid

``` powershell
# Controls the token construction, validation, and revocation
# operations. Core providers are
# "keystone.token.providers.[pkiz|pki|uuid].Provider". The
# default provider is uuid. (string value)
#provider=<None>

# Token persistence backend driver. (string value)
#driver=keystone.token.persistence.backends.sql.Token
```

配置文件会不会骗我们？
keystone源码中provider.py的class Manager，为了容易显示代码逻辑，这里把log和exception的打印信息都省略了。
``` powershell
@classmethod
def get_token_provider(cls):
      if CONF.signing.token_format:
        try:
		mapped = _FORMAT_TO_PROVIDER[CONF.signing.token_format]
        except KeyError:
            raise exception.UnexpectedError("error")
        return mapped
    if CONF.token.provider is None:
        return UUID_PROVIDER
    else:
        return CONF.token.provider
```

综上，确认了使用的uuid token这种方式。

在python-swiftclient端使用如下命令发起请求，分别在三方使用tcpdump抓包。
``` powershell
 swift --debug --info -v --os-auth-url http://192.168.1.28:5000/v2.0 --os-tenant-name houbin --os-username houbin --os-password houbin stat
```

## 抓包内容

### python-swiftclient
抓包命令
``` powershell
tcpdump -i ens33 port 80 or port 5000 or port 35357 -w /tmp/swiftclient.cap
```

#### 向keystone请求token1
-  swiftclient首先使用tenant/user/pass为houbin/houbin/houbin向keystone请求token1
-  keystone返回token1和serviceCatalog，token1为4ab5bd3bde7d4cdd83042b66620d038d。

消息内容为
token1请求消息：
![token1请求消息](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/keystone_analyze/swiftclient-req-token1.png)

token1回复消息：
![token1回复消息](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/keystone_analyze/swiftclient-resp-token1.png)

#### 向keystone请求token2
- swiftclient使用 (houbin的tenant-id)/user/pass为9e2deaaf93c348538f53c149ac2d018e/houbin/houbin向keystone请求token2
- keystone返回token2和serviceCatalog，token2为dc0fdca3b0da455aaed1785d74d9c623。

消息内容
token2请求消息：
![token2请求消息](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/keystone_analyze/swiftclient-req-token2.png)
token2回复消息:
![token2回复消息](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/keystone_analyze/swiftclient-resp-token2.png)

#### 向radosgw请求swift服务
- swiftclient使用token2为dc0fdca3b0da455aaed1785d74d9c623请求swift service，请求的命令为swift stat
- radosgw返回账户的信息

消息内容
请求swift服务：
![请求swift服务](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/keystone_analyze/swiftclient-req-swift-service.png)
回复swift服务：
![回复swift服务](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/keystone_analyze/swiftclient-resp-swift-service.png)

### keystone
抓包命令
``` powershell
tcpdump -i enp5s0f0 port 5000 -w /tmp/keystone.cap
```

#### 被swiftclient请求token1
- 被swiftclient请求获取token1
- 返回token1和serviceCatalog，token1为4ab5bd3bde7d4cdd83042b66620d038d。

消息内容与python-swiftclient的请求token1一样

#### 被swiftclient请求token2
- 被swiftclient请求token2
- keystone返回token2和serviceCatalog，token2为dc0fdca3b0da455aaed1785d74d9c623。

消息内容与python-swiftclient的请求token2一样

#### 被radosgw请求token3
- 被radosgw使用tenant/user/pass为aaa/aaa/aaa，即在ceph的配置文件中配置的管理账户，请求token3
- keystone返回token3和serviceCatalog，token3为0a180e7831cb4edf950e62f5866ce013。

消息内容如下
请求报文被分割成三个报文，主要内容在前两个报文截图中
被radosgw请求token3 - 1：
![被radosgw请求token3 - 1](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/keystone_analyze/rgw-req-token3-1.png)
被radosgw请求token3 - 2：
![被radosgw请求token3 - 2](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/keystone_analyze/rgw-req-token3-2.png)

![keystone回复token3和serviceCatalog](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/keystone_analyze/rgw-resp-token3.png)

#### 被radosgw验证token2是否合法

- 被radosgw使用自己的
token3：0a180e7831cb4edf950e62f5866ce013
验证swiftclient的
token2：dc0fdca3b0da455aaed1785d74d9c623
是否合法。
- keystone返回当前swift服务对应的token为token2和serviceCatalog，用于swift服务自己进行验证token的合法性。

消息内容如下
radogw验证token2是否合法：
![radogw验证token2是否合法](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/keystone_analyze/rgw-req-token2-legal.png)

keystone回复当前token和serviceCatalog：
![keystone回复当前token和serviceCatalog](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/keystone_analyze/rgw-resp-token2-legal.png)

### radosgw

抓包命令
``` powershell
tcpdump -i eth0 port 80 -w /tmp/rgw.cap
```
由于抓包是80端口，所以抓不到radosgw向keystone请求token3，然后验证token2的流程。

#### 被swiftclient请求swift服务
- 被swiftclient使用token2请求swift服务
- radosgw回复swiftclient账户的详细信息

现在猜着在被swiftclient使用token2请求swift服务之后，应该会有radosgw向keystone请求验证token2是否合法的过程，而且符合我们上面对keystone中包的分析。
有个佐证：请求消息时间 -> 发出回复消息时间 经过了0.19秒，这个过长时间可以佐证经过了验证token2的流程。

消息同swiftclient抓的包一致。

## 总结
综上分析，uuid token这种方式会把token保存在keystone的数据库中，由keystone提供token的验证信息，swift自己来验证token的合法性。


