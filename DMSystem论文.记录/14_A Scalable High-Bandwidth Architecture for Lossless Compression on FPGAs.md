## Abstract

本文构建的FPGA架构，能够支持最高5.6GB/s的数据压缩速率。本文的架构重点关注scalability（扩展性），并且能够很高效地利用FPGA资源（在相同的吞吐量的条件下，对FPGA占用的面积更小）。（**这里要看下实验结果，看下是怎么对FPGA的利用率高的**）

## 1 Introduction

最近的计算方式从个人计算转向云计算的转变强调了压缩在高效存储系统中的作用。

压缩算法LZ主要是找前后数据的相似性，用指针来替换后面重复出现的数据。压缩算法Huffman则是统计每个symbol出现的频率再用哈夫曼编码（encoding）来替换数据。压缩算法DEFLATE则是LZ和Huffman的结合达成更高的压缩比，GZIP、ZLIB、XPRESS、7ZIP等都是用的DEFLATE算法。

本文的架构可以达到16bytes/cycle的吞吐量，同时使用到的FPGA area是IBM的30%，同时本文的架构最高可达到32bytes/cycle（使用的是Stratix V FPGA）

## 2 Compression Algorithm

> 本章节先介绍DEFLATE，然后介绍我们作出的改动

这篇文章就是在讲作者在FPGA上设计的一个架构，以利用FPGA对DEFLATE算法进行硬件加速，同时进行了一些小的算法上的调整使得压缩算法的压缩比降低了但是速度提升了。

## 6 Related Works

Application-specific integrated circuits（ASIC）对数据压缩进行硬件加速领域，**AHA378可以达到80Gbps**，比Intel QAT 8970的66Gbps还要快一点。