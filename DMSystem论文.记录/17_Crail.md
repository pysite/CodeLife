## Abstract

NodeKernel架构是为temporary数据设计的一种多层次介质存储系统，在DRAM、NVM层次中都可以用RDMA技术进行数据传输（NVM直接用RDMA进行传输的技术是NVMf）。结合了KV存储和文件系统的接口。

## 1 Introduction

现有文件系统和KV存储系统都各有优劣，例如文件系统提供了分层namespace、适合存储并顺序访问大型数据，而分布式KV存储则是针对频繁小数据的访问而设计的。而现有的in-memory KV存储系统，又不适合存储大型数据，因为DRAM很贵。

基于现状的观察：

1. 对于临时数据，durability和fault-tolerance不是特别所需的。
2. 分布式文件系统和分布式KV存储不同点在于，文件系统在获取数据时要通过文件inode和offset然后才可映射到存储设备，然而这样一层额外的映射因为现代网络的高速网络设备以及多核CPU变得十分快速。

基于以上观察设计了NodeKernel架构，这个架构就是指数据是Node，而管理Node的就是Kernel，至于上层接口则可以灵活地选择FS或者KV类型的接口。这个架构就是不论是KV还是FS的数据都可以存，反正对于底层的storage kernel来说都叫node，不同的是这些node的type，例如类型为table或者kv的node，就支持创建同名的数据，反正last-put-wins；而对于类型为file或者directory的node，就支持像文件树一样的遍历。

Kernel分为metadata plane和data plane，元数据平面负责管理各个数据的元数据。数据平台则是负责利用现代高速网络设备对底层数据进行高速传输。

NodeKernel架构目前实际的实现就是Apache Crail，Crail在KV系统和FS系统原本适合的场景下达到了和他们匹配的性能，在这两个系统不适合的场景下更是达到3.4倍的高速性能。

Crail可以利用NVMe来替代一部分DRAM，同时性能损失十分下，但是cost下降了很多。

## 2 Background and Motivation

### 2.1 Requirements and Challenges

- Size, API, and Abstractions Diversity

  就是说不同workload中产生的临时数据大小各不相同，对于小数据适合KV存储，对于大数据（GB）适合文件系统存储。因此设计的NodeKernel需要适合不同大小的数据，同时要结合KV系统和FS系统的上层接口。

- Performance

  一些临时数据是random write，一些是sequence write。因此临时数据存储系统也要针对各种类型的访问模式都能提供高性能。

  同时临时数据存储系统也要利用好现代网络设备和存储设备。

- Beyond In-Memory Storage

  临时数据存储系统应该支持灵活的底层设备配置，让用户自主选择配置多少DRAM和NVMe SSD。

- Non-Requirements

  临时数据的fault-tolerance一般是由上层的应用来保证，例如Spark和Ray系统如果出现故障或丢失数据，就会直接重启任务来重建临时数据。

### 2.2 Limitations of Existing Approaches

略

## 3 The NodeKernel Architecture

NodeKernel就是将不同的语义（KV、FS）与下层的存储引擎分离，同时存储引擎中又分为metadata plane和data plane。

### 3.1 Storage Kernel and Node Types

Kernel以node为单位管理数据，node以path name进行区分（像文件系统中的文件一样）。

Node有以下5种type：

1. File and KeyValue

   两种类型都有append和read操作，唯一不同的是addChild在已经有文件的path name时，File类型的会失败，而KeyValue类型的会直接替换掉现有的node。

2. Directory and Table：

   Directory和Table分别是File和KeyValue类型的container，其中的数据就是里面node的path name，并且提供遍历（enumeration）子node name的功能。不同点就是Table可以设置为"non-enumerable"，这样在插入新子节点时会快一点，因为不需要更新table文件中的数据。

3. Bag

   Bag类型和Directory类型相似，是File类型节点的container，不同点在于Bag类型可以直接调用read函数就可以顺序依次读出其中包含的文件的数据，这比分别read每个File要快一些。Bag类型一般用Shuffle任务，使得应用只需要读取一次Bag就可以获取到散布到多个data node中的数据。

