> 2021-12-01更新：
>
> 谷歌自己对Chubby的总结：
>
> A Chubby service consists of five active replicas, one of which is elected to be the master and actively serve requests. The service is live when a majority of the replicas are running and can communicate with each other. Chubby uses the Paxos algorithm [Chandra et al. 2007; Lamport 1998] to keep its replicas consistent in the face of failure. Chubby provides a namespace that consists of directories and small files. Each directory or file can be used as a lock, and reads and writes to a file are atomic. The Chubby client library provides consistent caching of Chubby files. Each Chubby client maintains a session with a Chubby service. A client’s session expires if it is unable to renew its session lease within the lease expiration time. When a client’s session expires, it loses any locks and open handles. Chubby clients can also register callbacks on Chubby files and directories for notification of changes or session expiration.

## 0 Abstract

Chubby旨在为**松散（loosely-coupled）分布式系统**提供一套**粗粒度锁服务**，以及**可靠的存储服务**。

Chubby提供一套很像**带有advisory locks**的**分布式文件系统**的接口。

Chubby注重**可靠性**、**可用性**（availability），而非性能。

chubby是一个独立的服务，不专为GFS设计。所以GFS的master应该是另外的，和chubby中的master不是同一个概念，chubby只是一个lock service。

chubby使用paxos实现一致性，其它的服务借助chubby提供的lock service实现自己的一致性。



## 1 Introduction

Chubby的设计主要针对的是**通过高速网络互连的大量主机组成的松散分布式系统**，例如在同一个数据中心或者同一个机房的多台主机。

Chubby的目的是让clients同步他们的活动，并在一些基本信息上达成一致。Chubby的接口类似一个文件系统，提供整文件（whole-file）的读和写、咨询锁（advisory locks）、以及增加了一些event（例如file modification）。

Chubby的主要功能是给用户提供一种粗粒度同步服务，例如**在多台服务器之间选出leader**。GFS和BigTable中都用到了Chubby，把Chubby可以抽象成一个**存放分布式系统元数据**的地方。

在Chubby之前，谷歌的大多数分布式系统用的是ad hoc方法来进行相关操作，或者采用人工干预的方法来操作。



## 2 Design

### 2.1 Rationale

在分布式系统中，对于共识（Consensus）问题，一些人认为应该在现有的每个client主机上实现共识协议（例如paxos），而chubby则是独立出来的一组服务器提供中心化文件服务。

这样的好处就是简单高效、且拓展性好，可以很方便为各种分布式集群提供lock服务。并且这种带有锁服务的文件系统中的文件既可以当做锁（谁获取到一个文件就相当于获取到一个锁），还可以在文件中存储相关信息，例如向集群中其它主机公布"选举出的结果"。

Chubby中在client中缓存了用户使用过的文件，并不采用时间过期机制，而是当原始文件发生更新时，由Master通知每个缓存的replica过期。Chubby采用锁机制的接口，使得程序员能更方便地使用。

Chubby服务本身由多个服务器组成，其中一个被选举为Master。Chubby在其**内部运行有共识协议**，一般一个cell有五个severs，其中必须三个正在运行时才能为外面的client提供服务。

![image-20211130133209454](./images/image006.png)

粗粒度锁就是可能锁几小时或几天的锁，细粒度则是几秒甚至更短的时间。粗粒度的锁有许多好处：

- 首先lock sever的工作负载会小很多，
- 且使用粗粒度锁则lock频率和用户进程事务频率的关联就不大（如果是细粒度锁，则client事务频率的增加就很可能造成lock频率成正比增加），
- 且短时间的锁服务unavailablility不会影响用户程序（因为client很少和锁服务交互），
- 采用粗粒度锁，则锁服务器故障不应该导致lock丢失，
- 最后粗粒度锁使得大量client可以用几个lock server即可服务。

细粒度锁也有好处，采用细粒度锁则锁服务可以在故障时直接丢失所有的锁，因为锁被持有的时间很短，因此不会造成太大影响。



### 2.2 System Structure

如上面的图片，有client library，有server。除此之外还有个可选的proxy sever没画出来。五个lock sever组成一个cell，采用paxos算法选出master。

每个lock server都维持同样的一个数据库，由master负责读写，其它replica只是简单地从master中复制update（使用paxos算法）。

client通过DNS向任意一个lock sever发送master location request来获取此时此刻的master的位置。

写请求先通过paxos算法给cell中每个主机发消息，当得到过半主机的ack后，master才执行写操作。读请求则由master单独便可执行。

如果cell中一个成员故障了且在几个小时内无法恢复，则一个简单的替换系统会从空闲池中选择一台新机器开始运行chubby二进制文件，然后它更新DNS表（用新机器的ip地址替换旧机器的ip地址）。master会定期轮训DNS列表并注意到其中的变化，然后更新成员列表，这个成员列表通过一个简单的复制协议存储在每个成员主机中且保持一致性。新主机会去一个file server获取数据库的**较近版本**，并从当前一个活着的成员那里获取到**最新的updates**，这样操作下来新主机的数据库就和其他成员的保持相同。



### 2.3 Files, Directories, Handles

Chubby提供成类似文件系统的接口给client app使用。

例如``/ls/foo/wombat/pouch``，其中ls代表lock sevice，foo是cell名称（可以用cell名称通过DNS查询lock servers）（特殊的cell名称local，一般是指同一栋楼中的chubby），``/wombat/pouch``就是cell内部的文件位置，一个文件里面的内容一般是a sequence of uninterpreted bytes。

chubby中的文件系统服务不提供文件从一个目录到另一个目录的操作，不维护目录的修改时间、文件的上次访问时间，权限控制只由文件本身权限决定（而与路径中的目录无关），没有软硬链接。

chubby的namespace中只包含目录和文件，统称为node，每个node对应一个唯一的路径。node有永久node和暂时node之分，暂时node可以用来表明一个client是否alive。任何一个node都可以用作读写咨询锁。

chubby中每个node有自己的用户权限表ACL（access control list），新建一个文件时，会默认继承父目录的ACL。

chubby中每个node的元数据有以下内容：

1. instance number：有点类似inode number，总之当前node的instance number比之前同名文件的都要大；
2. content generation number：表明文件内容是第几代了，每次写文件都会加一；
3. lock generation numbser：表明是第几次获取该文件了，每次lock该文件都会加一；
4. ACL generation number：表明ACL表是第几代了，每次写ACL表都会加一；
5. file-content checksum：文件内容生成的checksum，用来表明文件内容是否发生改动；

chubby中client每次打开一个文件，便可获得文件的handle（类似UNIX中文件描述符），handle包含以下内容：

1. check码，防止client伪造handle；
2. sequence number，master可凭这个码来判断该handle是由自己还是前任master生成；
3. mode information，记录了文件打开时的信息，如果旧handle交给新的重启的master，则会让master重新构建自己的state；



## 2.4 Locks and Sequencers

> 看着有点难懂，还是看他人的博客：http://systemdesigns.blogspot.com/2016/01/chubby-lock-service_10.html?view=sidebar

