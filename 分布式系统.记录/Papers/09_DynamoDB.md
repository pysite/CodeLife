## 1 Introduction

亚马逊的网购服务需要高可靠性（reliability，不能停机），以及高拓展性。系统的可靠性和拓展性往往取决于它的应用的状态是如何维持的。可靠性中特别需要强调的点就是可用性（availability），任何时候用户都可以访问亚马逊平台进行购物。很显然，这个系统是由诸多主机组成，因此随时都会有部分主机发生故障，因此必须要处理这些故障使得不影响系统的可靠性、可用性。

亚马逊平台上的许多服务，例如seller lists、shopping carts，都只需要简单的key访问，而不需要复杂的关系型数据访问（性能低，拓展性低），只需要一种primary-key only interface。

Dynamo为了实现高可靠性，拓展性，采用了以下诸多技术：

1. 数据被切分、replicated，其中使用到了**一致性哈希**；
2. 采用**object versioning技术**进一步保障一致性；
3. 采用**a quorum-like技术**和**分布式replica synchronization protol**保证replica的一致性；

4. 采用**基于gossip的分布式故障检测和成员协议**；

Dynamo不需要太多人工操作，存储节点可以随时加入和退出Dynamo without manual partitioning or redistribution.



## 2 Background

传统的服务器状态存储在关系型数据库中，但其实大部分service所需的服务只不过是通过primary key获取到对应数据，而非复杂的querying，因此关系型数据库底层的那些复杂的基础设施是没必要的，同时也会导致系统性能低下。

### 2.1 System Assumptions and Requirements

> Dynamo所针对的服务的特点

- Query Model

一般来说，DynamoDB上面运行的services查询或修改某个data item一般是只通过key来进行，且没有操作会一次涉及多个item，因此也不需要schema。一般来说服务的状态都是以blob二进制形式（小于1MB）存储的。

- ACID Properties

DynamoDB不保证强一致性，只由weaker consistency，但提供高availability。

- Efficiency

DynamoDB建立基于commodity hardware infrastructure. 在其上的服务一般对延迟有很严格的要求。同时运行在DynamoDB之上的服务可以根据它们自身的需要，来在consistency、availability、cost efficiency等之间进行trade off（按需求configure）。

- Other Assumptions

Dynamo只用于Amazon内部，不涉及安全校验。每种服务有各自的Dynamo实例，其最初的设计规模是数百个主机的规模。

### 2.2 Service Level Agreements（SLA）

SLA就是service和其client之间定的一种约定，例如约定为在每秒500个请求的情况下，99.9%的请求都将会在300ms内被响应。

在Amazon平台上的服务，一个服务可能要调用几百个其它的服务。

DynamoDB所采用的SLA大部分都是99.9%这种比例的约定，而非像其它系统中那样采用average或median值，这样是为了更保证客户体验，从客户角度思考。

### 2.3 Design Considerations

- Data Replication Algorithm

DynamoDB采用的是乐观复制算法，即允许数据短暂不一致的情况，只保证最终一致性。但乐观算法会面临conflicting changes的问题，这些冲突的改动必须被detected和resolved。

解决冲突改动的两大问题，when和who解决。其它系统一般采用在write时解决，例如用户的write如果没能达到每个replica那里就会被拒绝，这种就会导致较低的availability。DynamoDB采用在read时解决，来保证write（修改数据）具有较强的availability。

who解决？即可以是底层的data store，也可以是上层的app来解决。如果是底层的话，则只能采取简单的策略，例如"last write wins"。

- Incremental Scalability

Dynamo在扩增主机（节点）时，应该尽量不影响整体的系统的正常运行。

- Symmetry

Dynamo中的每个主机应该具有和其相同地位的主机的相同responsibilities，不应存在具有特殊角色或额外职责的可区分节点，这样能简化系统的部署和维护。

-  Decentralization

Dynamo应该采用去中心化的设计。

- Heterogeneity

Dynamo的异构性需要保证允许整个系统中的各个主机型号、性能各不相同，并且能够根据它们各自的性能进行按比例分配工作请求，这样当未来加入高性能节点时不必升级之前的每个节点。

## 3 Related Work

### 3.1 Peer to Peer Systems

