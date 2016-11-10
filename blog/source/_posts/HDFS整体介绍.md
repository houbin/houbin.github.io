title: HDFS整体介绍
date: 2016-03-12 02:28:43
tags:
---

## 	HDFS介绍
HDFS是基于消费级硬件设计的分布式文件系统。与其他分布式文件系统区别为HDFS专为大文件设计，且具有高容错性。它提供了较少的POSIX接口。它现在是apache hadoop的子项目。	

<!-- more -->

## 假设和目标

1. 硬件故障
硬件故障作为一个常态存在，因此发现故障和快速、自动恢复是HDFS的架构设计目标

2. 流数据访问
应用程序需要流式的读取数据。HDFS不是通用的存储，它是专门为大文件设计的存储，对于批量的处理比较在行。它的设计目标倾向于高带宽，而非单个数据访问的低延时。POSIX中定义了一些hard requirement，但是这些hard requirement不是HDFS的应用程序所需要的。所以HDFS牺牲了一些POSIX的实现换取了数据读取的高带宽。

3. 大块数据集
HDFS最终的数据存储是写到一个个大文件上的，这就是HDFS为了达到对于大文件的更优支持而设计的。HDFS支持在单个实例上保存上千万个这样的大文件。

4. 简单的一致性模型
HDFS的应用需要一个write-once-read-many的文件访问模型。一个文件被created、writen and closed之后就永远不会被changed，这种前提性约束简化了数据的一致性模型，也极大地提高了数据读写的吞吐量。map-reduce的应用程序对于数据的需求也符合write-once-read-many的文件访问模型。

5. 计算迁移比数据迁移便宜
数据本地化对于计算有很大的便利，这样可以减少数据传输导致的网络压力和延迟。这种思路导出的推论是“把计算移动到数据在的位置" 比 "把数据移动到计算的位置" 有很大的好处。HDFS向应用程序提供将应用程序移的与数据近点的接口。

6. 对于各类硬件平台和软件平台的适应性
HDFS要设计成能够很方便的在各类平台迁移。

## NameNode和DataNodes
HDFS的NameNode采用了master/slave的架构。一个HDFS集群只有一个NameNode，NameNode负责管理文件系统的namespace，控制客户端对文件的访问。HDFS有多个DataNodes，一般一个节点一个DataNode，用于管理该节点下的所有存储。HDFS向外提供了一个文件系统的namespace，并允许用户数据以文件的形式存储。内部实现中，一个文件会被分成one or more blocks，这些blocks会被存到一个DataNode set中。DataNode就负责处理来自文件系统客户端的读写请求，同时DataNode也处理来自NameNode关于block的creation、deletion和replication。HDFS的数据流图如下
![HDFS数据流图](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/hdfs_architecture.gif)

在整个HDFS中，NameNode启动一个仲裁的作用，它提供了同一个namespace，负责整个文件目录树、文件和block的对应关系、block和DataNode的关系。

## 文件系统namespace
HDFS提供了传统文件系统的目录结构，一个用户可以创建一个目录，可以在这个目录中创建文件、删除文件、移动文件到其他目录、重命名文件。HDFS没有实现基于用户的配额，也没有支持soft link、hard link。但是，hdfs不排除会做这些feature。

## 数据副本
HDFS被设计成可靠的跨节点存储大文件，它是以block的形式存储文件；所有的block除了最后一个block都是一样的大小。block会被做副本以防止故障。block的大小和副本数是按文件配置的。应用程序可以指定单个文件的配置。副本数是文件在被create的时候指定，可以在之后被修改。HDFS中的文件是写一次，而且严格的只有一个writer。

NameNode会定期的收到来自DataNode的heartbeat和Blockreport。heartbeat是用于标识datanode还活着，blockreport包含了一个datanode的block list。block repliaction的数据流图如下
![block replication数据流图](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/hdfs_datanodes_block_replication.gif)

如图所示，NameNode对文件进行划分的结构如下
/users/sameerp/data/part-0, r:2, {1, 3} ...

1.副本分布
HDFS的副本一般为3个（这里说的是全部的数据副本数），分布方式为1个副本在一个node上，另外两个在另外一个机柜上的不同node上。
有个推论是由于机柜中节点之间是通过同一交换机连接的，不同机柜之间是两个交换机通过一条线连接，可知同个机柜的带宽是便宜的，而不同机柜之间的带宽是贵的。HDFS的副本写方式是流水式，即a -> b -> c，请求返回是c -> b -> a，这样将后两个副本放在一个机柜上就可以减少机柜之间的带宽压力。

2.副本读取
HDFS读取数据倾向于选择同一机柜上的数据。如果没有同一机柜上的数据，就倾向于选择同一data center的数据

3.safemode
HDFS在启动时NameNode首先进入safemode，这时NameNode没有data block的副本信息。NameNode收到DataNode的heartbeat和blockreport之后，如果block的副本数超过了指定的副本最小数目，则NameNode认为该block是被安全复制的。当被安全复制的副本比例达到指定值外加30s，则NameNode退出safemode。这时，NameNode会将少于指定比例副本数的block拷贝到其他的DataNode上。