Directory支持多层次，而Bag和Table都只支持flat的namespace。

**Why provide a single unified storage namespace？**这样做有两点好处，一是提供了一个统一的storage。二是使得应用可以根据自己的需求灵活地使用不同的type管理数据而非死板地根据数据size来决定类型，且NodeKernel中没有规定每种type地数据大小为多少。

### 3.2 System Architecture

就是说NodeKernel架构里分为metadata server和storage server，其中的node的数据是以Block为单位存储的。

Metadata server存储有node层次关系以及各个node的data block以及其所在Storage server的映射关系（这个block在哪个server上）。

Storage server在启动时会将自己拥有的data block在one of metadata server上进行注册。

Metadata server维持有一个free block list，知道自己所管理的Storage servers中free block的情况。

Node所涉及的相关函数addChild、removeChild、read、append、update等被实现成了一个client library，client程序调用这些函数就会根据需要去联系metadata server或storage server。

**Target deployment**：就是说优先关注性能而非durability或fault-tolerance。

**Performance challenges**：KV就是low latency，FS就是high band-width，NodeKernel就是要同时争取实现这两种。

#### 3.2.1 Low-latency metadata operations

为了保证metadata的获取足够快，NodeKernel将metadata plane中的操作只保留几种必要的：create（创建节点），lookup（获取节点metadata），remove（删除节点），move（移动节点在hierarchy中的位置），map（查找file节点中某个offset的位置是在哪个storage server），register（storage server刚启动时会向metadata server注册）。

操作enumeration由于要获取的data较多，因此由data plane负责。

#### 3.2.2 Metadata partitioning

NodeKernel中一台metadata server可支持metadata RPC操作的吞吐量为每秒10M次。（using RDMA）

NodeKernel支持多台metadata server，用hash进行分区即可，例如hash值为1的找metadata server1，hash值为2的找metadata server2。这种hash有好有坏，好处就是速度快，坏处就是可能造成负载不均衡的问题。但是动态partitioning的坏处就是一次查询可能要到多个metadata server，因此速度会较慢。

#### 3.2.3 Hardware-accelerated storage

Data plane要尽量接口简单，尽量implemented in storage。（offload to hardware）

#### 3.2.4 Tiered storage

根据介质给storage server进行分类，例如所有的DRAM storage server是class1，所有的NVMe SSD storage server就是class2.

并且一台物理机可能对应多个storage server，例如它的DRAM是一台，它的SSD又是另一台。

NodeKernel利用多层次storage server的方法就是"**上层不够了就去用下层的**"，而非根据什么LRU驱逐数据到下层。

## 4 Crail

``CrailStore``就是整个Crail服务的抽象。

Client在创建node时可以指出参数sc来表示希望这个数据存放的storage server的class。

Crail所有操作都是异步的，都是返回一个future object。（JAVA中的一个类，非阻塞函数都会返回这种）Crail API调用失败会返回invalid   future或者exception。

当前版本Crail不会隔离（isolate）多个用户的data。

### 4.1 Metadata plane

Crail metadata存储的三种data（前面说过）：1.the storage hierachy，2.the set of free-blocks，3.the assignments of blocks to "nodes"。

Crail Write的流程：

> 如果write的位置没有allocated block，则会client调用map RPC函数申请一个free block。
>
> metadata server收到这个请求后，首先根据sc参数到指定class中的storage server获取free block；如果这个class中的storage server都没有free block了则会尝试去下一class中获取。在一个class的所有storage servers中采用round-robin算法选则server分配新block。如果下级class的storage servers中都没有free block则函数会直接fail。

每一个metadata server对于用户来说就是个RPC service（使用RDMA实现的DaRPC）；在每个metadata server中每个CPU core负责一部分client connection，core用一个queue进行排队自己负责的那部分的消息。RPC buffer使用的内存就是自己core所在的NUMA节点的内存。

