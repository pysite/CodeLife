## Abstract

Big data application，例如in-situ analytics（现场分析）就主要分为两大部分：模拟和分析。模拟产生数据，**同时**分析模块则分析数据，数据就需要从模拟流向分析。

本文介绍了一种adaptive data compression algorithm。这种算法被实现成了一种中间件（middleware，一种介于系统软件和应用软件之间的软件）。

## 1 Introduction

就是说IO速度和CPU计算速度相比，成为瓶颈。并且在in-situ analytics中数据在模拟和分析两模块之间传输的开销很大。但数据压缩可能会和模拟模块、分析模块争用CPU，因此需要设计专门的机制来保护系统性能。

本文的贡献就是设计了一种决策机制来让系统决定什么时候该采用数据压缩，以及采用什么算法对数据进行压缩。

## 2 Background

*B. In-situ Analytics Placement Strategies*

In-situ Analytics主要分为以下四类：

1. Inline Processing

   simulate一步，再analysis一步，如此循环。

2. Helper-core Processing

   一台主机，部分core执行simulate，部分core执行analysis。

3. Dedicated-nodes Processing

   多台主机，部分主机执行simulate，部分主机执行analysis。

4. Offline Processing：

   一台主机，先执行simulate，把数据写入storage，然后再读取数据，再执行analysis。

## 3 Methodology

在In-situ Analytics中，数据流动分为两类，mem-to-mem（同时需要压缩和解压）和mem-to-storage（不需要解压）。然后**作者设置了一堆量化指标**，并且用这些指标表示出了end-to-end的latency（就是数据传输的总耗时），并且**算法就是如果不压缩直接传输的时间大于进行压缩的总时间，就使用压缩**。

作者表示，如果节点之间的传输带宽太高的话，例如16Gbps，大概需要492个core进行并行压缩才不会拖累带宽...说明在本论文中是允许多核进行压缩工作的。总之就是带宽太高，则压缩/解压所需的CPU核数越多。

## 4 Implementation

没有一种方法可以适用所有场景，因此本文设计了一种adaptive data compression algorithm.

作者开发了两种选择算法，tentative selection和predictive selection.

### 4.1 Tentative Selecting Compression Algorithms

Tentative select就是分为多个round，每个round开始时依次拿几个data block（貌似是各不相同的data block）给各个压缩算法去尝试压缩后并发送，并记录end-to-end的总耗时，尝试完一轮后再选出耗时最短的算法作为本round的胜者，作为本round剩余时间内所采用的算法。当发送的end-to-end的current latency超过一定的threshold（这个阈值可以设定为tentative latency）时，就会停止本round，系统进入下一个round，开始选举新的胜者。

作者说这种方法没有考虑资源状态，就是说可能是因为资源不足导致current latency变慢，而非当前压缩算法不适合。

### 4.2 Predictive Deciding Compression Algorithms

Predictive就是实时检测系统中的各种硬件资源状态，然后根据前面说的那些量化指标和延时公式，计算出每种压缩算法的预测延时，然后选择一个最好的。

作者说这种方法没有考虑数据格式的变化使得一些量化指标不准确了。（就是一些量化指标是检测不到的，是预先测量的，但是在运行时数据格式的变化可能会使这些指标改变）

### 4.3 Performance Comparison of Two Implementations

最后的结论就是，理想情况下，Predictive方法较好，但是在数据量很大，系统状态变化不剧烈的情况下，两种方法差距不大（我甚至认为在实际环境下Tentative方法更好）。

### 4.4 Hybrid T_Selection & P_Selection

就是说在predictive selection的中途时不时地进行一次tentative selection来调整现有的一些量化参数。

### 4.5 Integrating Adaptive Compression Into I/O Middleware

就是说数据占用大约3倍的内存，例如20MB的数据，则其压缩以及发送buffer这些一共需要大约60MB的内存。
