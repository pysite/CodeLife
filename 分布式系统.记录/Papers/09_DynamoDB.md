## 1 Introduction

亚马逊的网购服务需要高可靠性（reliability，不能停机），以及高拓展性。系统的可靠性和拓展性往往取决于它的应用的状态是如何维持的。可靠性中特别需要强调的点就是可用性（availability），任何时候用户都可以访问亚马逊平台进行购物。很显然，这个系统是由诸多主机组成，因此随时都会有部分主机发生故障，因此必须要处理这些故障使得不影响系统的可靠性、可用性。

亚马逊平台上的许多服务，例如seller lists、shopping carts，都只需要简单的key访问，而不需要复杂的关系型数据访问（性能低，拓展性低），只需要一种primary-key only interface。

Dynamo为了实现高可靠性，拓展性，采用了以下诸多技术：

1. 数据被切分、replicated，其中使用到了**一致性哈希**；
2. 采用**object versioning技术**进一步保障一致性；
3. 采用**a quorum-like技术**和**分布式replica synchronization protol**保证replica的一致性；

4. 采用**基于gossip的分布式故障检测和成员协议**；

Dynamo不需要太多人工操作，存储节点可以随时加入和退出Dynamo without manual partitioning or redistribution.