## 文件元数据的persistence（持久化）

HDFS的命名空间即所有的元数据存储在NameNode上。NameNode使用一个transaction log（名字为EditLog）持久化保存文件系统元数据的每一个变化，例如，创建文件，改变文件的副本都会在NameNode中插入一条记录。上面说的用一句话就是所有的元数据增量变化存储在EditLog中。
元数据的全局即namespace存储在fsimage中，fsimage也使NameNode本地文件系统的一个文件。

HDFS在内存中保存全部文件系统的一个镜像，当元数据发生变化时，一方面更新内存中的这个全部文件系统的镜像，一方面更新EditLog文件（这个是要确认sync的）。而当前持久化的全部文件系统文件不一定是最新的。
HDFS的架构文档中说现在的实现是只有NameNode启动时，才会使用Editlog更新到持久化的fsimage上，之后会采用定时更新fsimage的方法。

## 传输协议
所有的数据通信都是基于tcp的。NameNode侦听配置好的端口，client和datanode通过该端口与namenode通信。所有的通信协议都是通过RPC实现的。

## Robustness
HDFS的设计目标是在故障成为常态时提供可靠的存储。常见的故障包含NameNode故障、DataNode故障和网络故障。

1.磁盘故障、心跳和re-replication
每一个DataNode都会定时的向NameNode发送心跳。当出现网络故障时，NameNode在一段时间不会收到心跳，则NameNode就会认为该DataNode出现故障，现在就需要重新副本。NameNode总是会检查block是否要re-replication，导致这种情况有如下原因：DataNode失效、副本损坏、磁盘坏掉、文件的副本数发生变化。

2.cluster重平衡
新加NameNode时，重新平衡？

3.数据一致性
客户端在写入文件时，会计算每一个block的checksum，并放到NameNode中。当客户端读取数据时，会计算每个block的数据的checksum，并与NameNode中的checksum进行比对。

4.元数据磁盘故障
fsimage和editlog是HDFS的中心。存放fsimage和editlog的磁盘故障比较要命，所以NameNode需要被配置成fsimage和editlog多副本，这就导致降低元数据的更新速度（sync更新多个editlog文件），但是这个降级是可以接受的，因为HDFS的应用对于元数据不是那么的敏感。

## 数据组织
1.Data block
HDFS为大文件设计的，HDFS的应用也是比较大的文件，而且符合write-once-read-many的语义。一个典型的HDFS的块大小为64M，一个文件会被分割成若干个64M的块，每个块会被分配到不同的datanode上。

2.staging
客户端创建文件时，不是向NameNode发送创建文件的消息，而是先将要写的文件存到本地的临时文件。当本地的临时文件达到HDFS的块大小时，客户端便将缓存的发送到NameNode指定的DataNode。当文件关闭时，客户端缓存的数据会flush到DataNode中。客户端这时会告诉NameNode文件已经关闭。这时，NameNode便会将文件创建操作持久化。如果NameNode在文件关闭前挂掉，文件就会丢失。

3.副本流水线
当客户端向HDFS的文件写数据时，客户端从NameNode得到一个DataNode集合，假设该文件的副本数是3。客户端将数据发送到第一个DataNode，然后DataNode以4K大小接收到数据，然后将数据写到它的数据仓库中（现在是弄文件系统），在写入数据仓库的同时，数据发送到第二个DataNode，第二个DataNode的行为与第一个DataNode一样。

## 访问
HDFS提供了许多形式的访问方式。方法1是HDFS提供了一套JAVA的api；方法2是提供了一套c接口；方法3是可以通过http浏览器查看HDFS的文件。

1.FS shell
HDFS允许用户的数据以文件和目录的形式组织。HDFS提供了一套命令集合，可以通过命令以类似文件系统操作的形式操作HDFS系统。

``` bash
创建一个/foodir的目录 		bin/hadoop dfs -mkdir /foodir
删除一个/foodir的目录		bin/hadoop dfs -rmr /foodir
查看一个/foodir的目录		bin/haddop dfs -cat /foodir/myfile.txt
```

2.HDFS管理
``` bash
集群进入safemode			bin/hadoop dfsadmin -safemode enter
生成DataNode的列表 		bin/hadoop dfsadmin -report
增加DataNode				bin/hadoop dfsadmin -refreshNodes
```

3.浏览器接口
HDFS提供了一个web server，这个web server绑定一个可配置的tcp port。

## 空间回收

1.文件删除与去删除
当HDFS中删除一个文件时，文件会被重命名到/trash目录，所以文件可以被恢复，当且仅当它还在/trash目录中。/trash目录中的文件的生命期是6个小时。所以从文件被删除，到HDFS的系统空间增大会有一个延时。

2.减少副本数量
例如现在副本数量为3，将副本数量设置为2。在下一次心跳时，会将该信息传输给DataNode，这样DataNode就会删除对应的block。所以从减少副本数量到HDFS容量变化也会有一个延迟。