Crail的container类型的node的data就是装有多条fixed-size records的数组，每条记录带有一个valid flag。

就是说有时候可能container node的数据中某个record仍然是valid的，但是在metadata server中这个record对应的node可能已经被删除了，这个时候就以metadata server为准。

### 4.2 Data plane

Crail中只有2个Class的storage server，DRAM和NVMe-based SSD；

**RDMA storage class**：这种server就是用RDMA管理内存。

**NVMe-over-Fabrics storage class**：就是用RDMA访问远程NVMe设备。Crail的NVMf storage server只负责连接一下NVMf controller然后把相关信息汇报给metadata server，这样client跟metadata server拿到相关信息后就直接跟远端NVMf controller连接并进行读写block。

### 4.3 Failure semantics and persistence

Crail当前不提供fault-tolerance，但是也可以启动一种log机制来进行fault-tolerance，就是在metadata中对operation历史记录进行log，在DRAM storage server中将内存区域map到SSD中进行持久化存储。

### 4.4 Anatomy of data access

就是说，读取一个block分为两个阶段，一是获取metadata，二是读取数据，当一个request涉及到多个block时这两个阶段会流水线式地进行。



## 5 Evaluation

### 5.1 Microbenchmarks

#### Small and medium-size values

将Crail和另外两个SOTA的KV系统（DRAM存储系统RAMCloud，以及NVMe存储系统Aerospike）进行不同大小的读写latency测试。但这两个SOTA系统都不支持大数据的KV（16MB和128MB）。

注意作者在这部分画的图是指数级增长的。

#### IOPS scaling

这一部分实验就是测试Crail在固定大小的消息（256B）下的IOPS数据（16个client机器，每个机器4个进程，因此共64个client进程）（只有1个storage server，只有1个metadata server）

结果就是随着IOPS的增加，其latency会缓慢地增加（可以接受），但当IOPS超过设备极限时latency就会指数级暴增。

#### Metadata performance

这一部分实验只测试了metadata server的性能，具体是分别用1、2、4个metadata server，测试整个系统的IOPS的上限，与另外两个分布式文件系统Octopus和Alluxio进行比较。

#### Accessing large datasets

这一部分实验就是测试在不同大小的client buffer情况下Crail的reading带宽。

#### Summary

结论就是说Crail适合各种大小的数据，且在旧系统擅长的环境下能和他们55开，在不适合旧系统的环境能远胜他们，



### 5.2 Systems-level Benchmarks

#### 5.2.1 NoSQL（非关系型数据库） workloads

YCSB是专门测试NoSQL数据库的工作负载。这部分实验是在一个metadata server，1个storage server，1个client的条件下跑YCSB的测试，来看在真实工作负载下各种操作的latency（及其所占百分比）。

这一部分实验使用了YCSB中的workload B，包含95%的read操作和5%的update操作。

#### Summary

1. Crail能将底层存储设备的性能优势带给上层的workload level。
2. Crail适合各种大小的dataset。

#### 5.2.2 Spark Implementation

这一部分是将Spark跑在Crail上，用大数据应用来实际测试Crail的性能。（让Spark应用中的Broadcast和Shuffle操作使用Crail）

- Broadcast

  Broadcast就是一次write，然后被多个reader读取，依然是测试latency的分布情况。（没说几台metadata 或storage server）

- Shuffle

  Crail中的Shuffle，分为writer和reducer（多对多的关系），每个reducer对应一个Bag node，每个Bag中的每个File对应一个writer，因此等writer将数据分发到不同Bag中的某个File后，各个reducer只需读取自己的那个Bag node即可。在这里测试的是处理完512GB数据所需的runtime，以及在这个过程中系统的吞吐量变化。



### 5.3 Efficiency of hybrid DRAM/NVM setup

这一部分使用Spark中著名的workload Terasort对Crail 进行测试，Terasort就是一种外排序算法，就是用200GB的workload进行测试，比较任务完成的整个耗时，变量是Crail中DRAM和SSD的占比。