最早的P2P系统主要用于文件共享系统，早期的非结构化P2P系统节点之间的连接都是任意连接的，会有query泛洪的问题。然后出现了结构化P2P系统，采用全局一致性协议来使得query不需要泛洪，而是可以准确通过路由表到达存有相应数据的节点。在一些结构化的P2P系统中每个节点存储足够多的信息在本地，使得可以进一步加速路由过程，同时使得query可以在限定跳数内得到answer。

### 3.2 Distributed File Systems and Databases

P2P系统一般不支持目录树型的namespace服务，而分布式文件系统则可以支持多层次namespaces。一般采用Replica来提供high availability，如果发生update conflict，则采用specialized conflict resolution procedures，这类文件系统有例如GFS等。

一些文件系统即使是在发生了"网络分区"、"网络中断"等异常情况时也依然能运行。本文的DynamoDB也支持在出现以上网络问题时的持续运行，并保证"最终一致性"，采用多种conflict resolution  mechanisms解决更新冲突。

## 3.3 Discussion

总之本文的DynamoDB，是个KV存储系统，不保证强一致性，只保证最终一致性，目标是提供高可用性的服务，即使是在网络出现问题时依然提供服务（update请求不会被拒绝），并且DynamoDB是用在内部系统中，不关注安全性（信任每个节点）。Dynamo不支持多层次的namespace服务，也不提供像关系型数据库那样的复杂的schema查询。DynamoDB关注时延的问题，SLA以99.9%来作为标准，为了减少时延，Dynamo存储足够多的路由信息在每个节点，以减少查询时的路由次数。



## 4 System Architecture

实际运行在生产环境的存储系统十分复杂，包含以下组件：load balancing, membership, failure detection, failure recovery, replica synchronization, overload handling, state transfer, concurrency and job scheduling, request marshalling, request routing, system monitoring, alarming。

论文中只介绍：partitioning, replication, versioning, membership, failure handling, scaling.

### 4.1 System Interface

Dynamo提供两个API，get(key)和put(key, context, object)，context中包含了数据的元信息，例如object的version（调用者不需要管context）。

Dynamo采用MD5码对key进行计算，计算出一个128位的id，由这个id决定将该object存储到哪些nodes。

### 4.2 Partitioning Algorithm

Dynamo为了支持灵活地变动nodes集合，采用一致性哈希，在一致性哈希中，所有可能的id分布在一个环上，每个node和每个KV值在这个环上都有个位置。

一致性哈希的缺点：

- 数据在各个节点上可能分布不均。
- 此算法忽略了异构系统中每个node的性能不同。

因此，DynamoDB采用了一种不同的一致性哈希算法，就是一个node可以在环上分布多个point（virtual point），有以下三个好处：

- 当节点故障时，其负责的数据可以均匀地分散给其它节点；
- 当节点加入系统时，其又可以均匀地从周围节点中获取数据；
- 一个节点的容量越大、性能越强，系统就可以给其分配更多的virtual node。

### 4.3 Replication

假设一个key值对应的object应该存储N份replica，DynamoDB，KV会先交给一个coordinator node，然后再由这个coordinator向顺时针方向传播N-1次。同时因为virtual point的存在，为了保证KV能存在至少N个不同的物理node上，每个key值有个preference list，coordinator会在向前传播时跳过已经访问过的node（边向前传播，边建立起preference list）。这个preference list中node的数量实际上是比N多的，目的是对节点故障有容错性。

### 4.4 Data Versioning

Dynamo是最终一致性，在key值对应的node上面的更新都是异步的。

Dynamo上的应用必须要明确同一数据存在多个版本的可能性，要有merge多种版本的操作。

Dynamo使用矢量时钟来记录各个object上各个version的先后因果关系，矢量时钟就是(node, counter)这种组成的集合，一个(node, counter)元组就代表这个node的第counter个操作是对这个object进行的操作。矢量时钟存储在context中，在get时可以得到context，在put时应用又会传入一个context。例子：

> 假设有三个node A B C，开始时写入一个数据，由A负责处理这个请求，则此时这个数据的context就是{(A, 1)}，并且这个context也传递给了B和C知晓，然后用户又写入了一个数据，仍然是A负责处理，则此时这个数据的context就是{(A, 2)}，然后用户又写入了一个数据，由B处理，则context是{(A, 2), (B, 1)}，假设此时C出现了网络故障（此时C那边看到的这个object的context仍然是{(A, 2)}）。之后用户又写入了一个数据，由C处理，则此时C中的context是{(A,2), (C, 1)}，然而C在将其消息传递给A和B时，A和B便会发现自己的{(A, 2), (B, 1)}与{(A,2), (C, 1)}是并行关系。此时A和B并不会丢弃这个消息，而是组成a list of objects，并且将context变为{(A, 2), (B, 1), (C, 1)}。
>
> 之后假设用户读取这个数据，由A负责处理，则A便会返回给用户a list of objects（两个object），并且带有context为{(A, 2), (B, 1), (C, 1)}，之后用户写入数据时便会从a list of objects中选择一个object，以及传入context，这样就相当于由用户解决了分支。

