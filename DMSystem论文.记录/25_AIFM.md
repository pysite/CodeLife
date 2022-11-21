## Abstract

AIFM就是去掉page swap机制的，工作在用户态的infiniswap、Fastswap。靠的是程序运行时runtime。

## 1 Introduction

靠OS的swap机制的remote memory都会受到OS kernel机制限制，性能很差。AIFM将swap机制集成到了app-level，当AIFM检测到memory pressure就自然会把一个指针变为remote指针。