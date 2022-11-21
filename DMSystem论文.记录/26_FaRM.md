## Abstract

> FaRM exposes the memory of machines in the cluster as a shared address space. Applications can use transactions to allocate, read, write, and free objects in the address space with location transparency.

## 1 Introduction

FaRM的两大提升性能的设计：

1. 无锁READ（被严格的序列事务化）
2. 支持collocating objects和function shipping

## 2 Background on RDMA

[未完待续]