### 4.5 Execution of Get() and Put() operations

Dynamo系统里任何一个节点都有资格接受并执行用户的get和put请求。负责处理这个请求的node就叫做coordinator。

有两种策略将用户的请求传送到某个node上：1. 传给load balancer，由这个balancer选择一个node；2. 由用户选择一个node（using a partition-aware client library）。第一种策略不需要用户应用参与Dynamo，第二种需要用户应用参与，但是可以减少一定的latency。

Read或Write操作在执行时，preference list中前N个健康的节点将被访问到。

Dynamo采用quorum-like的replica一致性协议，其中有两个参数W和R，分别代表写入时要求成功写入的replica数量和读取时成功读取的replica数量。

### 4.6 Handling Failures: Hinted Handoff

> 只采用quorum-like策略是不足够应对节点故障或网络故障的情况。

Dynamo采用一种"Hinted Handoff"技术，当一个node故障时，则在write时本该发送到该节点上的数据会被传递给其在环上的下一个节点，这些临时发送给下一个node节点会被打上标签（hinted），且被存储到本地的一个separate数据库中，一旦检测到原先故障的节点恢复时，便会又将这些hinted数据传回去（且在本地数据库中删除）。

Dynamo应对整个数据库故障的情况主要靠的还是preference list，前面说过一个key值对应的preference list上面记录了object所在的node列表，而实际上这些node貌似也是在不同的datacenter中，因此key值对应的各个object实际上是分布在不同datacenter，因此一个datacenter故障掉也不会有太大问题。

### 4.7 Handling permanent failures: Replica synchronization

> 如果系统成员流失率大，或者故障都是永久性时，Hinted Handoff这一策略就不太行了。特别是hinted data在返回到之前节点时就故障的话。 

Merkle树（其实就是hash树）：每个非叶子节点都存有hash，叶子节点是数据块，其hash值是由子节点的hash值计算出来的，因此比较两个Merkle树根节点的hash值就可以知道这两棵树的不同。

Dynamo中，每个virtual node有一个merkle树，然后使用树遍历的方法就知道两个树是否有不同以及哪里有不同。用来识别每个node中的数据是否有"out of sync"的，如果有就和其它节点采取一定措施进行同步。

### 4.8 Membership and Failure Detection

#### 4.8.1 Ring Membership

管理人员可以通过命令行工具或者浏览器连接到一个Dynamo node（已经在ring中的），来让一个其它的node加入或移出ring。负责处理这个请求的dynamo node会将成员身份变更这一事件以及发生的时间写入永久存储当中。不同dynamo node之间采用一种gossip-based一致性协议来沟通membership change histories，每个node每秒随机选择一个对等node来互相沟通。

在每个node的磁盘中存储有virtual node到物理node的mapping，也是通过gossip-based一致性协议在多个node之间保持一致的。

#### 4.8.2 External Discovery

为了避免多次入环操作形成不同子环的可能性，Dynamo预先配置了一些种子（seed）node。这些种子节点是通过一些外部机制被所有节点所知的节点（知道种子节点是真正环上的节点）。种子可以在从静态配置或配置服务中获得。

#### 4.8.3 Failure Detection

Failure Detection是为了避免对那些已经故障的节点进行无效的操作。Dynamo中，当节点A给节点B发送消息，而节点B很久不响应的话，则节点A就认为B是故障了（即使节点B正在和其它节点通信），则节点A会立马选择一个可以替代节点B的节点进行通信，并且开始周期性地给节点B发探测性消息，来检测节点B是否恢复正常。

### 4.9 Adding/Removing Storage Nodes

> 注意，在本文中token就是指环上的一个id（一个point），key range就是指每个point负责管理的那部分区域。

当节点加入环时，其它节点就不需要存储一部分data，这些data会转移到这个新节点上。（当节点从环上移除时，过程相反）。在转移时会向该新节点进行一次确认，确保新节点不会收到重复传输。



